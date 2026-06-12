# ShowTime Database Schema Design

## PostgreSQL Schema

```sql
-- Enable UUID generation
CREATE EXTENSION IF NOT EXISTS "pgcrypto";

--------------------------------------------------
-- VENUES
--------------------------------------------------

CREATE TABLE venues (
    id BIGSERIAL PRIMARY KEY,
    name VARCHAR(255) NOT NULL,
    city VARCHAR(100) NOT NULL,
    capacity INTEGER NOT NULL CHECK (capacity > 0),
    created_at TIMESTAMP NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_venues_city
ON venues(city);

--------------------------------------------------
-- EVENTS
--------------------------------------------------

CREATE TABLE events (
    id BIGSERIAL PRIMARY KEY,
    venue_id BIGINT NOT NULL,
    name VARCHAR(255) NOT NULL,
    start_time TIMESTAMP NOT NULL,
    status VARCHAR(20) NOT NULL
        CHECK (status IN ('upcoming','on_sale','sold_out','cancelled')),
    total_seat_count INTEGER NOT NULL CHECK (total_seat_count > 0),
    created_at TIMESTAMP NOT NULL DEFAULT NOW(),

    CONSTRAINT fk_events_venue
        FOREIGN KEY (venue_id)
        REFERENCES venues(id)
        ON DELETE RESTRICT
);

CREATE INDEX idx_events_status
ON events(status);

CREATE INDEX idx_events_start_time
ON events(start_time);

--------------------------------------------------
-- USERS
--------------------------------------------------

CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email VARCHAR(255) NOT NULL UNIQUE,
    phone VARCHAR(20) NOT NULL UNIQUE,
    name VARCHAR(255) NOT NULL,
    created_at TIMESTAMP NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_users_email
ON users(email);

--------------------------------------------------
-- SEATS
--------------------------------------------------

CREATE TABLE seats (
    id BIGSERIAL PRIMARY KEY,

    event_id BIGINT NOT NULL,

    section VARCHAR(50) NOT NULL,
    row_name VARCHAR(20) NOT NULL,
    seat_number VARCHAR(20) NOT NULL,

    category VARCHAR(20) NOT NULL
        CHECK (category IN ('VIP','Premium','General')),

    price NUMERIC(10,2) NOT NULL
        CHECK (price > 0),

    status VARCHAR(20) NOT NULL
        CHECK (status IN ('available','held','booked')),

    held_until TIMESTAMP NULL,

    held_by UUID NULL,

    version INTEGER NOT NULL DEFAULT 0
        CHECK (version >= 0),

    created_at TIMESTAMP NOT NULL DEFAULT NOW(),

    CONSTRAINT fk_seats_event
        FOREIGN KEY (event_id)
        REFERENCES events(id)
        ON DELETE CASCADE,

    CONSTRAINT fk_seats_user
        FOREIGN KEY (held_by)
        REFERENCES users(id)
        ON DELETE SET NULL,

    CONSTRAINT uq_event_seat
        UNIQUE(event_id, section, row_name, seat_number)
);

-- Critical index required by assignment
CREATE INDEX idx_seats_event_status
ON seats(event_id, status);

CREATE INDEX idx_seats_category
ON seats(event_id, category);

--------------------------------------------------
-- BOOKINGS
--------------------------------------------------

CREATE TABLE bookings (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),

    user_id UUID NOT NULL,
    event_id BIGINT NOT NULL,

    status VARCHAR(20) NOT NULL
        CHECK (status IN ('pending','confirmed','failed','refunded')),

    total_amount NUMERIC(12,2) NOT NULL
        CHECK (total_amount > 0),

    payment_reference VARCHAR(255),

    created_at TIMESTAMP NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMP NOT NULL DEFAULT NOW(),

    CONSTRAINT fk_bookings_user
        FOREIGN KEY (user_id)
        REFERENCES users(id)
        ON DELETE RESTRICT,

    CONSTRAINT fk_bookings_event
        FOREIGN KEY (event_id)
        REFERENCES events(id)
        ON DELETE RESTRICT
);

-- Critical index required by assignment
CREATE INDEX idx_bookings_user
ON bookings(user_id, created_at DESC);

-- Partial index for unresolved bookings
CREATE INDEX idx_bookings_unresolved
ON bookings(status, created_at)
WHERE status IN ('pending','failed');

--------------------------------------------------
-- BOOKING_SEATS
--------------------------------------------------

CREATE TABLE booking_seats (
    id BIGSERIAL PRIMARY KEY,

    booking_id UUID NOT NULL,
    seat_id BIGINT NOT NULL,

    quantity INTEGER NOT NULL DEFAULT 1
        CHECK (quantity > 0),

    created_at TIMESTAMP NOT NULL DEFAULT NOW(),

    CONSTRAINT fk_booking_seats_booking
        FOREIGN KEY (booking_id)
        REFERENCES bookings(id)
        ON DELETE CASCADE,

    CONSTRAINT fk_booking_seats_seat
        FOREIGN KEY (seat_id)
        REFERENCES seats(id)
        ON DELETE RESTRICT,

    CONSTRAINT uq_booking_seat
        UNIQUE(booking_id, seat_id)
);

CREATE INDEX idx_booking_seats_booking
ON booking_seats(booking_id);

CREATE INDEX idx_booking_seats_seat
ON booking_seats(seat_id);
```

---

# Design Commentary

## Why UUID for booking.id instead of SERIAL?

UUIDs provide globally unique identifiers that are difficult to predict. If SERIAL integers were used, an attacker could enumerate booking records simply by incrementing IDs. UUIDs improve security, support distributed systems, and allow IDs to be generated before database insertion, which is useful for idempotent booking requests.

---

## Why does the seats table have a version column?

The version column supports optimistic locking.

Example:

1. User A reads seat version = 0.
2. User B reads seat version = 0.
3. User A updates seat and increments version to 1.
4. User B attempts update using version = 0.
5. Update fails because the version no longer matches.

This prevents race conditions without requiring long-running database locks.

---

## Why use held_until instead of application-level seat holds?

Application memory is not reliable because:

* Servers can restart.
* Multiple API servers may exist.
* Held seats must automatically expire.

The held_until timestamp keeps hold information in the database where all servers can access it consistently. A background cleanup job can periodically release seats whose hold period has expired.

Example:

```sql
UPDATE seats
SET status = 'available',
    held_until = NULL,
    held_by = NULL
WHERE status = 'held'
  AND held_until < NOW();
```

This guarantees abandoned carts do not block seat inventory forever.

---

## Why use a Partial Index on bookings.status?

Most bookings eventually become either:

* confirmed
* refunded

These historical records are rarely queried by operational workers.

The payment worker primarily searches for:

* pending bookings
* failed bookings

A partial index stores only these unresolved records:

```sql
WHERE status IN ('pending','failed')
```

Benefits:

* Smaller index size
* Faster lookups
* Reduced memory usage
* Improved worker performance during large ticket sales

This is especially important during high-volume events where millions of historical bookings may exist but only a small percentage require active processing.
