# Concurrency Strategy Design

## Problem Statement

ShowTime must support:

* 500,000 concurrent users
* Zero double-bookings
* API response time below 500ms
* AWS budget of $2,000/month

The most critical challenge is ensuring that only one user can successfully reserve a seat when thousands of users attempt to book it simultaneously.

---

# Option A: PostgreSQL Row-Level Locking (SELECT FOR UPDATE)

## How It Prevents Double-Booking

The booking process runs inside a transaction.

```sql
BEGIN;

SELECT id, status
FROM seats
WHERE id = $1
AND status = 'available'
FOR UPDATE;

UPDATE seats
SET status = 'held',
    held_until = NOW() + INTERVAL '10 minutes',
    held_by = $2
WHERE id = $1;

INSERT INTO bookings (
    user_id,
    event_id,
    status,
    total_amount
)
VALUES (
    $3,
    $4,
    'pending',
    $5
);

COMMIT;
```

### Why It Works

* The first transaction acquires the row lock.
* All subsequent transactions attempting to access the same seat must wait.
* Only one transaction can update the seat.
* Double-booking becomes impossible.

---

# Capacity Analysis

Given:

```text
max_connections = 500
20% payment requests
80% normal requests

payment request duration = 0.8 sec
normal request duration = 0.02 sec
```

Formula:

```text
Connections Held =
(RPS × 80% × 0.02)
+
(RPS × 20% × 0.8)
```

Substituting:

```text
Connections Held =
(RPS × 0.016)
+
(RPS × 0.16)

Connections Held =
RPS × 0.176
```

Pool exhaustion occurs when:

```text
RPS × 0.176 = 500
```

Therefore:

```text
RPS = 500 / 0.176

RPS ≈ 2841
```

### Result

The PostgreSQL connection pool becomes exhausted at approximately:

**2,841 RPS**

This is dramatically below the expected launch traffic of approximately 25,000 RPS.

---

# Deadlock Risk

Consider:

User A books:

```text
Seat A12
Seat A13
```

User B books:

```text
Seat A13
Seat A12
```

Potential sequence:

```text
User A locks A12
User B locks A13

User A waits for A13
User B waits for A12
```

Deadlock occurs.

### Mitigation

Always lock seats in deterministic order.

Example:

```text
Sort seat IDs ascending

A12
A13
A14
```

All transactions acquire locks in the same order.

This removes circular wait conditions.

---

# Option B: Redis Distributed Lock (SETNX)

## Lock Structure

Redis key:

```text
seat_lock:{event_id}:{seat_id}
```

Example:

```text
seat_lock:1001:432
```

Including event_id prevents collisions across events.

---

## Lock Acquisition

```javascript
const acquired = await redis.set(
  lockKey,
  lockValue,
  'NX',
  'EX',
  30
);
```

Where:

```text
NX = Set only if key does not exist
EX = Expiry in seconds
```

If lock acquisition fails:

```json
{
  "error": "SEAT_TEMPORARILY_UNAVAILABLE"
}
```

---

## Safe Lock Release

```lua
if redis.call("get", KEYS[1]) == ARGV[1]
then
    return redis.call("del", KEYS[1])
else
    return 0
end
```

This prevents a user from deleting another user's lock.

---

## How It Prevents Double-Booking

Flow:

1. User attempts booking.
2. Redis lock acquired.
3. Only lock holder can access booking workflow.
4. Database verifies seat availability.
5. Seat status changes to held.
6. Lock released.

All other users fail immediately.

No waiting queue forms.

---

# What If Redis Fails Mid-Lock?

Redis failure does not automatically create double-bookings.

Reason:

PostgreSQL remains the source of truth.

Booking confirmation still requires:

```text
Seat available?
Version matches?
Database transaction succeeds?
```

Even if Redis becomes unavailable:

* Throughput decreases.
* Booking requests may fail.
* Seat integrity remains protected by PostgreSQL validation.

Therefore:

```text
Redis failure = availability issue

NOT

Redis failure = correctness issue
```

---

# Choosing the Lock TTL

We choose:

```text
TTL = 30 seconds
```

### Too Short

Example:

```text
TTL = 5 seconds
```

Risk:

* Lock expires during DB write.
* Another user acquires lock.
* Increased race conditions.

### Too Long

Example:

```text
TTL = 300 seconds
```

Risk:

* Abandoned locks block seats.
* Poor customer experience.

### Why 30 Seconds?

* Long enough for DB operations.
* Short enough to recover from crashes.
* Matches expected booking initiation duration.

---

# Chosen Strategy

## We Choose a Hybrid Approach

### Redis SETNX

Used for:

* Initial seat acquisition
* High-volume contention management

### PostgreSQL

Used for:

* Final booking confirmation
* Optimistic locking via version column
* Transactional consistency

---

# Why Not Pure PostgreSQL?

Pure PostgreSQL reaches its connection limit at:

```text
~2,841 RPS
```

Expected launch traffic:

```text
~25,000 RPS
```

This creates a significant scalability gap.

The database becomes the bottleneck long before launch traffic is reached.

---

# Why Not Pure Redis?

Redis alone cannot be the source of truth.

Problems:

* Locks expire.
* Redis nodes can fail.
* Seat ownership must be persisted.

Final correctness still requires database validation.

---

# Budget Consideration

Budget:

```text
$2,000/month
```

A Redis cluster is affordable within the allocated architecture budget.

Approximate allocation:

```text
Redis Cluster
≈ $359/month
```

This provides enough throughput to handle seat lock traffic while remaining under budget.

---

# Known Limitations

The hybrid approach still has limits.

Cannot fully handle:

* Complete Redis cluster outage
* Regional AWS outages
* Payment gateway failures
* Massive flash-sale traffic beyond planned capacity

Fallback behavior:

* Booking attempts may fail.
* Users may retry.
* Double-bookings still remain prevented.

---

# When Would We Switch Strategies?

## Switch to Pure PostgreSQL If

Conditions:

* Traffic below 5,000 booking attempts/sec
* Smaller events
* Extremely limited budget
* Simpler operational requirements

Benefits:

* Fewer moving parts
* Lower operational complexity

---

## Switch to More Advanced Distributed Locking If

Conditions:

* Multi-region deployment
* Millions of concurrent users
* Global ticket sales

Potential upgrades:

* Redis Redlock
* DynamoDB conditional writes
* Dedicated reservation service

---

# Final Decision

ShowTime adopts a Hybrid Concurrency Strategy:

1. Redis SETNX distributed locks handle high-contention seat acquisition.
2. PostgreSQL optimistic locking and transactions guarantee correctness.
3. Redis provides scalability.
4. PostgreSQL provides consistency.
5. The solution remains within the $2,000/month AWS budget.

This architecture provides the best balance between throughput, correctness, operational complexity, and cost.
