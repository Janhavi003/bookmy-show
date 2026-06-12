# Async Order Processing Queue Design

## Objective

ShowTime expects up to **500,000 concurrent users** during ticket launches. Payment gateways typically take between **200ms and 2000ms** to respond.

If the API waits for payment completion before responding, database connections remain occupied for long periods, causing connection pool exhaustion and service degradation.

To solve this, ShowTime uses **Amazon SQS** and **asynchronous payment workers**.

---

# Section 1: Why Async?

## The Synchronous Problem

Traditional flow:

```text
User
  ↓
API Server
  ↓
Hold Seats
  ↓
Call Payment Gateway
  ↓
Update Booking
  ↓
Return Response
```

The API waits for the payment gateway.

---

## Connection Pool Analysis

Assumptions:

```text
max_connections = 500

80% normal requests
20% payment requests

normal query time = 20ms = 0.02s
payment processing = 800ms = 0.8s
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

Pool exhaustion:

```text
RPS × 0.176 = 500
```

Therefore:

```text
RPS = 500 / 0.176

RPS ≈ 2,841
```

---

## Why This Is Catastrophic

Expected launch traffic:

```text
≈ 25,000 RPS
```

Database exhaustion:

```text
≈ 2,841 RPS
```

This means the database pool collapses at only ~11% of expected load.

---

# Async Solution

Instead of waiting for payment:

```text
User
  ↓
API
  ↓
Hold Seats
  ↓
Create Pending Booking
  ↓
Publish SQS Message
  ↓
Return 202 Accepted
```

API latency:

```text
~50ms
```

Database connection duration:

```text
~10ms
```

Payment processing occurs separately.

---

# Section 2: Queue Message Format

## SQS Message Body

```json
{
  "bookingId": "a2f91e6b-1111-2222-3333-abcdef123456",
  "userId": "b7c22a3d-4444-5555-6666-fedcba987654",
  "eventId": 1001,
  "seatIds": [501, 502, 503],
  "totalAmount": 15000.00,
  "paymentToken": "tok_xxxxxxxxx",
  "idempotencyKey": "booking_a2f91e6b"
}
```

---

## Field Explanation

### bookingId

```text
Unique booking identifier.
```

Used to update booking status.

---

### userId

```text
Customer identifier.
```

Required for notifications and auditing.

---

### eventId

```text
Event associated with booking.
```

Allows worker to validate event details.

---

### seatIds

```text
Reserved seat identifiers.
```

Worker updates seat status after payment completion.

---

### totalAmount

```text
Amount charged to customer.
```

Prevents recalculation errors.

---

### paymentToken

```text
Secure payment authorization token.
```

Used when calling Razorpay or PayU.

---

### idempotencyKey

```text
Unique processing identifier.
```

Prevents duplicate payment execution.

Even if the same message is processed twice, payment remains safe.

---

# Section 3: Worker Logic

## Worker Success Flow

### Step 1

Read message from SQS.

```text
ReceiveMessage()
```

---

### Step 2

Verify idempotency key.

```text
Already processed?
```

If yes:

```text
Delete message
Exit
```

---

### Step 3

Call payment gateway.

```text
Razorpay
PayU
Stripe
```

---

### Step 4

Payment succeeds.

Update booking:

```sql
UPDATE bookings
SET status = 'confirmed'
WHERE id = :bookingId;
```

---

### Step 5

Mark seats booked.

```sql
UPDATE seats
SET status = 'booked'
WHERE id IN (...);
```

---

### Step 6

Send notifications.

```text
SMS
Email
Push Notification
```

---

### Step 7

Delete SQS message.

```text
DeleteMessage()
```

Processing complete.

---

# Worker Failure Flow

### Step 1

Receive SQS message.

---

### Step 2

Call payment gateway.

---

### Step 3

Payment fails.

Update booking:

```sql
UPDATE bookings
SET status = 'failed'
WHERE id = :bookingId;
```

---

### Step 4

Release seats.

```sql
UPDATE seats
SET status = 'available',
    held_until = NULL,
    held_by = NULL
WHERE id IN (...);
```

---

### Step 5

Send failure notification.

```text
Payment failed
Please retry
```

---

### Step 6

Delete message.

Processing complete.

---

# Dead Letter Queue (DLQ)

If processing repeatedly fails:

```text
Message
  ↓
Retry
  ↓
Retry
  ↓
Retry
  ↓
DLQ
```

Messages in the DLQ require manual investigation.

Possible causes:

* Payment provider outage
* Corrupted payload
* Internal service bug

---

# Section 4: Edge Cases

## Edge Case 1

### API Server Crashes After SQS Publish But Before Response

Flow:

```text
Booking Created
Message Published
Server Crashes
Response Lost
```

Customer sees:

```text
Request timed out
```

However:

```text
Booking exists
Message exists
Worker continues processing
```

Result:

* Payment may still complete.
* Seats remain reserved.
* User can retrieve booking later through booking history.

No data loss occurs.

---

## Edge Case 2

### Payment Gateway Timeout

Gateway returns:

```text
Unknown Status
```

Neither success nor failure.

Worker actions:

### Attempt 1

Retry payment status query.

```text
Check provider transaction status
```

---

### Attempt 2

If still unknown:

Leave message unacknowledged.

```text
Visibility timeout expires
```

---

### Attempt 3

Message becomes visible again.

Another worker retries.

---

### After Maximum Retries

Move message to:

```text
Dead Letter Queue
```

Manual investigation required.

---

# Section 5: SQS Configuration

## Visibility Timeout

Chosen value:

```text
30 seconds
```

---

### Why 30 Seconds?

Expected payment duration:

```text
200ms - 2000ms
```

Rule:

```text
Visibility Timeout
≈ 2 × Maximum Processing Time
```

30 seconds provides:

* Retry safety
* Worker crash recovery
* Protection against duplicate processing

---

## Max Receive Count

Chosen value:

```text
3 retries
```

Configuration:

```text
maxReceiveCount = 3
```

---

### Why 3?

One failure may be transient.

Examples:

```text
Network issue
Payment provider slowdown
Temporary database issue
```

After three failures:

* Problem is likely persistent.
* Automated retries are unlikely to help.

Message should be moved to the DLQ.

---

# Complete Async Flow

```text
User
  ↓
POST /bookings
  ↓
API Server
  ↓
Hold Seats
  ↓
Create Pending Booking
  ↓
Publish Message to SQS
  ↓
Return 202 Accepted

-------------------------

SQS Queue
  ↓
Payment Worker
  ↓
Payment Gateway

SUCCESS:
  ↓
Confirm Booking
  ↓
Book Seats
  ↓
Send Notifications
  ↓
Delete Message

FAILURE:
  ↓
Mark Failed
  ↓
Release Seats
  ↓
Send Notification
  ↓
Delete Message

REPEATED FAILURE:
  ↓
Dead Letter Queue
```

---

# Final Design Decision

ShowTime uses an asynchronous payment architecture based on Amazon SQS.

Benefits:

* Keeps API latency below 500ms
* Prevents database connection exhaustion
* Supports 500,000 concurrent users
* Allows independent worker scaling
* Provides fault tolerance through retries and DLQs
* Fits within the $2,000/month AWS budget

This design separates customer-facing responsiveness from payment processing complexity while preserving correctness and reliability.
