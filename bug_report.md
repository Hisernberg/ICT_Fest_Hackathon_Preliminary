# Bug Report — CoWork API

Each entry lists where the bug was, what it broke, and how it was fixed.
Line numbers refer to the **original** (broken) code.

---

## Auth

### 1. Access tokens lived 900 minutes instead of 900 seconds
- **File:** `app/auth.py`, line 50 (`create_access_token`)
- **Bug:** `timedelta(minutes=ACCESS_TOKEN_EXPIRE_MINUTES * 60)` = `timedelta(minutes=900)`,
  so `exp − iat` was 54 000 seconds. The contract requires exactly 900 seconds.
- **Fix:** `timedelta(minutes=ACCESS_TOKEN_EXPIRE_MINUTES)`.

### 2. Logout never actually revoked the token
- **File:** `app/auth.py`, line 97 (`get_token_payload`)
- **Bug:** `revoke_access_token` stores the token's `jti` in `_revoked_tokens`, but the
  request-time check tested `payload.get("sub")` (the user id) against that set.
  A `jti` (uuid hex) never equals a user id, so a logged-out access token kept
  working forever.
- **Fix:** check `payload.get("jti") in _revoked_tokens`.

### 3. Refresh tokens were not single-use
- **File:** `app/routers/auth.py` (`refresh`), `app/auth.py`
- **Bug:** `/auth/refresh` issued new tokens but never invalidated the presented
  refresh token, so the same refresh token could be replayed indefinitely.
  The spec requires reuse → 401.
- **Fix:** added `consume_refresh_token()` in `app/auth.py` that records the presented
  refresh token's `jti` in a used set (under a lock, so concurrent replays can't both
  pass) and raises 401 on reuse. Called from the `/auth/refresh` handler.

### 4. Registering an existing username returned 201 with the existing user
- **File:** `app/routers/auth.py`, lines 32–43 (`register`)
- **Bug:** a duplicate username inside the org silently returned the *existing*
  user's data with a 201 instead of failing. Contract: 409 `USERNAME_TAKEN`.
- **Fix:** raise `AppError(409, "USERNAME_TAKEN", …)`. Also wrapped the user/org
  commits in `IntegrityError` handling so concurrent duplicate registrations map to
  409 (username) or fall back to joining the org (org-name race) instead of a 500.

## Datetime handling

### 5. UTC offsets were dropped instead of converted
- **File:** `app/timeutils.py`, line 13 (`parse_input_datetime`)
- **Bug:** `dt.replace(tzinfo=None)` threw away the offset without converting, so
  `2026-07-10T15:00:00+06:00` was stored as 15:00 UTC instead of 09:00 UTC. That
  corrupted prices/overlaps/refund tiers for any offset-carrying input.
- **Fix:** `dt.astimezone(timezone.utc).replace(tzinfo=None)`.

## Booking creation

### 6. 300-second grace window for past start times
- **File:** `app/routers/bookings.py`, line 86
- **Bug:** `if start <= now - timedelta(seconds=300)` accepted bookings starting up to
  5 minutes in the past. Spec: `start_time` strictly in the future, no grace window.
- **Fix:** `if start <= now`.

### 7. No minimum-duration / `end_time > start_time` validation
- **File:** `app/routers/bookings.py`, lines 89–94
- **Bug:** only the *maximum* duration (8 h) was checked. Durations of 0 (end == start)
  and negative whole hours (end before start) were accepted, creating zero/negative
  price bookings. Spec: minimum 1 hour, `end_time` strictly after `start_time`,
  all violations → 400 `INVALID_BOOKING_WINDOW`.
- **Fix:** `if duration_hours < MIN_DURATION_HOURS or duration_hours > MAX_DURATION_HOURS`.
- Also wrapped `parse_input_datetime` in try/except so an unparseable datetime string
  returns 400 `INVALID_BOOKING_WINDOW` instead of an unhandled 500.

