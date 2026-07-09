# Bug Report — CoWork Multi-Tenant Coworking Space Booking API

All line numbers refer to the **original (buggy) code**. Each entry lists the file/lines,
what the bug was and why it caused incorrect behavior, and how it was fixed.

## Summary

| # | Bug | File | Rule violated |
|---|-----|------|---------------|
| 1 | Access tokens live 900 minutes instead of 900 seconds | `app/auth.py` | 8 (Auth) |
| 2 | Logout never actually invalidates the token (`sub` vs `jti`) | `app/auth.py` | 8 (Auth) |
| 3 | Refresh tokens are reusable, not single-use | `app/routers/auth.py` | 8 (Auth) |
| 4 | Duplicate registration returns the existing account instead of 409 | `app/routers/auth.py` | 15 (Registration) |
| 5 | UTC offset dropped instead of converted to UTC | `app/timeutils.py` | 1 (Datetimes) |
| 6 | 5-minute grace window allows bookings starting in the past | `app/routers/bookings.py` | 2 (Booking window) |
| 7 | Missing minimum-duration check (0h / negative durations accepted) | `app/routers/bookings.py` | 2 (Booking window) |
| 8 | Back-to-back bookings rejected as conflicts (`<=` vs `<`) | `app/routers/bookings.py` | 3 (No double-booking) |
| 9 | Pagination: wrong sort order, off-by-one offset, hardcoded page size | `app/routers/bookings.py` | 11 (Pagination) |
| 10 | `GET /bookings/{id}` returns `created_at` as `start_time` | `app/routers/bookings.py` | API contract |
| 11 | Members can read other members' bookings | `app/routers/bookings.py` | 10 (Visibility) |
| 12 | Refund tiers: 48h boundary excluded and <24h refunds 50% instead of 0% | `app/routers/bookings.py` | 6 (Refunds) |
| 13 | Refund rounding: banker's rounding in response, truncation in ledger | `app/routers/bookings.py`, `app/services/refunds.py` | 6 (Refunds) |
| 14 | Stale caches: report not invalidated on create, availability not on cancel | `app/routers/bookings.py` | 12, 13 (Freshness) |
| 15 | CSV export leaks other organizations' bookings | `app/services/export.py` | 9 (Multi-tenancy) |
| 16 | Race: duplicate reference codes under concurrent creation | `app/services/reference.py` | 7 (Reference codes) |
| 17 | Race: rate limiter loses requests under concurrency | `app/services/ratelimit.py` | 5 (Rate limit) |
| 18 | Race: room stats lose updates under concurrency | `app/services/stats.py` | 14 (Room stats) |
| 19 | Race: double-booking / quota bypass under concurrent creates | `app/routers/bookings.py` | 3, 4 |
| 20 | Race: concurrent cancels produce duplicate refunds | `app/routers/bookings.py` | 6 |
| 21 | Deadlock: opposite lock order in notifications hangs the service | `app/services/notifications.py` | 16 (Liveness) |
| 22 | Malformed booking datetimes crash with HTTP 500 | `app/routers/bookings.py` | Error contract / 16 |

---

## Bug 1 — Access Tokens Live 900 Minutes Instead of 900 Seconds

**File:** `app/auth.py`, line 50

### What / Why
Rule 8 requires `exp − iat` = exactly 900 seconds. The lifetime was computed as
`timedelta(minutes=ACCESS_TOKEN_EXPIRE_MINUTES * 60)` — with `ACCESS_TOKEN_EXPIRE_MINUTES = 15`
that is **900 minutes (15 hours)**, 60× too long. The `* 60` treats minutes as if they were
seconds while still passing them to the `minutes=` parameter.

### Fix
```python
# Before
lifetime = timedelta(minutes=ACCESS_TOKEN_EXPIRE_MINUTES * 60)
# After
lifetime = timedelta(minutes=ACCESS_TOKEN_EXPIRE_MINUTES)
```

---

## Bug 2 — Logout Never Invalidates the Access Token

**File:** `app/auth.py`, line 97

