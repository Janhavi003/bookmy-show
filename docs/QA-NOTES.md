# QA Notes - Panel Discussion

## Question 1

### Question

What happens if Redis crashes after a seat lock is acquired but before the booking is completed?

### My Answer

This is a realistic failure scenario because Redis is a critical component in the concurrency layer.

In my design, Redis is used only for high-speed lock acquisition. PostgreSQL remains the source of truth.

If Redis crashes after a lock is acquired:

1. New lock requests may fail.
2. Existing Redis lock state may be lost.
3. However, bookings are still validated against PostgreSQL.
4. The seat record contains status and version information.
5. Optimistic locking prevents conflicting updates.

Therefore Redis failure impacts availability and throughput but should not create double-bookings.

### Design Limitation

Booking success rates may decrease while Redis is unavailable.

### Future Improvement

Deploy Redis with Multi-AZ failover and automatic recovery.

### Self Evaluation

Complete Answer: Yes

---

## Question 2

### Question

At what traffic level does PostgreSQL become the bottleneck?

### My Answer

I calculated this during the concurrency analysis.

Using:

* Maximum connections = 500
* 80% normal requests at 20ms
* 20% payment requests at 800ms

Formula:

Connections Held =
(RPS × 0.016)
+
(RPS × 0.16)

Connections Held =
RPS × 0.176

Setting this equal to 500 gives:

RPS ≈ 2,841

Expected launch traffic is approximately 25,000 RPS.

This is why I avoided a pure PostgreSQL locking strategy and adopted Redis SETNX plus PostgreSQL validation.

### Design Limitation

The primary database remains a scaling bottleneck for writes.

### Future Improvement

Database sharding or reservation-service decomposition.

### Self Evaluation

Complete Answer: Yes

---

## Question 3

### Question

What prevents one user from holding 200 seats and blocking everyone else?

### My Answer

This is a valid abuse scenario during ticket sales.

I would implement a per-user seat hold limit.

Example:

```text
Maximum held seats per user = 6
```

Redis key:

```text
holds:{userId}:count
```

Before creating a seat hold:

1. Read current hold count.
2. Reject request if limit exceeded.
3. Increment count on successful hold.
4. Decrement count when booking completes or expires.

This prevents seat hoarding and improves fairness.

### Design Limitation

Users could attempt to create multiple accounts.

### Future Improvement

Phone verification and fraud-detection rules.

### Self Evaluation

Complete Answer: Yes

---

## Question 4

### Question

Your AWS bill reaches $3,200 this month. How would you reduce costs?

### My Answer

First I would identify the cost driver.

Possible contributors:

* Excess API instances
* Overprovisioned Redis cluster
* Database replicas
* CloudFront bandwidth

Immediate actions:

1. Tighten auto-scaling thresholds.
2. Reduce maximum API server count.
3. Increase cache hit rates.
4. Optimize expensive database queries.
5. Review unused infrastructure.

Because event traffic is highly bursty, improving cache efficiency usually reduces both compute and database costs.

### Design Limitation

Aggressive cost reductions may reduce peak capacity.

### Future Improvement

Scheduled scaling based on known ticket-launch windows.

### Self Evaluation

Complete Answer: Yes

---

## Question 5

### Question

Why not use PostgreSQL SELECT FOR UPDATE instead of Redis locks?

### My Answer

I considered PostgreSQL row-level locking carefully.

The advantage is simplicity and strong consistency.

However, the scalability analysis showed that PostgreSQL becomes the bottleneck at roughly 2,841 RPS.

Expected launch traffic is approximately 25,000 RPS.

Using PostgreSQL locks alone would force thousands of requests to wait for database resources.

Redis SETNX provides much higher throughput for lock acquisition while PostgreSQL remains responsible for final validation and persistence.

The hybrid approach combines scalability and correctness.

### Design Limitation

The architecture is more complex than a database-only design.

### Future Improvement

For small-scale events with low traffic, pure PostgreSQL locking could be simpler and cheaper.

### Self Evaluation

Complete Answer: Yes

---

# Key Lessons From Q&A

1. Correctness and scalability often require different technologies.
2. Redis improves throughput but PostgreSQL remains the source of truth.
3. Async processing prevents payment latency from exhausting database resources.
4. Cost optimization must be balanced against peak traffic requirements.
5. Every architecture decision involves tradeoffs and revision triggers.
