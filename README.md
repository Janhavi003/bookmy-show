# ShowTime Ticketing System - Constraints Analysis

## Project Overview

ShowTime is a ticket booking platform designed to handle extremely high-demand events such as Coldplay concerts and IPL finals. The system must support 5 lakh concurrent users, prevent all double-bookings, maintain API response times below 500ms, and operate within a monthly AWS budget of $2,000.

---

# Constraint 1: 5 Lakh Concurrent Users at 12:00:00 Noon

## Peak Request Per Second (RPS)

Assumption:

* 500,000 users attempt booking simultaneously.
* Each user generates approximately 3 API requests during the first minute:

  * Event details request
  * Seat selection request
  * Booking request

Peak RPS:

RPS = (500,000 × 3) / 60

RPS ≈ 25,000 requests/second

This represents the expected peak load during ticket launch.

## What Hits Its Limit First?

The primary bottleneck is the PostgreSQL database connection pool.

Reasons:

* Every booking request requires seat validation.
* Every booking request performs writes.
* Traditional row-level locking increases connection hold time.
* PostgreSQL typically supports a limited number of active connections (approximately 500 with PgBouncer).

At very high concurrency, requests begin waiting for available database connections, causing increased latency and possible failures.

Secondary bottlenecks include:

* Payment gateway latency
* Redis lock contention on highly demanded seats
* API server CPU utilization

---

# Constraint 2: Zero Acceptable Double-Bookings

## What Is a Double-Booking?

A double-booking occurs when the same seat is successfully assigned to more than one booking.

Example:

booking_seats

Booking A → Seat A12

Booking B → Seat A12

Both bookings appear confirmed.

This creates legal, financial, and customer trust issues.

## Mechanisms to Prevent Double-Booking

To guarantee correctness, the system uses a hybrid concurrency strategy:

### Redis Distributed Lock (SETNX)

Purpose:

* Prevent multiple users from simultaneously attempting to reserve the same seat.

Example Lock Key:

seat_lock:{event_id}:{seat_id}

Benefits:

* Extremely fast
* Sub-millisecond lock acquisition
* Handles high concurrency

### PostgreSQL Transaction Validation

After obtaining the Redis lock:

* Verify seat status in PostgreSQL.
* Update seat status within a database transaction.
* Use optimistic locking via the version column.

This ensures the database remains the final source of truth.

---

# Constraint 3: $2,000 Monthly AWS Budget

## Infrastructure Supported by $2,000

Approximate monthly allocation:

| Component              | Configuration       | Cost    |
| ---------------------- | ------------------- | ------- |
| EC2 API Servers        | 6 × t3.xlarge       | $719    |
| PostgreSQL Primary     | db.r6g.xlarge       | $262    |
| PostgreSQL Replicas    | 2 × db.r6g.large    | $262    |
| Redis Cluster          | 3 × cache.r6g.large | $359    |
| ECS Payment Workers    | Auto-scaled Fargate | $73     |
| ALB + CloudFront + SQS | Managed Services    | $165    |
| Total                  |                     | ~$1,840 |

This leaves a small operational buffer while remaining under budget.

---

# How the Constraints Interact

## Speed vs Correctness

Stronger locking improves correctness but increases latency.

Examples:

* Table lock → safest but slowest
* Row lock → better
* Redis distributed lock → highest throughput

The design must balance response time and data integrity.

## Scale vs Budget

Unlimited scale could be achieved by continuously adding servers.

However, the $2,000 budget prevents horizontal scaling without limits.

Therefore the architecture must:

* Cache aggressively
* Use asynchronous processing
* Minimize database reads
* Reduce connection hold times

## Correctness vs Budget

The most fault-tolerant architecture would use larger Redis clusters, additional replicas, and multi-region deployments.

These options exceed the available budget.

Therefore the system adopts:

* Redis distributed locking
* PostgreSQL optimistic locking
* SQS asynchronous processing

to maximize correctness within financial constraints.

---

# What Would Change If Budget Were Reduced to $500?

Several architectural decisions would need modification:

### Remove Read Replicas

All reads and writes would use a single PostgreSQL instance.

Tradeoff:

* Increased database load
* Higher latency

### Reduce Redis Infrastructure

Move from a multi-node Redis cluster to a single-node Redis deployment.

Tradeoff:

* Lower fault tolerance
* Higher risk during node failure

### Reduce API Server Count

Fewer EC2 instances.

Tradeoff:

* Lower peak throughput

### More Aggressive Caching

Longer cache TTLs would reduce database traffic.

Tradeoff:

* Increased data staleness

The system would still function but would support significantly less traffic and provide weaker reliability guarantees.

---

# Final Design Principles

1. Use Redis locks to handle extreme concurrency.
2. Use PostgreSQL as the source of truth.
3. Use optimistic locking to prevent conflicts.
4. Process payments asynchronously through SQS.
5. Cache heavily accessed data while avoiding caching individual seat states.
6. Stay within the $2,000 monthly infrastructure budget.

These principles guide all subsequent design decisions in the system architecture.