### 8. Back-to-back bookings rejected as conflicts
- **File:** `app/routers/bookings.py`, line 50 (`_has_conflict`)
- **Bug:** the overlap test used inclusive comparisons
  (`b.start_time <= end and start <= b.end_time`), so a booking ending exactly when
  another starts was treated as a conflict. Spec: overlap iff
  `existing.start < new.end AND new.start < existing.end` (back-to-back allowed).
- **Fix:** strict `<` on both sides.

### 9. Double-booking / quota race under concurrent requests
- **File:** `app/routers/bookings.py` (`create_booking`)
- **Bug:** the conflict check, quota check, and insert were a non-atomic
  check-then-act sequence (with deliberate `time.sleep` "warmup/audit" pauses widening
  the window). Two concurrent requests for the same slot both passed `_has_conflict`
  and both got 201; the same race let a member exceed the 3-booking quota.
- **Fix:** a module-level `threading.Lock` (`_booking_lock`) now serializes the
  conflict check + quota check + insert + commit, so exactly one of two conflicting
  concurrent requests succeeds and the quota holds.

## Booking listing / detail

### 10. Pagination was broken three ways
- **File:** `app/routers/bookings.py`, lines 136–141 (`list_bookings`)
- **Bug:**
  1. ordered by `start_time.desc()` — spec says ascending;
  2. `offset(page * limit)` — page 1 skipped the first `limit` items (should be
     `(page − 1) × limit`);
  3. `.limit(10)` hardcoded — the `limit` query parameter was ignored.
- **Fix:** `order_by(start_time.asc(), id.asc()).offset((page - 1) * limit).limit(limit)`.

### 11. Members could read other members' bookings
- **File:** `app/routers/bookings.py` (`get_booking`)
- **Bug:** `GET /bookings/{id}` only scoped by org, so any member could read any
  booking in their org. Spec: another member's booking id → 404 `BOOKING_NOT_FOUND`
  (only admins may read any org booking).
- **Fix:** added the same ownership check as cancel:
  `if user.role != "admin" and booking.user_id != user.id → 404`.

### 12. `start_time` in the detail response was overwritten with `created_at`
- **File:** `app/routers/bookings.py`, line 166 (`get_booking`)
- **Bug:** `response["start_time"] = iso_utc(booking.created_at)` replaced the real
  start time with the creation timestamp.
- **Fix:** removed the line.

## Cancellation & refunds

### 13. Refund tiers were wrong at both boundaries
- **File:** `app/routers/bookings.py`, lines 200–206 (`cancel_booking`)
- **Bug:**
  1. the 100% tier used `int(notice_hours) > 48` on a floor-divided hour count, so
     exactly 48 h notice (and anything under 49 h) got 50% instead of 100%;
  2. the `else` branch returned **50%** for notice < 24 h — spec says **0%**.
- **Fix:** compare the raw `timedelta`: `notice >= 48h → 100`, `notice >= 24h → 50`,
  else `0`.

### 14. Refund rounding used banker's rounding instead of half-up
- **File:** `app/routers/bookings.py`, line 208
- **Bug:** `round(booking.price_cents * (refund_percent / 100.0))` rounds half to even:
  50% of 1001 = `round(500.5)` = **500**. Spec: half-cents round up → **501**.
- **Fix:** integer half-up arithmetic: `(price_cents * refund_percent + 50) // 100`.

### 15. RefundLog stored a different amount than the cancel response returned
- **File:** `app/services/refunds.py`, lines 14–17 (`log_refund`)
- **Bug:** the log recomputed the amount independently via float dollars and
  truncation (`int(refund_dollars * 100)`), so 50% of 1001¢ was stored as 500 while
  the endpoint returned a differently-rounded value. Spec: the cancel response amount
  must equal the RefundLog amount.
- **Fix:** `log_refund` now takes the already-computed `amount_cents` from the cancel
  handler, so a single value is both stored and returned.

### 16. Concurrent cancels produced two refunds
- **File:** `app/routers/bookings.py` (`cancel_booking`)
- **Bug:** the status check, refund insert, and status update were non-atomic (with a
  planted `_settlement_pause()` between refund logging and the status write). Two
  concurrent cancels of the same booking both saw `status == "confirmed"`, both got
  200, and **two** RefundLog rows were written. Spec: exactly one RefundLog; the
  second cancel → 409 `ALREADY_CANCELLED`.
