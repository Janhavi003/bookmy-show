# Design Decisions

## Decision: Hybrid Concurrency Strategy (Redis SETNX + PostgreSQL)

**Context:**
The system must support 500,000 concurrent users during flash sales while guaranteeing zero double-bookings.

**Options considered:**

1. **PostgreSQL SELECT FOR UPDATE**

   * Considered because it provides strong consistency and prevents concurrent modifications.
   * Not chosen because connection pool exhaustion occurs at approximately 2,841 RPS, which is far below the expected launch traffic of ~25,000 RPS.

2. **Hybrid Strategy: Redis SETNX + PostgreSQL Optimistic Locking (Chosen)**

   * Redis handles high-contention seat acquisition.
   * PostgreSQL remains the source of truth and validates final booking state.

**Why chosen:**
Part A capacity analysis showed that PostgreSQL row-level locking becomes a bottleneck long before expected traffic levels. Redis SETNX provides sub-millisecond lock acquisition while PostgreSQL optimistic locking guarantees correctness. This combination supports high throughput while remaining within the $2,000/month infrastructure budget.

**Tradeoffs accepted:**

* Redis becomes a critical dependency.
* Redis outages reduce availability.
* Additional operational complexity compared to a database-only solution.

**Revision trigger:**
If the system expands to multi-region deployments or millions of concurrent users, I would evaluate Redis Redlock, DynamoDB conditional writes, or a dedicated reservation service.

---

## Decision: Cache Invalidation Strategy

**Context:**
Event pages and seat availability information are read-heavy. Direct database reads for every request would overload PostgreSQL during peak sales.

**Options considered:**

1. **TTL-Only Caching**

   * Simple to implement and maintain.
   * Not chosen because stale availability information could remain visible until expiration.

2. **Cache-Aside with Event-Driven Invalidation (Chosen)**

   * Database remains the source of truth.
   * Cache entries are deleted when relevant data changes.
   * Next request rebuilds the cache.

**Why chosen:**
This approach balances freshness and simplicity. Seat availability caches use a 30-second TTL while also being invalidated immediately when seat status changes. It significantly reduces database load while keeping data reasonably accurate.

**Tradeoffs accepted:**

* Temporary cache misses after invalidation.
* Additional cache management logic.
* Availability counts may still be slightly stale for a short period.

**Revision trigger:**
If stale availability information causes measurable user dissatisfaction, I would consider real-time updates using Redis Pub/Sub or WebSockets.

---

## Decision: UUID for Booking IDs

**Context:**
Booking IDs must uniquely identify orders and should not expose predictable sequences.

**Options considered:**

1. **SERIAL / BIGSERIAL**

   * Easy to implement and efficient for indexing.
   * Not chosen because IDs become predictable and can be enumerated.

2. **UUID (Chosen)**

   * Globally unique.
   * Difficult to guess.
   * Supports distributed systems.

**Why chosen:**
UUIDs improve security and support future horizontal scaling. Booking IDs can be generated independently of database inserts, making them useful for idempotent APIs and distributed workflows.

**Tradeoffs accepted:**

* Larger index sizes.
* Slightly slower index performance compared to sequential integers.
* Less human-readable identifiers.

**Revision trigger:**
If storage and indexing overhead become significant at very large scale, I would evaluate ULIDs or Snowflake-style IDs.

---

## Decision: SQS Visibility Timeout = 30 Seconds

**Context:**
Payment workers require enough time to process messages without creating duplicate payment attempts.

**Options considered:**

1. **5 Seconds**

   * Fast retry behaviour.
   * Not chosen because legitimate payment processing may exceed the timeout.

2. **60 Seconds**

   * Reduces risk of duplicate processing.
   * Not chosen because failed messages would take too long to become available for retry.

3. **30 Seconds (Chosen)**

   * Covers expected payment processing duration.
   * Allows recovery from worker crashes.
   * Provides reasonable retry timing.

**Why chosen:**
Part A analysis showed payment processing typically completes within 200ms–2000ms. A 30-second visibility timeout provides sufficient safety margin while maintaining responsiveness for retries and failure recovery.

**Tradeoffs accepted:**

* Messages may remain hidden longer than necessary after worker failure.
* Long-running payment operations may require visibility timeout extensions.

**Revision trigger:**
If payment latency increases significantly or a new payment provider introduces longer processing times, I would increase the timeout or implement dynamic visibility timeout extension.

---

# Summary

| Decision       | Chosen Option                    | Primary Reason                                       |
| -------------- | -------------------------------- | ---------------------------------------------------- |
| Concurrency    | Redis SETNX + PostgreSQL         | Supports ~25,000 RPS while preventing double-booking |
| Cache Strategy | Cache-Aside + Event Invalidation | Balances freshness and performance                   |
| Booking IDs    | UUID                             | Security and distributed-system compatibility        |
| Queue Timeout  | 30 Seconds                       | Safe payment processing with efficient retries       |

These decisions collectively support the system goals of scalability, correctness, availability, and cost efficiency while remaining aligned with the constraints established in Part A.
