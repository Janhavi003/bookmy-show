# ShowTime Architecture

## System Architecture Diagram

```text
                                     ┌──────────────────────┐
                                     │ Browser / Mobile App │
                                     └──────────┬───────────┘
                                                │
                                                │ HTTPS Requests
                                                ▼

┌───────────────────────────────────────────────────────────────────────┐
│                           CloudFront CDN                             │
├───────────────────────────────────────────────────────────────────────┤
│ Serves:                                                               │
│ • Static Assets (JS/CSS/Images)                                       │
│ • Cached Event Pages                                                  │
│                                                                        │
│ Cache Hit  → Return Cached Response                                   │
│ Cache Miss → Forward Request to Origin                                │
│                                                                        │
│ Part A Reference: CACHE.md                                            │
└───────────────────────────────┬────────────────────────────────────────┘
                                │
                                │ Dynamic API Requests
                                ▼

┌───────────────────────────────────────────────────────────────────────┐
│                    Application Load Balancer (ALB)                    │
├───────────────────────────────────────────────────────────────────────┤
│ • SSL Termination                                                     │
│ • Health Check Interval = 10 sec                                      │
│ • Rate Limit Rule = 200 req/IP/min                                    │
│ • Route /api/* requests to Node.js API Cluster                        │
│                                                                        │
│ Part A Reference: CONCURRENCY.md                                      │
└───────────────────────────────┬────────────────────────────────────────┘
                                │
                                ▼

┌───────────────────────────────────────────────────────────────────────┐
│                  Node.js API Auto Scaling Group                       │
├───────────────────────────────────────────────────────────────────────┤
│ Instances: 4 → 20                                                     │
│ Scale Trigger: CPU > 70%                                              │
│                                                                        │
│ Responsibilities:                                                     │
│ • Event APIs                                                          │
│ • Seat Selection                                                      │
│ • Booking APIs                                                        │
│ • Redis Cache Reads                                                   │
│ • Redis Lock Acquisition                                              │
│ • PostgreSQL Writes                                                   │
│ • Publish Payment Jobs to SQS                                         │
│                                                                        │
│ Part A Reference: QUEUE.md                                            │
└───────────────┬───────────────────────────────┬───────────────────────┘
                │                               │
                │ Cache / Locks                 │ Writes
                ▼                               ▼

┌─────────────────────────────┐      ┌─────────────────────────────┐
│        Redis Cluster        │      │     PostgreSQL Primary      │
├─────────────────────────────┤      ├─────────────────────────────┤
│                             │      │                             │
│ CACHE LAYER                 │      │ WRITE OPERATIONS            │
│                             │      │                             │
│ event:{eventId}             │      │ • Create Booking            │
│ TTL = 1 hour                │      │ • Reserve Seats             │
│                             │      │ • Confirm Payment           │
│ availability:{event}:{cat}  │      │ • Update Booking Status     │
│ TTL = 30 sec                │      │                             │
│                             │      │ Source of Truth             │
│ LOCK LAYER                  │      │                             │
│                             │      │ Optimistic Locking          │
│ seat_lock:{event}:{seat}    │      │ via version column          │
│ SETNX                       │      │                             │
│ TTL = 30 sec                │      │                             │
│                             │      │                             │
│ UPDATE #1                   │      │                             │
│ holds:{userId}:count        │      │                             │
│ Max Holds = 8 Seats         │      │                             │
│                             │      │                             │
│ Part A References:          │      │ Part A References:          │
│ CACHE.md                    │      │ SCHEMA.md                   │
│ CONCURRENCY.md              │      │ CONCURRENCY.md              │
└──────────────┬──────────────┘      └──────────────┬──────────────┘
               │                                    │
               │ Replication                        │
               ▼                                    ▼

       ┌────────────────────┐            ┌────────────────────┐
       │ PostgreSQL Replica │            │ PostgreSQL Replica │
       │        #1          │            │        #2          │
       └────────────────────┘            └────────────────────┘

                Read Queries:
                • Event Details
                • Seat Maps
                • Availability Browsing
                • Booking History

                Part A Reference: SCHEMA.md

                                │
                                ▼

┌───────────────────────────────────────────────────────────────────────┐
│                     Queue Circuit Breaker                             │
├───────────────────────────────────────────────────────────────────────┤
│ UPDATE #2                                                             │
│                                                                        │
│ Trigger: SQS Publish Failures > 60 seconds                            │
│                                                                        │
│ Fallback:                                                             │
│ • Reduced Throughput                                                  │
│ • Synchronous Payment Processing                                      │
│ • Alert Operations Team                                               │
│                                                                        │
│ Purpose: Prevent complete booking outage                              │
└───────────────────────────────┬────────────────────────────────────────┘
                                │
                                ▼

┌───────────────────────────────────────────────────────────────────────┐
│                         SQS Payment Queue                             │
├───────────────────────────────────────────────────────────────────────┤
│ Message Format:                                                       │
│                                                                        │
│ {                                                                      │
│   bookingId,                                                           │
│   userId,                                                              │
│   eventId,                                                             │
│   seatIds,                                                             │
│   totalAmount,                                                         │
│   paymentToken,                                                        │
│   idempotencyKey                                                       │
│ }                                                                      │
│                                                                        │
│ Visibility Timeout = 30 sec                                            │
│ Max Receive Count = 3                                                  │
│ Dead Letter Queue Enabled                                              │
│                                                                        │
│ Part A Reference: QUEUE.md                                             │
└───────────────────────────────┬────────────────────────────────────────┘
                                │
                                │ Consume Messages
                                ▼

┌───────────────────────────────────────────────────────────────────────┐
│                    ECS Fargate Payment Workers                        │
├───────────────────────────────────────────────────────────────────────┤
│ 1. Read SQS Message                                                   │
│ 2. Validate Idempotency Key                                           │
│ 3. Call Payment Gateway                                               │
│ 4. Update PostgreSQL                                                  │
│ 5. Publish SNS Event                                                  │
│ 6. Delete SQS Message                                                 │
│                                                                        │
│ Success Path:                                                         │
│ Pending → Confirmed                                                   │
│ Seats → Booked                                                        │
│                                                                        │
│ Failure Path:                                                         │
│ Pending → Failed                                                      │
│ Seats Released                                                        │
│                                                                        │
│ Part A Reference: QUEUE.md                                            │
└───────────────────────────────┬────────────────────────────────────────┘
                                │
                                ▼

┌───────────────────────────────────────────────────────────────────────┐
│                              AWS SNS                                  │
├───────────────────────────────────────────────────────────────────────┤
│ Trigger: Booking Confirmed                                            │
│                                                                        │
│ Fan-out Notifications                                                 │
└─────────────────────┬───────────────────────────────┬─────────────────┘
                      │                               │
                      ▼                               ▼

         ┌──────────────────────┐      ┌────────────────────────┐
         │       AWS SES        │      │      SMS Provider      │
         ├──────────────────────┤      ├────────────────────────┤
         │ Booking Confirmation │      │ Ticket Confirmation    │
         │ Email                │      │ SMS                    │
         └──────────────────────┘      └────────────────────────┘


┌───────────────────────────────────────────────────────────────────────┐
│                       CloudWatch Monitoring                           │
├───────────────────────────────────────────────────────────────────────┤
│ UPDATE #3                                                             │
│                                                                        │
│ QueueDepth > 10,000                                                   │
│ OldestMessageAge > 120 sec                                            │
│ ECS Worker Failure Alerts                                             │
│ Redis Memory Alerts                                                   │
│ RDS CPU Alerts                                                        │
│                                                                        │
│ Actions:                                                              │
│ • Slack Alert                                                         │
│ • PagerDuty Alert                                                     │
│ • Auto-scale Payment Workers                                          │
└───────────────────────────────────────────────────────────────────────┘
```