- **Fix:** the whole read-check-refund-update-commit sequence now runs under
  `_booking_lock`, so the second cancel re-reads the committed `cancelled` status and
  gets 409.

## Caches / reporting freshness

### 17. Usage report not invalidated on booking creation; availability not invalidated on cancel
- **Files:** `app/routers/bookings.py` (`create_booking`, `cancel_booking`),
  `app/routers/rooms.py` (`create_room`)
- **Bug:** cached responses went stale: creating a booking never invalidated the org's
  usage-report cache; cancelling never invalidated the room's availability cache for
  the booking's date; creating a room never invalidated the report cache (reports must
  include zero-booking rooms). Spec: reports and availability reflect the current
  state immediately.
- **Fix:** `create_booking` also calls `cache.invalidate_report(org_id)`;
  `cancel_booking` also calls `cache.invalidate_availability(room_id, start_date)`;
  `create_room` calls `cache.invalidate_report(org_id)`.

## Concurrency in services

### 18. Rate limiter lost updates under concurrency
- **File:** `app/services/ratelimit.py` (`record_and_check`)
- **Bug:** each request copied the user's bucket, slept 100 ms (`_settle_pause`), then
  wrote the copy back. Concurrent requests each appended to their own stale copy and
  overwrote one another, so the counter undercounted and the 20/60 s limit was never
  enforced under load.
- **Fix:** trim + append + store now happen atomically under a `threading.Lock`
  (the bookkeeping pause moved outside the critical section).

### 19. Duplicate reference codes under concurrent creation
- **File:** `app/services/reference.py` (`next_reference_code`)
- **Bug:** read of the counter and the increment were separated by a 120 ms sleep;
  concurrent creations read the same counter value and issued identical codes.
  Spec: reference codes unique, including under concurrent creation.
- **Fix:** the read-increment is atomic under a `threading.Lock`; formatting pause
  moved outside.

### 20. Room stats drifted under concurrent activity
- **File:** `app/services/stats.py` (`record_create`, `record_cancel`)
- **Bug:** same lost-update pattern: read count/revenue, sleep 100 ms, write back.
  Concurrent bookings/cancellations overwrote each other's updates, so
  `/rooms/{id}/stats` diverged from the values derivable from the bookings.
- **Fix:** the read-modify-write runs under a `threading.Lock`; pause moved outside.

### 21. Deadlock between create and cancel notifications (liveness)
- **File:** `app/services/notifications.py` (`notify_cancelled`)
- **Bug:** `notify_created` acquired `_email_lock` → `_audit_lock`, while
  `notify_cancelled` acquired `_audit_lock` → `_email_lock`. A concurrent create +
  cancel could each grab one lock and wait forever on the other, permanently hanging
  those worker threads (and eventually the whole service) — violating the liveness
  rule.
- **Fix:** both functions now acquire the locks in the same order
  (email, then audit).

## Multi-tenancy

### 22. CSV export leaked other organizations' bookings
- **File:** `app/services/export.py`, lines 22–29 / 48–52 (`generate_export`)
- **Bug:** with `include_all=true&room_id=<id>`, the export used
  `fetch_bookings_raw`, which filtered by room id only — no org scoping. An admin
  could pass another org's room id and download that org's booking data (cross-tenant
  leak).
- **Fix:** all export paths go through the org-scoped query
  (`_fetch_scoped(db, org_id, …, room_id)`); the unscoped helper was removed.

## Tooling

### 23. Smoke test not runnable as documented
- **File:** `requirements.txt`
- **Bug:** the README says `pip install -r requirements.txt` then `pytest`, but
  neither `pytest` nor `httpx` (required by FastAPI's `TestClient`) was in
  `requirements.txt`, so the documented test command failed with import errors.
- **Fix:** added `pytest` and `httpx` pins.
