# Cache Design

## Objective

The ShowTime ticketing platform expects up to 500,000 concurrent users during ticket launches. Without caching, every request would hit PostgreSQL directly, creating excessive database load and increasing latency.

Our caching strategy aims to:

* Reduce database reads
* Keep API latency below 500ms
* Protect PostgreSQL from traffic spikes
* Maintain acceptable data freshness
* Stay within the $2,000/month infrastructure budget

Redis is used as the primary caching layer.

---

# Cache Strategy Overview

| Data Type               | Cache? | TTL        | Invalidation       |
| ----------------------- | ------ | ---------- | ------------------ |
| Event Details           | Yes    | 1 Hour     | Event Update       |
| Seat Availability Count | Yes    | 30 Seconds | Seat Status Change |
| Seat Map Layout         | Yes    | 24 Hours   | Event Cancellation |
| Individual Seat Status  | No     | N/A        | N/A                |

---

# 1. Event Details Cache

## Redis Key

```text id="7h4j9g"
event:{event_id}
```

Example:

```text id="ddrh0z"
event:1001
```

---

## Cached Data

```json id="58ez2y"
{
  "eventId": 1001,
  "name": "Coldplay India Tour",
  "venue": "DY Patil Stadium",
  "date": "2026-01-20T18:00:00Z",
  "status": "on_sale"
}
```

---

## TTL

```text id="zuwk4l"
3600 seconds (1 hour)
```

### Why 1 Hour?

Event information changes very rarely.

Possible updates:

* Venue change
* Event cancellation
* Sale status update

Caching for 1 hour dramatically reduces read traffic while keeping data reasonably fresh.

---

## Invalidation Trigger

Invalidate when:

```text id="qiw56d"
events table UPDATE
```

Example:

```sql id="jstp2h"
UPDATE events
SET status = 'cancelled'
WHERE id = 1001;
```

Then:

```text id="r69eww"
DEL event:1001
```

---

## Strategy

```text id="jstx9v"
TTL + Event-Driven Invalidation
```

---

# 2. Seat Availability Count Cache

## Redis Key

```text id="f7sn8q"
availability:{event_id}:{category}
```

Examples:

```text id="nhv1iu"
availability:1001:VIP
availability:1001:Premium
availability:1001:General
```

---

## Cached Data

```json id="n1c6ot"
{
  "availableSeats": 42
}
```

---

## TTL

```text id="l44s8q"
30 seconds
```

### Why 30 Seconds?

#### Why Not 5 Seconds?

Problems:

* Frequent cache regeneration
* Increased database load

#### Why Not 5 Minutes?

Problems:

* Seat counts become highly inaccurate
* Poor customer experience

#### Why 30 Seconds?

Balance between:

* Freshness
* Reduced database reads
* User experience

A seat count being off by a few seats for a short period is acceptable.

---

## Invalidation Trigger

Any change to seat status:

```text id="0kydxv"
available → held
held → booked
held → available
booked → refunded
```

Example:

```sql id="v7kpsa"
UPDATE seats
SET status = 'booked'
WHERE id = 5001;
```

Invalidate:

```text id="0lrspg"
DEL availability:1001:VIP
```

---

## Strategy

```text id="lfsm6k"
TTL + Event-Driven Invalidation
```

This ensures the cache never remains stale for long.

---

# 3. Seat Map Layout Cache

## Redis Key

```text id="9xezfy"
seatmap:{event_id}
```

Example:

```text id="3fuh6r"
seatmap:1001
```

---

## Cached Data

```json id="wzeb4f"
{
  "sections": [
    {
      "name": "VIP",
      "rows": ["A","B","C"]
    },
    {
      "name": "General",
      "rows": ["D","E","F"]
    }
  ]
}
```

---

## TTL

```text id="m5m6l7"
86400 seconds (24 hours)
```

### Why 24 Hours?

Seat layouts are effectively static.

They only change if:

* Event configuration changes
* Venue configuration changes
* Event is cancelled

Long TTL significantly reduces repetitive reads.

---

## Invalidation Trigger

Only:

```text id="v4o5wd"
Event cancellation
Venue seating reconfiguration
```

Example:

```text id="r5a8ij"
DEL seatmap:1001
```

---

## Strategy

```text id="y7gns9"
Long TTL + Rare Event Invalidation
```

---

# What We Explicitly Do NOT Cache

## Individual Seat Status

Examples:

```text id="g1m2e9"
Seat A12 = available
Seat A13 = booked
Seat A14 = held
```

---

## Why Not?

Consider:

```text id="gls6c8"
User A reads cache
Seat A12 = available

User B reads cache
Seat A12 = available
```

Both proceed to checkout.

Even if concurrency controls eventually prevent double-booking, users experience:

* Failed checkouts
* Increased contention
* Poor user experience

Seat state changes too frequently to be safely cached.

---

## Source of Truth

Seat status always comes from:

1. Redis lock state
2. PostgreSQL seat table

Never from a cached seat-status object.

---

# Cache Invalidation Strategy

## Chosen Pattern: Cache-Aside

We use:

```text id="kzqg5v"
Cache Aside Pattern
```

Process:

```text id="tgjm6g"
Read:
Cache -> DB -> Cache

Write:
DB -> Delete Cache
```

---

## Why Cache-Aside?

Advantages:

* Simple implementation
* Database remains source of truth
* Prevents cache/data inconsistency
* Lower operational complexity

---

## Why Not Write-Through?

Write-through requires:

```text id="w6mt1x"
Update DB
Update Redis
```

Potential issues:

* Two write operations
* Partial failure handling
* Additional complexity

For ShowTime, cache-aside is sufficient.

---

# Cache Invalidation Workflow

## When Seat Status Changes

Example:

```text id="hmf6zu"
Seat A12
available → booked
```

### Step 1

Update database.

```sql id="oq8np4"
UPDATE seats
SET status = 'booked'
WHERE id = 5001;
```

---

### Step 2

Delete cache.

```text id="fzmgr4"
DEL availability:1001:VIP
```

---

### Step 3

Next Request

Cache miss occurs.

```text id="75lfm9"
GET availability:1001:VIP
```

Returns:

```text id="cx1hks"
NULL
```

---

### Step 4

Application queries PostgreSQL.

```sql id="l18s89"
SELECT COUNT(*)
FROM seats
WHERE event_id = 1001
AND category = 'VIP'
AND status = 'available';
```

---

### Step 5

Cache is rebuilt.

```text id="gqez2n"
SET availability:1001:VIP
TTL 30s
```

---

# Final Cache Design Summary

## Cached

### Event Details

```text id="0pafqg"
event:{event_id}
TTL: 1 hour
```

### Availability Counts

```text id="gh4mlv"
availability:{event_id}:{category}
TTL: 30 seconds
```

### Seat Maps

```text id="y9s0rl"
seatmap:{event_id}
TTL: 24 hours
```

---

## Not Cached

```text id="h4gq8s"
Individual seat availability
```

Reason:

```text id="ok1l0y"
High write frequency
Race-condition risk
Potential stale booking information
```

---

## Overall Decision

The cache layer uses Redis with a cache-aside strategy and targeted event-driven invalidation.

This approach:

* Minimizes database load
* Preserves booking correctness
* Provides low-latency reads
* Supports 500,000 concurrent users
* Fits within the allocated AWS budget