---

# Booking Flow

1. User opens an event page.
2. CloudFront serves cached content or forwards request.
3. API checks Redis cache.
4. User selects seats.
5. Redis SETNX acquires seat lock.
6. Redis hold counter validates the user has fewer than 8 active seat holds.
7. Booking is created in PostgreSQL Primary.
8. API publishes payment job to SQS.
9. Payment Worker processes payment.
10. PostgreSQL updates booking and seat status.
11. SNS publishes booking confirmation event.
12. SES Email and SMS notifications are sent.

---

# Part A Design References

| Component                  | Related Document |
| -------------------------- | ---------------- |
| Database Schema            | SCHEMA.md        |
| Redis Concurrency Locking  | CONCURRENCY.md   |
| Cache Layer & TTL Strategy | CACHE.md         |
| Async Queue Processing     | QUEUE.md         |

---

# Post-Roast Updates

## Update 1: Seat Hold Abuse Protection

Added Redis key:

```text
holds:{userId}:count
```

Maximum active seat holds:

```text
8 seats
```

Purpose:

* Prevent seat hoarding
* Improve fairness during flash sales

---

## Update 2: Queue Circuit Breaker

Trigger:

```text
SQS Publish Failures > 60 seconds
```

Fallback:

```text
Reduced Throughput
Synchronous Payment Processing
```

Purpose:

* Continue accepting bookings during queue failures

---

## Update 3: Operational Monitoring

Added CloudWatch alarms:

```text
QueueDepth > 10,000
OldestMessageAge > 120 seconds
```

Purpose:

* Detect payment backlogs
* Trigger auto-scaling
* Alert operations team before customer impact

```
```
