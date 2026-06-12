# BookMyShow System Architecture

```text
                              Mobile App / Browser
                                        │
                                        │ HTTP/HTTPS Request
                                        ▼

┌──────────────────────────────────────────────────────────────┐
│ CloudFront CDN                                               │
│ ------------------------------------------------------------ │
│ Serves:                                                      │
│ • Static Assets (JS/CSS/Images)                              │
│ • Cached Event Pages                                         │
│                                                              │
│ Cache Hit  → Response Returned Directly                      │
│ Cache Miss → Forward Request to Origin                       │
│                                                              │
│ ← CACHE.md: CDN reduces origin traffic during peak sales     │
└──────────────────────────────────────────────────────────────┘
                                        │
                                        │ Dynamic API Requests
                                        ▼

┌──────────────────────────────────────────────────────────────┐
│ Application Load Balancer (ALB)                              │
│ ------------------------------------------------------------ │
│ • SSL Termination                                             │
│ • Health Checks: Every 10 sec                                │
│ • Rate Limit: 200 req/IP/min                                 │
│ • Route Rule: /api/* → Node.js Cluster                       │
│                                                              │
│ ← CONCURRENCY.md: protects backend during flash sales        │
└──────────────────────────────────────────────────────────────┘
                                        │
                                        │ API Traffic
                                        ▼

         ┌──────────────────────────────────────────────┐
         │      Node.js API Auto Scaling Group          │
         │----------------------------------------------│
         │ Node #1                                      │
         │ Node #2                                      │
         │ Node #3                                      │
         │ Node #N                                      │
         │                                              │
         │ Scale Out: CPU > 70% for 2 min              │
         │ Scale Range: 4 → 20 instances               │
         │                                              │
         │ Responsibilities:                            │
         │ • Handle HTTP Requests                       │
         │ • Read Redis Cache                           │
         │ • Acquire Seat Locks                         │
         │ • Write Bookings to DB                       │
         │ • Publish Payment Jobs to SQS               │
         │                                              │
         │ ← QUEUE.md: async processing reduces DB load │
         └──────────────────────────────────────────────┘

             │ READ CACHE
             │ availability:{event}:{category}
             │ event:{eventId}
             ▼

┌──────────────────────────────────────────────────────────────┐
│ Redis Cluster (3 Nodes)                                      │
│ ------------------------------------------------------------ │
│ CACHE LAYER                                                  │
│ • availability:{event}:{category}                            │
│   TTL = 30 sec                                               │
│ • event:{eventId}                                            │
│   TTL = 3600 sec                                             │
│                                                              │
│ LOCK LAYER                                                   │
│ • seat_lock:{eventId}:{seatId}                               │
│ • SETNX                                                      │
│ • Lock TTL = 30 sec                                          │
│                                                              │
│ ← CONCURRENCY.md: Redis SETNX seat locking                   │
│ ← CACHE.md: cache-aside strategy                             │
└──────────────────────────────────────────────────────────────┘

             ▲
             │ WRITE TRANSACTIONS
             │ Create Booking
             │ Seat Status Update
             │ Payment Status Update
             ▼

┌──────────────────────────────────────────────────────────────┐
│ PostgreSQL Primary                                           │
│ ------------------------------------------------------------ │
│ Writes Only                                                  │
│ • Create Booking                                             │
│ • Reserve Seat                                               │
│ • Confirm Booking                                            │
│ • Update Payment Status                                      │
│                                                              │
│ ← SCHEMA.md: source of truth                                 │
│ ← CONCURRENCY.md: optimistic locking via version column      │
└──────────────────────────────────────────────────────────────┘

                       │
                       │ Replication
                       ▼

       ┌─────────────────────┐      ┌─────────────────────┐
       │ PostgreSQL Replica1 │      │ PostgreSQL Replica2 │
       └─────────────────────┘      └─────────────────────┘

                 Used For:
                 • Event Details
                 • Seat Maps
                 • Availability Browsing
                 • User Booking History

                 ← SCHEMA.md: read/write separation
                 ← CACHE.md: reduces primary DB load

                       ▲
                       │ Publish Payment Job
                       │
                       ▼

┌──────────────────────────────────────────────────────────────┐
│ SQS Payment Queue                                            │
│ ------------------------------------------------------------ │
│ Message Format                                               │
│ {                                                            │
│   bookingId,                                                 │
│   userId,                                                    │
│   eventId,                                                   │
│   seatIds,                                                   │
│   totalAmount,                                               │
│   paymentToken,                                              │
│   idempotencyKey                                             │
│ }                                                            │
│                                                              │
│ Visibility Timeout = 30 sec                                 │
│ Max Receive Count = 3                                        │
│ Dead Letter Queue = payment-dlq                              │
│                                                              │
│ ← QUEUE.md: async payment processing                         │
│ ← QUEUE.md: prevents DB pool exhaustion                      │
└──────────────────────────────────────────────────────────────┘

                       │
                       │ Consume Message
                       ▼

┌──────────────────────────────────────────────────────────────┐
│ ECS Fargate Payment Workers (x10)                            │
│ ------------------------------------------------------------ │
│ 1. Read Message from SQS                                     │
│ 2. Call Razorpay Payment Gateway                             │
│ 3. Update PostgreSQL Primary                                 │
│ 4. Publish Booking Event to SNS                              │
│ 5. Delete Message from SQS                                   │
│                                                              │
│ ← QUEUE.md: success/failure workflow                         │
└──────────────────────────────────────────────────────────────┘

                       │
                       │ Booking Confirmed Event
                       ▼

┌──────────────────────────────────────────────────────────────┐
│ AWS SNS                                                      │
│ ------------------------------------------------------------ │
│ Fan-out Notification Service                                 │
│ Trigger: Successful Booking Confirmation                     │
│                                                              │
│ ← QUEUE.md: notification decoupling                          │
└──────────────────────────────────────────────────────────────┘

            │                                   │
            │ Email Notification                │ SMS Notification
            ▼                                   ▼

┌─────────────────────┐          ┌──────────────────────────┐
│ AWS SES             │          │ SNS SMS / Twilio         │
│ Booking Confirmation│          │ Ticket Confirmation SMS │
│                     │          │                          │
│ ← QUEUE.md          │          │ ← QUEUE.md              │
└─────────────────────┘          └──────────────────────────┘


──────────────────────────────────────────────────────────────
BOOKING FLOW
──────────────────────────────────────────────────────────────

1. User opens event page
2. CloudFront serves cached page or forwards request
3. User selects seats
4. Node.js acquires Redis SETNX seat lock
5. Booking created in PostgreSQL Primary
6. Payment job published to SQS
7. ECS worker processes payment
8. PostgreSQL updated with booking result
9. SNS publishes booking event
10. SES Email and SMS confirmation sent

──────────────────────────────────────────────────────────────
PART A DESIGN REFERENCES
──────────────────────────────────────────────────────────────

• SCHEMA.md → PostgreSQL schema, optimistic locking, replicas
• CONCURRENCY.md → Redis SETNX + PostgreSQL hybrid strategy
• CACHE.md → Cache-aside pattern, TTLs, invalidation
• QUEUE.md → Async payment processing, retries, DLQ
```
