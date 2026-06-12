# Design Updates After Panel Review

## Update 1: Per-User Seat Hold Limit

**Triggered by:**
Panel Question 3 – What prevents one user from holding 200 seats and blocking everyone else?

**What changed:**
Added a Redis-based seat hold counter.

New Redis Key:

```text
holds:{userId}:count
```

Example:

```text
holds:7b2d91a4:count
```

Maximum concurrent held seats:

```text
8 seats per user
```

Booking Flow Update:

1. User selects seats.
2. System checks current hold count.
3. If hold count >= 8:

   * Reject request.
4. Otherwise:

   * Increment counter.
   * Create seat holds.
5. Counter decreases when:

   * Booking succeeds.
   * Hold expires.
   * Payment fails.

**Why this is necessary:**
The original architecture prevented double-bookings but did not prevent seat hoarding.

A malicious user could temporarily reserve hundreds of seats and prevent legitimate customers from booking.

This update improves fairness during high-demand events.

**What it costs:**

* One additional Redis lookup.
* Slightly more application logic.
* Additional monitoring requirements.

**What it still doesn't solve:**

Users can create multiple accounts.

Future mitigation:

* OTP verification
* Device fingerprinting
* Fraud detection rules

---

## Update 2: SQS Publish Circuit Breaker

**Triggered by:**
Panel Question 1 – What happens if SQS becomes unavailable?

**What changed:**

Added a Queue Circuit Breaker.

New Behavior:

If SQS publish operations fail continuously for:

```text
60 seconds
```

the API enters:

```text
DEGRADED MODE
```

During degraded mode:

1. New booking requests are throttled.
2. Payment processing becomes synchronous.
3. Maximum booking throughput is reduced.
4. Alert is sent to operations team.

Normal operation resumes automatically after SQS recovers.

**Why this is necessary:**

The original design assumed SQS availability.

If SQS fails:

* Pending bookings cannot be processed.
* Users cannot complete purchases.

The circuit breaker allows the platform to continue operating at reduced capacity rather than failing completely.

**What it costs:**

* Additional operational complexity.
* Higher API latency during degraded mode.
* Increased database connection usage.

**What it still doesn't solve:**

Large-scale AWS regional outages.

Future mitigation:

* Cross-region queue replication.
* Multi-region deployment.

---

## Update 3: CloudWatch Queue Depth Alarm

**Triggered by:**
Panel Question 4 – What if scaling causes operational problems before users notice?

**What changed:**

Added monitoring and alerting.

New CloudWatch Alarm:

```text
QueueDepth > 10,000 messages
for 5 minutes
```

Trigger Actions:

1. PagerDuty Alert
2. Slack Alert
3. Auto-scale Payment Workers

Additional Metric:

```text
OldestMessageAge
```

Alarm Threshold:

```text
> 120 seconds
```

**Why this is necessary:**

The original architecture could process payments asynchronously but lacked proactive monitoring.

Large backlogs could develop before customers notice delays.

This update enables early detection.

**What it costs:**

* Small CloudWatch charges.
* Additional operational configuration.

**What it still doesn't solve:**

If payment providers are down, worker scaling alone cannot clear the backlog.

Provider redundancy would still be required.

---

# Summary of Changes

| Update                 | Purpose                                |
| ---------------------- | -------------------------------------- |
| Per-User Hold Limit    | Prevent seat hoarding                  |
| SQS Circuit Breaker    | Maintain service during queue failures |
| CloudWatch Queue Alarm | Detect payment backlog early           |

These updates improve fairness, resilience, and operational visibility while remaining within the overall architecture constraints established in Part A.