### What / Why
`revoke_access_token()` stores the token's **`jti`** in the `_revoked_tokens` set, but
`get_token_payload()` checked whether the token's **`sub`** (the user id) is in that set.
A user id never equals a random `jti` hex string, so the check never matched: any access
token continued to work after `POST /auth/logout`, violating rule 8 ("logout immediately
invalidates the presented access token").

### Fix
```python
# Before
if payload.get("sub") in _revoked_tokens:
# After
if payload.get("jti") in _revoked_tokens:
```

---

## Bug 3 — Refresh Tokens Are Not Single-Use

**File:** `app/routers/auth.py`, lines 81–93 (`refresh` endpoint)

### What / Why
Rule 8: refresh tokens are single-use — refreshing must invalidate the presented refresh
token, and reuse must yield 401. The endpoint decoded the token and issued new tokens but
never recorded the presented token as used, so the same refresh token could be replayed
indefinitely (a stolen refresh token would remain valid for 7 days).

### Fix
Check the presented token's `jti` against the revocation set and revoke it once used
(helper `is_token_revoked` added in `app/auth.py`):
```python
if is_token_revoked(data):
    raise AppError(401, "UNAUTHORIZED", "Token has been revoked")
user = db.query(User).filter(User.id == int(data["sub"])).first()
if user is None:
    raise AppError(401, "UNAUTHORIZED", "Unknown user")
revoke_access_token(data)  # single-use: invalidate the presented refresh token
```

---

## Bug 4 — Duplicate Registration Silently Returns the Existing Account

**File:** `app/routers/auth.py`, lines 37–43 (`register` endpoint)

### What / Why
Rule 15: a duplicate username within the org must yield `409 USERNAME_TAKEN`. Instead, the
endpoint returned HTTP 201 with the **existing** user's `user_id`, `username` and `role` —
without checking any password. Besides violating the contract, this leaked account
information and let anyone "successfully register" as an existing username.

### Fix
```python
# Before
if existing is not None:
    return {"user_id": existing.id, "org_id": org.id, "username": existing.username, "role": existing.role}
# After
if existing is not None:
    raise AppError(409, "USERNAME_TAKEN", "Username already taken in this organization")
```

---

## Bug 5 — Input Datetime Offset Dropped Instead of Converted

**File:** `app/timeutils.py`, line 13

### What / Why
Rule 1: input datetimes carrying a UTC offset must be **converted** to UTC. The parser did
`dt.replace(tzinfo=None)`, which throws the offset away and keeps the local wall-clock time.
`"2026-07-10T18:00:00+06:00"` (= 12:00 UTC) was stored as 18:00 UTC — six hours wrong. That
corrupted stored times, conflict checks, quota windows, refund-notice calculations and all
responses for any client that sent zoned datetimes.

### Fix
```python
# Before
dt = dt.replace(tzinfo=None)
# After
dt = dt.astimezone(timezone.utc).replace(tzinfo=None)
```

---

## Bug 6 — Grace Window Allows Bookings Starting in the Past

**File:** `app/routers/bookings.py`, line 86

### What / Why
Rule 2: `start_time` must be **strictly in the future at request time — no grace window**.
The check `start <= now - timedelta(seconds=300)` accepted any start up to 5 minutes in the
past (and a start exactly equal to `now`).

### Fix
```python
# Before
if start <= now - timedelta(seconds=300):
# After
if start <= now:
```

---

## Bug 7 — Minimum Duration Never Enforced

**File:** `app/routers/bookings.py`, lines 90–94

### What / Why
Rule 2: duration must be a whole number of hours, **minimum 1**, maximum 8, and `end_time`
strictly after `start_time`. Only the maximum was checked. A 0-hour booking
(`end == start`) or even a negative whole-hour duration (`end` before `start`) passed
validation, producing free/negative-price bookings. The `MIN_DURATION_HOURS` constant
existed but was never used.

### Fix
```python
# Before
if duration_hours > MAX_DURATION_HOURS:
# After
if duration_hours < MIN_DURATION_HOURS or duration_hours > MAX_DURATION_HOURS:
```
(≤ 0 durations now fall below the minimum and are rejected with `400 INVALID_BOOKING_WINDOW`.)

---

## Bug 8 — Back-to-Back Bookings Rejected as Conflicts

**File:** `app/routers/bookings.py`, line 50 (`_has_conflict`)

### What / Why
Rule 3 defines overlap as `existing.start < new.end AND new.start < existing.end`
(strict inequalities), explicitly allowing back-to-back bookings. The code used `<=` on
both sides, so a booking ending exactly when another starts (10:00–11:00 then 11:00–12:00)
was wrongly rejected with `409 ROOM_CONFLICT`.

### Fix
```python
# Before
if b.start_time <= end and start <= b.end_time:
# After
if b.start_time < end and start < b.end_time:
```

---

## Bug 9 — Pagination Broken Three Ways

**File:** `app/routers/bookings.py`, lines 136–140 (`list_bookings`)

### What / Why
Rule 11: items sorted **ascending** by `start_time` (ties by ascending id); page N with
limit L returns items `[(N−1)·L, N·L)`. The code had three defects:
1. `order_by(Booking.start_time.desc(), ...)` — descending instead of ascending;
2. `.offset(page * limit)` — off by one page: page 1 skipped the first `limit` items
   entirely (items 0–9 were unreachable);
3. `.limit(10)` — hardcoded, ignoring the `limit` query parameter.

### Fix
```python
# Before
base.order_by(Booking.start_time.desc(), Booking.id.asc()).offset(page * limit).limit(10)
# After
base.order_by(Booking.start_time.asc(), Booking.id.asc()).offset((page - 1) * limit).limit(limit)
```

---

## Bug 10 — Booking Detail Returns `created_at` as `start_time`

**File:** `app/routers/bookings.py`, line 166 (`get_booking`)

### What / Why
After serializing the booking correctly, the endpoint overwrote the response's
`start_time` with the booking's **`created_at`** timestamp:
`response["start_time"] = iso_utc(booking.created_at)`. Every `GET /bookings/{id}`
reported a wrong start time (the moment the booking was created, not when it starts).

### Fix
Removed the overwrite line; `serialize_booking()` already sets the correct `start_time`.

---

## Bug 11 — Members Can Read Other Members' Bookings

**File:** `app/routers/bookings.py`, lines 156–163 (`get_booking`)

### What / Why
Rule 10: members may read **only their own** bookings; another member's booking id must
return `404 BOOKING_NOT_FOUND`. The query filtered only by org (via the Room join), so any
member could read every booking in their org. The cancel endpoint had the correct
owner-or-admin check; the read endpoint was missing it.

### Fix
Added the same check used by cancel:
```python
if user.role != "admin" and booking.user_id != user.id:
    raise AppError(404, "BOOKING_NOT_FOUND", "Booking not found")
```

---

## Bug 12 — Refund Tiers Wrong at Both Boundaries

**File:** `app/routers/bookings.py`, lines 200–206 (`cancel_booking`)

### What / Why
Rule 6: notice ≥ 48h → 100%; 24h ≤ notice < 48h → 50%; notice < 24h → **0%**. Two defects:
1. `if notice_hours > 48:` — strict `>` on floored hours meant a notice of exactly 48h
   (and anything up to 48h59m59s… floored to 48) fell into the 50% tier instead of 100%.
2. The `else` branch returned `refund_percent = 50` — cancellations with less than 24 hours'
   notice were refunded 50% instead of 0%.

### Fix
```python
# Before
if notice_hours > 48:   refund_percent = 100
elif notice >= timedelta(hours=24): refund_percent = 50
else:                   refund_percent = 50
# After
if notice_hours >= 48:  refund_percent = 100
elif notice >= timedelta(hours=24): refund_percent = 50
else:                   refund_percent = 0
```

---

## Bug 13 — Refund Amount: Wrong Rounding and Response/Ledger Mismatch

**Files:** `app/routers/bookings.py` line 208, `app/services/refunds.py` lines 15–17

### What / Why
Rule 6: the refund rounds to the nearest cent with **half-cents rounding up** (README
example: 50% of 1001 = 501), and the amount in the cancel response must **equal** the
amount stored in the RefundLog. Two different wrong computations were used:
- Endpoint: `round(price * percent/100)` — Python's `round()` uses banker's rounding
  (half-to-even), so 50% of 1001 → `round(500.5)` → **500**, not 501.
- Ledger: `int(refund_dollars * 100)` — float truncation, e.g. 50% of 999 →
  `int(499.49999…)` → **499** (and truncation is not "nearest cent" at all).

So the response and the stored log could disagree with the spec *and* with each other.

### Fix
Both places now use the same exact integer half-up formula:
```python
amount = (price_cents * percent + 50) // 100
```
(e.g. 1001 × 50 = 50050, +50 = 50100, //100 = **501** ✓)

---

## Bug 14 — Stale Caches: Missing Invalidation on Create/Cancel

**File:** `app/routers/bookings.py` (end of `create_booking` and `cancel_booking`)

### What / Why
Rules 12/13: the usage report and availability must reflect the current state
**immediately**. Both are cached, but invalidation was asymmetric:
- `create_booking` invalidated availability but **not** the usage report → a cached report
  kept showing pre-booking counts/revenue.
- `cancel_booking` invalidated the report but **not** availability → a cancelled booking
  kept showing as a busy interval.

### Fix
```python
# create_booking — added:
cache.invalidate_report(user.org_id)
# cancel_booking — added:
cache.invalidate_availability(booking.room_id, booking.start_time.date().isoformat())
```

---

## Bug 15 — CSV Export Leaks Other Organizations' Bookings

**File:** `app/services/export.py`, lines 48–52 (`generate_export`)

### What / Why
Rule 9: every code path must be scoped to the caller's org; cross-org ids behave as
non-existent. With `include_all=true&room_id=<id>`, the export called
`fetch_bookings_raw(db, room_id)`, which queries by room id **with no org filter**. An
admin of org A could pass a room id belonging to org B and download all of org B's
bookings (ids, user ids, times, prices) — a multi-tenancy data breach.

### Fix
```python
# Before
if include_all:
    if room_id is not None:
        rows = fetch_bookings_raw(db, room_id)
    else:
        rows = _fetch_scoped(db, org_id, None, None)
# After
if include_all:
    rows = _fetch_scoped(db, org_id, None, room_id)
```
A cross-org `room_id` now simply matches nothing (header-only CSV).

---

## Bug 16 — Race: Duplicate Reference Codes (Hard)

**File:** `app/services/reference.py`, lines 17–21

### What / Why
Rule 7: reference codes must be unique **including under concurrent creation**. The
counter did a non-atomic read → (0.12s `_format_pause()`) → write. Two concurrent
requests both read the same `current`, slept, and both wrote `current + 1` — issuing the
**same code twice** and losing a counter increment. The sleep sits exactly inside the race
window, making collisions near-certain under concurrent load.

### Fix
Guarded the read-increment with a `threading.Lock`:
```python
_lock = threading.Lock()

def next_reference_code() -> str:
    with _lock:
        current = _counter["value"]
        _format_pause()
        _counter["value"] = current + 1
    return f"CW-{current:06d}"
```

---

## Bug 17 — Race: Rate Limiter Undercounts Concurrent Requests (Hard)

**File:** `app/services/ratelimit.py`, lines 18–26

### What / Why
Rule 5: `POST /bookings` limited to 20 requests / rolling 60s per user, holding under
concurrent requests. `record_and_check` read the user's bucket, slept 0.1s
(`_settle_pause()`), then wrote it back. N concurrent requests all read the same bucket
and each wrote back a bucket containing only its own timestamp — dropping the others.
The recorded count stayed far below reality, so a burst well beyond 20 requests never
produced `429 RATE_LIMITED`.

### Fix
Wrapped the trim-record sequence in a `threading.Lock` so each request atomically reads,
records, and writes its bucket.

---

## Bug 18 — Race: Room Stats Lose Updates (Hard)

**File:** `app/services/stats.py`, lines 15–26

### What / Why
Rule 14: stats must always equal the values derivable from the bookings, including after
bursts of concurrent activity. `record_create`/`record_cancel` did read → 0.1s
`_aggregate_pause()` → write on the shared `_stats` dict. Concurrent bookings for the same
room overwrote each other's updates (classic lost update), leaving
`total_confirmed_bookings`/`total_revenue_cents` permanently short.

### Fix
Guarded both record functions with a `threading.Lock`, making each increment/decrement
atomic.

---

## Bug 19 — Race: Double-Booking and Quota Bypass on Concurrent Creates (Hard)

**File:** `app/routers/bookings.py` (`create_booking`, conflict/quota check → insert)

### What / Why
Rules 3 and 4 must hold **under concurrent requests**. The conflict check and quota check
were classic time-of-check/time-of-use races: two concurrent requests for the same slot
both ran `_has_conflict` (which sleeps 0.12s in `_pricing_warmup()`) before either had
committed, both saw "no conflict", and **both inserted** — a double-booking. Identically,
`_check_quota` (0.1s `_quota_audit()` sleep) let a member blow past the 3-booking quota by
firing requests in parallel.

### Fix
Added a module-level `threading.Lock` (`_write_lock`) and wrapped the critical section —
conflict check, quota check, insert, commit — so validation and insert are atomic with
respect to other booking writes. (Single-process deployment per the Dockerfile, so a
process-level lock is sufficient; SQLite serializes at file level anyway.)

```python
with _write_lock:
    if _has_conflict(db, room.id, start, end):
        raise AppError(409, "ROOM_CONFLICT", ...)
    _check_quota(db, user.id, now, start)
    ...
    db.add(booking); db.commit(); db.refresh(booking)
```

---

## Bug 20 — Race: Concurrent Cancels Produce Duplicate Refunds (Hard)

**File:** `app/routers/bookings.py` (`cancel_booking`)

### What / Why
Rule 6: a cancelled booking has **exactly one** RefundLog entry; must hold under
concurrent cancel requests. The status check (`if booking.status == "cancelled"`) and the
status write were separated by `log_refund()` and a 0.12s `_settlement_pause()`. Two
concurrent cancels both read `status == "confirmed"`, both logged a refund, and both
returned 200 — double-refunding the booking and never returning the required
`409 ALREADY_CANCELLED` to the loser.

### Fix
Moved the fetch + status check + refund log + status commit inside the same `_write_lock`
critical section. The second request now re-reads the booking after the winner commits,
sees `cancelled`, and gets `409 ALREADY_CANCELLED`; exactly one RefundLog row is written,
and the response amount always equals the logged amount (see Bug 13).

---

## Bug 21 — Deadlock: Opposite Lock Order in Notifications Hangs the Service (Hard)

**File:** `app/services/notifications.py`, lines 24–35

### What / Why
Rule 16: no combination of concurrent valid requests may hang the service. The two
notification paths acquired the same two locks in **opposite order**:
- `notify_created`: `_email_lock` → `_audit_lock`
- `notify_cancelled`: `_audit_lock` → `_email_lock`

With the 0.1–0.12s sleeps held inside the outer lock, a concurrent create + cancel
reliably interleaved into an ABBA deadlock: each thread held one lock and waited forever
for the other. Both requests hung, and every later create/cancel queued up behind the dead
locks — the service stopped responding to booking traffic entirely.

### Fix
Made `notify_cancelled` acquire the locks in the same global order as `notify_created`
(`_email_lock` → `_audit_lock`), which makes circular wait impossible.

---

## Bug 22 — Malformed Booking Datetimes Crash With HTTP 500

**File:** `app/routers/bookings.py`, lines 87–88 (`create_booking`)

### What / Why
`BookingCreateRequest` declares `start_time`/`end_time` as plain `str`, so Pydantic does
not validate the format. `parse_input_datetime()` then calls `datetime.fromisoformat()`,
which raises an unhandled `ValueError` for any non-ISO value (e.g.
`{"start_time": "tomorrow"}`), producing an **HTTP 500 Internal Server Error** instead of
a well-formed application error. The error contract defines `INVALID_BOOKING_WINDOW`
(400) for bad booking windows, and the sibling endpoints that parse date strings
(`GET /rooms/{id}/availability`, `GET /admin/usage-report`) both catch `ValueError` and
return `400 INVALID_BOOKING_WINDOW` — this endpoint was missing the same guard.

### Fix
```python
try:
    start = parse_input_datetime(payload.start_time)
    end = parse_input_datetime(payload.end_time)
except ValueError:
    raise AppError(400, "INVALID_BOOKING_WINDOW", "Invalid datetime")
```

---

## Verification

All fixes were verified end-to-end against a running server — **99 checks, all passing**:
- **58 sequential checks** covering every business rule and the exact API contract
  (status codes, error codes, JSON field names, JWT claims, CSV header).
- **15 concurrency checks**: concurrent same-slot creates (exactly one 201), concurrent
  quota (exactly 3 succeed), concurrent cancels (one 200 / one RefundLog), 25-request
  rate-limit burst (exactly 5 × 429), reference-code uniqueness under bursts,
  stats consistency after a create+cancel storm, and no deadlock/hang.
- **26 edge-case checks**: malformed datetimes (400, not 500), 1h/8h boundary durations,
  `Z`-suffix and `+06:00` offset inputs, non-hour-aligned whole-hour bookings, cross-org
  room ids (404), token-type confusion (401), pagination extremes (`limit=100`, far
  pages, 422 on out-of-range params), cross-midnight availability, inclusive
  usage-report boundaries, export scoping with and without `include_all`, and cancelled
  bookings remaining visible with exactly one refund entry.
- The repository's included smoke test (`pytest`) passes.
