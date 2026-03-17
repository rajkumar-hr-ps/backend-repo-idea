I now have a comprehensive understanding of the ShowPass architecture, test patterns, task structure, and conventions. Let me produce the full architecture document for MeetSync.

---

# MeetSync -- Appointment Scheduling API: Complete Architecture Document

---

## 1. Application Concept

### What is MeetSync?

MeetSync is a backend-only REST API for appointment scheduling, designed in the same vein as products like Calendly, Cal.com, and Acuity Scheduling. It allows **hosts** (professionals, consultants, sales teams, tutors, or anyone who offers their time) to define when they are available, create shareable booking links with specific constraints, and allow **guests** (external or internal users) to book appointments in those open slots. MeetSync then manages the full lifecycle of each appointment from creation through completion, cancellation, or rescheduling, and notifies external systems about appointment events via a configurable webhook system.

The product vision is intentionally narrow: MeetSync handles the **scheduling backend only**. There is no calendar UI, no email sending, no video conferencing integration. This keeps the codebase at the right size for a 2-3 hour assessment (12-15 source files) while still requiring candidates to work with the kind of complex, interlocking business logic found in real scheduling systems.

### Who are the users?

There are two primary personas:

1. **Hosts** -- Authenticated users who register on the platform, define their availability windows, create booking links with custom settings (meeting duration, buffer time between meetings, max bookings per day), and manage their appointments. A host might be a doctor, a hiring manager, a tutor, or a consultant.

2. **Guests** -- People who book time with a host. Guests do not need an account. They provide their name, email, and optionally a note when booking. This is a critical design choice: real scheduling tools (Calendly, Cal.com) allow anyone with a link to book without signing up.

There is also an **admin** role for platform-level management, consistent with the ShowPass pattern.

### How does it compare to Calendly/Cal.com?

MeetSync is a simplified version of these platforms, focusing on the core scheduling API patterns:

- **Availability rules** map to Calendly's "Weekly Hours" -- recurring day-of-week windows (e.g., Monday-Friday 9:00-17:00 in America/New_York).
- **Availability overrides** map to Calendly's "Date overrides" -- specific dates that are blocked or have custom hours (e.g., "Dec 25: unavailable" or "Dec 24: 9:00-12:00 only").
- **Booking links** map to Calendly's "Event Types" -- shareable URLs with a specific meeting duration, buffer time, and daily limit.
- **Appointments** map to Calendly's "Scheduled Events."
- **Webhook subscriptions** map to Calendly's webhook API for integrating with CRMs, Slack, etc.

### What makes this a good assessment domain?

Appointment scheduling is universally understood. Every candidate has booked a meeting, a doctor's appointment, or a job interview. This eliminates domain confusion and lets candidates focus on the engineering challenges.

The domain naturally produces complex engineering problems that mirror real production systems:

- **Conflict detection**: Overlapping time slot queries, buffer time math, timezone arithmetic.
- **State machines**: Appointments move through a well-defined lifecycle (pending, confirmed, completed, cancelled, no_show).
- **Idempotency**: Two guests clicking "Book" at the same instant must not double-book the same slot.
- **Caching**: Availability calculations are expensive (recurring rules + overrides + existing appointments) and benefit from Redis caching, but must be invalidated when bookings change.
- **Webhooks**: External systems subscribe to appointment events; delivery must be reliable with retries and backoff.
- **Pagination**: A host with hundreds of past appointments needs filtered, paginated listing.
- **Timezone handling**: A host in New York and a guest in London must agree on the actual UTC time of the meeting.
- **Rate limiting**: Public booking endpoints must be protected from abuse.
- **Soft-delete**: Cancelled appointments must not leak into availability calculations or appointment listings.
- **Input validation**: Dates, times, timezones, durations, emails -- all must be validated carefully.

---

## 2. Data Models

### 2.1 User

Represents an authenticated platform user (host, admin, or basic user).

**Fields:**

| Field | Type | Required | Default | Validation | Notes |
|---|---|---|---|---|---|
| `_id` | ObjectId | auto | auto | | Mongoose default |
| `name` | String | yes | | `maxlength: 100`, `trim: true` | Display name |
| `email` | String | yes | | `unique: true`, `lowercase: true`, `trim: true` | Login email |
| `password` | String | yes | | `minlength: 8` | bcrypt hashed |
| `role` | String | yes | `'host'` | `enum: ['host', 'admin']` | Platform role |
| `timezone` | String | yes | `'UTC'` | Must be valid IANA timezone | Host's default timezone |
| `created_at` | Date | auto | | | Mongoose timestamps |
| `updated_at` | Date | auto | | | Mongoose timestamps |
| `deleted_at` | Date | no | `null` | | Soft delete (plugin) |

**Indexes:**
- `{ email: 1 }` -- unique, for login lookup
- `{ role: 1 }` -- for admin queries

**Pre-save hook:**
- Hash password with bcrypt (cost factor 12) if modified.

**Instance methods:**
- `comparePassword(candidate)` -- bcrypt compare.

**Relationships:**
- One-to-many with AvailabilityRule (host's rules)
- One-to-many with AvailabilityOverride (host's overrides)
- One-to-many with BookingLink (host's links)
- One-to-many with Appointment (as host)
- One-to-many with WebhookSubscription (host's subscriptions)

**Schema sketch:**
```javascript
const userSchema = new mongoose.Schema(
  {
    name: { type: String, required: true, maxlength: 100, trim: true },
    email: { type: String, required: true, unique: true, lowercase: true, trim: true },
    password: { type: String, required: true, minlength: 8 },
    role: { type: String, required: true, enum: USER_ROLES, default: 'host' },
    timezone: { type: String, required: true, default: 'UTC' },
  },
  { timestamps: { createdAt: 'created_at', updatedAt: 'updated_at' } }
);
userSchema.plugin(softDeletePlugin);
```

### 2.2 AvailabilityRule

Defines recurring weekly availability for a host (e.g., "Every Monday 09:00-17:00").

**Fields:**

| Field | Type | Required | Default | Validation | Notes |
|---|---|---|---|---|---|
| `_id` | ObjectId | auto | auto | | |
| `host_id` | ObjectId (ref: User) | yes | | | The host this rule belongs to |
| `day_of_week` | Number | yes | | `min: 0, max: 6` (0=Sunday, 6=Saturday) | Day this rule applies to |
| `start_time` | String | yes | | Format: `"HH:mm"`, 24-hour | Start of availability window |
| `end_time` | String | yes | | Format: `"HH:mm"`, 24-hour | End of availability window |
| `timezone` | String | yes | | Valid IANA timezone string | Timezone for the times |
| `created_at` | Date | auto | | | |
| `updated_at` | Date | auto | | | |
| `deleted_at` | Date | no | `null` | | Soft delete |

**Indexes:**
- `{ host_id: 1, day_of_week: 1 }` -- for looking up a host's rules by day
- `{ host_id: 1 }` -- for fetching all rules for a host

**Pre-save hook:**
- Validate `end_time > start_time` (e.g., "09:00" < "17:00").
- Validate time format is `HH:mm`.

**Relationships:**
- Many-to-one with User (host)

**Schema sketch:**
```javascript
const availabilityRuleSchema = new mongoose.Schema(
  {
    host_id: { type: mongoose.Schema.Types.ObjectId, ref: 'User', required: true },
    day_of_week: { type: Number, required: true, min: 0, max: 6 },
    start_time: { type: String, required: true },
    end_time: { type: String, required: true },
    timezone: { type: String, required: true },
  },
  { timestamps: { createdAt: 'created_at', updatedAt: 'updated_at' } }
);
availabilityRuleSchema.plugin(softDeletePlugin);
```

### 2.3 AvailabilityOverride

Specific date exceptions to recurring rules (e.g., "Dec 25 -- unavailable" or "Dec 24 -- available 09:00-12:00 only").

**Fields:**

| Field | Type | Required | Default | Validation | Notes |
|---|---|---|---|---|---|
| `_id` | ObjectId | auto | auto | | |
| `host_id` | ObjectId (ref: User) | yes | | | |
| `date` | String | yes | | Format: `"YYYY-MM-DD"` | The specific date |
| `is_available` | Boolean | yes | | | `false` = blocked all day, `true` = custom hours |
| `start_time` | String | no | `null` | Format: `"HH:mm"`, required if `is_available: true` | Custom start |
| `end_time` | String | no | `null` | Format: `"HH:mm"`, required if `is_available: true` | Custom end |
| `timezone` | String | yes | | Valid IANA timezone | |
| `reason` | String | no | `''` | `maxlength: 500` | Optional note (e.g., "Holiday") |
| `created_at` | Date | auto | | | |
| `updated_at` | Date | auto | | | |
| `deleted_at` | Date | no | `null` | | |

**Indexes:**
- `{ host_id: 1, date: 1 }` -- unique compound, one override per date per host

**Pre-save hook:**
- If `is_available: true`, require `start_time` and `end_time` and validate `end_time > start_time`.
- If `is_available: false`, `start_time` and `end_time` must be null.
- Validate date format is `YYYY-MM-DD`.

**Relationships:**
- Many-to-one with User (host)

**Schema sketch:**
```javascript
const availabilityOverrideSchema = new mongoose.Schema(
  {
    host_id: { type: mongoose.Schema.Types.ObjectId, ref: 'User', required: true },
    date: { type: String, required: true },
    is_available: { type: Boolean, required: true },
    start_time: { type: String, default: null },
    end_time: { type: String, default: null },
    timezone: { type: String, required: true },
    reason: { type: String, maxlength: 500, default: '' },
  },
  { timestamps: { createdAt: 'created_at', updatedAt: 'updated_at' } }
);
availabilityOverrideSchema.index({ host_id: 1, date: 1 }, { unique: true });
availabilityOverrideSchema.plugin(softDeletePlugin);
```

### 2.4 BookingLink

A shareable scheduling link with custom settings. Like a Calendly Event Type.

**Fields:**

| Field | Type | Required | Default | Validation | Notes |
|---|---|---|---|---|---|
| `_id` | ObjectId | auto | auto | | |
| `host_id` | ObjectId (ref: User) | yes | | | |
| `slug` | String | yes | | `unique: true`, `lowercase: true`, `trim: true`, `maxlength: 100` | URL-safe identifier (e.g., "30-min-consultation") |
| `title` | String | yes | | `maxlength: 200`, `trim: true` | Display name (e.g., "30 Minute Consultation") |
| `description` | String | no | `''` | `maxlength: 1000` | Optional description |
| `duration_minutes` | Number | yes | | `enum: [15, 30, 45, 60, 90, 120]` | Meeting duration |
| `buffer_before_minutes` | Number | no | `0` | `min: 0, max: 60` | Buffer time before each meeting |
| `buffer_after_minutes` | Number | no | `0` | `min: 0, max: 60` | Buffer time after each meeting |
| `max_bookings_per_day` | Number | no | `null` | `min: 1` | Daily booking limit (null = unlimited) |
| `min_notice_hours` | Number | no | `24` | `min: 0` | Minimum hours in advance to book |
| `max_advance_days` | Number | no | `60` | `min: 1, max: 365` | How far in the future guests can book |
| `status` | String | yes | `'active'` | `enum: ['active', 'paused', 'archived']` | Link status |
| `created_at` | Date | auto | | | |
| `updated_at` | Date | auto | | | |
| `deleted_at` | Date | no | `null` | | |

**Indexes:**
- `{ slug: 1 }` -- unique, for public URL lookup
- `{ host_id: 1, status: 1 }` -- for host's active links
- `{ host_id: 1 }` -- for fetching all links for a host

**Relationships:**
- Many-to-one with User (host)
- One-to-many with Appointment

**Schema sketch:**
```javascript
export const BookingLinkStatus = {
  ACTIVE: 'active',
  PAUSED: 'paused',
  ARCHIVED: 'archived',
};

export const VALID_LINK_TRANSITIONS = {
  [BookingLinkStatus.ACTIVE]: [BookingLinkStatus.PAUSED, BookingLinkStatus.ARCHIVED],
  [BookingLinkStatus.PAUSED]: [BookingLinkStatus.ACTIVE, BookingLinkStatus.ARCHIVED],
  [BookingLinkStatus.ARCHIVED]: [],
};

const bookingLinkSchema = new mongoose.Schema(
  {
    host_id: { type: mongoose.Schema.Types.ObjectId, ref: 'User', required: true },
    slug: { type: String, required: true, unique: true, lowercase: true, trim: true, maxlength: 100 },
    title: { type: String, required: true, maxlength: 200, trim: true },
    description: { type: String, maxlength: 1000, default: '' },
    duration_minutes: { type: Number, required: true, enum: [15, 30, 45, 60, 90, 120] },
    buffer_before_minutes: { type: Number, default: 0, min: 0, max: 60 },
    buffer_after_minutes: { type: Number, default: 0, min: 0, max: 60 },
    max_bookings_per_day: { type: Number, default: null, min: 1 },
    min_notice_hours: { type: Number, default: 24, min: 0 },
    max_advance_days: { type: Number, default: 60, min: 1, max: 365 },
    status: { type: String, required: true, enum: BOOKING_LINK_STATUSES, default: 'active' },
  },
  { timestamps: { createdAt: 'created_at', updatedAt: 'updated_at' } }
);
bookingLinkSchema.plugin(softDeletePlugin);
```

### 2.5 Appointment

A booked meeting between a host and a guest.

**Fields:**

| Field | Type | Required | Default | Validation | Notes |
|---|---|---|---|---|---|
| `_id` | ObjectId | auto | auto | | |
| `host_id` | ObjectId (ref: User) | yes | | | |
| `booking_link_id` | ObjectId (ref: BookingLink) | yes | | | Which link was used |
| `guest_name` | String | yes | | `maxlength: 100`, `trim: true` | Guest's name |
| `guest_email` | String | yes | | `lowercase: true`, `trim: true` | Guest's email |
| `guest_notes` | String | no | `''` | `maxlength: 1000` | Optional message from guest |
| `start_time` | Date | yes | | | Appointment start (UTC) |
| `end_time` | Date | yes | | | Appointment end (UTC) |
| `duration_minutes` | Number | yes | | | Copied from booking link at creation time |
| `timezone` | String | yes | | Valid IANA timezone | Guest's timezone |
| `status` | String | yes | `'pending'` | `enum: ['pending', 'confirmed', 'cancelled', 'completed', 'no_show']` | |
| `cancellation_reason` | String | no | `null` | `maxlength: 500` | Reason for cancellation |
| `cancelled_by` | String | no | `null` | `enum: ['host', 'guest', null]` | Who cancelled |
| `idempotency_key` | String | yes | | `unique: true` | Prevent double-booking |
| `created_at` | Date | auto | | | |
| `updated_at` | Date | auto | | | |
| `deleted_at` | Date | no | `null` | | |

**Indexes:**
- `{ host_id: 1, start_time: 1 }` -- for conflict detection and availability queries
- `{ host_id: 1, status: 1 }` -- for filtered listing
- `{ booking_link_id: 1, start_time: 1 }` -- for daily booking count
- `{ guest_email: 1 }` -- for guest lookup
- `{ idempotency_key: 1 }` -- unique, for idempotency

**Pre-save hook:**
- Validate `end_time > start_time`.
- Validate `end_time - start_time === duration_minutes`.
- State machine validation (see Status Lifecycles section).

**Status enum and transitions:**
```javascript
export const AppointmentStatus = {
  PENDING: 'pending',
  CONFIRMED: 'confirmed',
  CANCELLED: 'cancelled',
  COMPLETED: 'completed',
  NO_SHOW: 'no_show',
};

export const VALID_APPOINTMENT_TRANSITIONS = {
  pending: ['confirmed', 'cancelled'],
  confirmed: ['completed', 'cancelled', 'no_show'],
  cancelled: [],
  completed: [],
  no_show: [],
};
```

**Relationships:**
- Many-to-one with User (host)
- Many-to-one with BookingLink

**Schema sketch:**
```javascript
const appointmentSchema = new mongoose.Schema(
  {
    host_id: { type: mongoose.Schema.Types.ObjectId, ref: 'User', required: true },
    booking_link_id: { type: mongoose.Schema.Types.ObjectId, ref: 'BookingLink', required: true },
    guest_name: { type: String, required: true, maxlength: 100, trim: true },
    guest_email: { type: String, required: true, lowercase: true, trim: true },
    guest_notes: { type: String, maxlength: 1000, default: '' },
    start_time: { type: Date, required: true },
    end_time: { type: Date, required: true },
    duration_minutes: { type: Number, required: true },
    timezone: { type: String, required: true },
    status: { type: String, required: true, enum: APPOINTMENT_STATUSES, default: 'pending' },
    cancellation_reason: { type: String, maxlength: 500, default: null },
    cancelled_by: { type: String, enum: ['host', 'guest', null], default: null },
    idempotency_key: { type: String, required: true, unique: true },
  },
  { timestamps: { createdAt: 'created_at', updatedAt: 'updated_at' } }
);
appointmentSchema.plugin(softDeletePlugin);
```

### 2.6 WebhookSubscription

A registered webhook endpoint for receiving event notifications.

**Fields:**

| Field | Type | Required | Default | Validation | Notes |
|---|---|---|---|---|---|
| `_id` | ObjectId | auto | auto | | |
| `host_id` | ObjectId (ref: User) | yes | | | |
| `url` | String | yes | | Must start with `https://` | Target URL |
| `events` | [String] | yes | | Each must be valid event type | Which events to receive |
| `secret` | String | yes | | auto-generated | Signing secret for HMAC |
| `status` | String | yes | `'active'` | `enum: ['active', 'inactive']` | |
| `created_at` | Date | auto | | | |
| `updated_at` | Date | auto | | | |
| `deleted_at` | Date | no | `null` | | |

**Valid event types:**
```javascript
export const WEBHOOK_EVENT_TYPES = [
  'appointment.created',
  'appointment.confirmed',
  'appointment.cancelled',
  'appointment.completed',
  'appointment.no_show',
  'appointment.rescheduled',
];
```

**Indexes:**
- `{ host_id: 1, status: 1 }` -- for finding active subscriptions
- `{ host_id: 1 }` -- general lookup

**Relationships:**
- Many-to-one with User (host)
- One-to-many with WebhookDelivery

### 2.7 WebhookDelivery

Log of each webhook delivery attempt.

**Fields:**

| Field | Type | Required | Default | Validation | Notes |
|---|---|---|---|---|---|
| `_id` | ObjectId | auto | auto | | |
| `subscription_id` | ObjectId (ref: WebhookSubscription) | yes | | | |
| `event_type` | String | yes | | Valid event type | e.g., "appointment.created" |
| `payload` | Object | yes | | | The JSON payload sent |
| `delivery_id` | String | yes | | `unique: true` | Unique delivery ID for idempotency |
| `status` | String | yes | `'pending'` | `enum: ['pending', 'delivered', 'failed', 'exhausted']` | |
| `http_status` | Number | no | `null` | | Response status code |
| `response_body` | String | no | `null` | `maxlength: 5000` | Truncated response |
| `attempt_count` | Number | yes | `0` | | Number of delivery attempts |
| `max_attempts` | Number | yes | `5` | | Maximum retries |
| `next_retry_at` | Date | no | `null` | | When to retry next |
| `last_attempted_at` | Date | no | `null` | | Timestamp of last attempt |
| `created_at` | Date | auto | | | |
| `updated_at` | Date | auto | | | |

**Indexes:**
- `{ subscription_id: 1 }` -- for listing deliveries by subscription
- `{ status: 1, next_retry_at: 1 }` -- for retry processor
- `{ delivery_id: 1 }` -- unique

**Status transitions:**
```javascript
export const DeliveryStatus = {
  PENDING: 'pending',
  DELIVERED: 'delivered',
  FAILED: 'failed',
  EXHAUSTED: 'exhausted',
};
```

---

## 3. API Endpoints

All endpoints are prefixed with `/api/v1`. Authentication uses Bearer JWT tokens. The standard error response format is `{ "error": "message" }`.

### 3.1 Auth Endpoints

| # | Method | Path | Auth | Description |
|---|---|---|---|---|
| 1 | POST | `/auth/register` | No | Register a new user |
| 2 | POST | `/auth/login` | No | Login, receive JWT |
| 3 | GET | `/auth/me` | Yes | Get current user profile |

**POST /auth/register**
- Body: `{ name, email, password, timezone? }`
- Response 201: `{ user: { _id, name, email, role, timezone, created_at }, token }`
- Errors: 400 (missing fields, invalid email, weak password), 400 (email already exists)

**POST /auth/login**
- Body: `{ email, password }`
- Response 200: `{ user: { _id, name, email, role, timezone }, token }`
- Errors: 401 (invalid credentials)

**GET /auth/me**
- Response 200: `{ user: { _id, name, email, role, timezone, created_at, updated_at } }`
- Errors: 401 (not authenticated)

### 3.2 Availability Rule Endpoints

| # | Method | Path | Auth | Description |
|---|---|---|---|---|
| 4 | GET | `/availability/rules` | Yes | List all rules for the authenticated host |
| 5 | POST | `/availability/rules` | Yes | Create a new availability rule |
| 6 | PUT | `/availability/rules/:id` | Yes | Update an availability rule |
| 7 | DELETE | `/availability/rules/:id` | Yes | Soft-delete an availability rule |

**POST /availability/rules**
- Body: `{ day_of_week, start_time, end_time, timezone }`
- Response 201: `{ rule: { ... } }`
- Business logic: Validates time format, end > start, valid timezone, valid day_of_week. Invalidates availability cache for this host.
- Errors: 400 (invalid day, bad time format, end before start, invalid timezone)

**GET /availability/rules**
- Response 200: `{ rules: [ ... ] }`
- Only returns non-deleted rules for the authenticated host.

**PUT /availability/rules/:id**
- Body: `{ day_of_week?, start_time?, end_time?, timezone? }`
- Response 200: `{ rule: { ... } }`
- Must own the rule. Invalidates availability cache.
- Errors: 404 (not found / not owner), 400 (validation)

**DELETE /availability/rules/:id**
- Response 200: `{ message: "rule deleted" }`
- Soft-delete. Invalidates availability cache.
- Errors: 404 (not found / not owner)

### 3.3 Availability Override Endpoints

| # | Method | Path | Auth | Description |
|---|---|---|---|---|
| 8 | GET | `/availability/overrides` | Yes | List all overrides for the authenticated host |
| 9 | POST | `/availability/overrides` | Yes | Create an override |
| 10 | PUT | `/availability/overrides/:id` | Yes | Update an override |
| 11 | DELETE | `/availability/overrides/:id` | Yes | Soft-delete an override |

**POST /availability/overrides**
- Body: `{ date, is_available, start_time?, end_time?, timezone, reason? }`
- Response 201: `{ override: { ... } }`
- Business logic: If `is_available: false`, start_time/end_time must be null. If `is_available: true`, both are required and end > start. Date must be in the future. One override per date per host (upsert or reject). Invalidates availability cache.
- Errors: 400 (invalid date format, past date, missing times when available, end before start), 409 (duplicate date)

### 3.4 Booking Link Endpoints

| # | Method | Path | Auth | Description |
|---|---|---|---|---|
| 12 | GET | `/booking-links` | Yes | List all booking links for the authenticated host |
| 13 | POST | `/booking-links` | Yes | Create a new booking link |
| 14 | GET | `/booking-links/:id` | Yes | Get a specific booking link by ID |
| 15 | PUT | `/booking-links/:id` | Yes | Update a booking link |
| 16 | PATCH | `/booking-links/:id/status` | Yes | Change booking link status |

**POST /booking-links**
- Body: `{ slug, title, description?, duration_minutes, buffer_before_minutes?, buffer_after_minutes?, max_bookings_per_day?, min_notice_hours?, max_advance_days? }`
- Response 201: `{ booking_link: { ... } }`
- Business logic: Validate slug uniqueness, duration enum, buffer ranges, numeric bounds.
- Errors: 400 (validation), 400 (slug already exists)

**PATCH /booking-links/:id/status**
- Body: `{ status }`
- Response 200: `{ booking_link: { ... } }`
- Business logic: State machine validation (see BookingLink status lifecycle). Only owner can change status.
- Errors: 400 (invalid transition), 404 (not found / not owner)

### 3.5 Appointment Endpoints (Public + Authenticated)

| # | Method | Path | Auth | Description |
|---|---|---|---|---|
| 17 | GET | `/booking-links/:slug/slots` | No | Get available time slots for a booking link (public) |
| 18 | POST | `/booking-links/:slug/book` | No | Book an appointment (public) |
| 19 | GET | `/appointments` | Yes | List host's appointments (paginated, filterable) |
| 20 | GET | `/appointments/:id` | Yes | Get a single appointment |
| 21 | PATCH | `/appointments/:id/status` | Yes | Change appointment status (confirm, complete, no_show) |
| 22 | PATCH | `/appointments/:id/cancel` | Yes* | Cancel an appointment (host auth or guest token) |
| 23 | PATCH | `/appointments/:id/reschedule` | Yes | Reschedule an appointment |

**GET /booking-links/:slug/slots** (PUBLIC, rate-limited)
- Query: `{ date: "YYYY-MM-DD", timezone?: "America/New_York" }`
- Response 200:
```json
{
  "booking_link": { "title": "...", "duration_minutes": 30, ... },
  "date": "2025-01-15",
  "timezone": "America/New_York",
  "slots": [
    { "start_time": "2025-01-15T09:00:00-05:00", "end_time": "2025-01-15T09:30:00-05:00" },
    { "start_time": "2025-01-15T09:30:00-05:00", "end_time": "2025-01-15T10:00:00-05:00" }
  ]
}
```
- Business logic: Calculate availability for the given date using:
  1. Get the day_of_week for the requested date
  2. Look up AvailabilityRules for the host + day_of_week
  3. Check for AvailabilityOverride on that date (override wins)
  4. Slice the available window into slots of `duration_minutes`
  5. Remove slots that conflict with existing non-cancelled appointments (including buffer)
  6. Remove slots that violate `min_notice_hours` or `max_advance_days`
  7. If `max_bookings_per_day` is set, check the count for that date
  8. Return remaining slots, cached in Redis for 60 seconds
- Errors: 400 (missing date, invalid date format, invalid timezone), 404 (booking link not found or not active)

**POST /booking-links/:slug/book** (PUBLIC, rate-limited)
- Body: `{ start_time, guest_name, guest_email, guest_notes?, timezone, idempotency_key }`
- Response 201:
```json
{
  "appointment": {
    "_id": "...",
    "host_id": "...",
    "booking_link_id": "...",
    "guest_name": "Jane Smith",
    "guest_email": "jane@example.com",
    "start_time": "2025-01-15T14:00:00.000Z",
    "end_time": "2025-01-15T14:30:00.000Z",
    "duration_minutes": 30,
    "timezone": "America/New_York",
    "status": "pending"
  }
}
```
- Business logic:
  1. Find booking link by slug, verify it is active
  2. Validate all inputs (email, name, start_time format, timezone)
  3. Check idempotency: if `idempotency_key` already exists, return existing appointment
  4. Calculate `end_time = start_time + duration_minutes`
  5. Verify the slot is still available (re-check conflicts with buffer)
  6. Verify `min_notice_hours` and `max_advance_days`
  7. Verify `max_bookings_per_day` not exceeded
  8. Create appointment with status `pending`
  9. Invalidate availability cache for this host + date
  10. Trigger webhook delivery for `appointment.created`
- Errors: 400 (validation, slot unavailable, daily limit reached, too soon, too far), 404 (link not found / inactive), 409 (time slot conflict)

**GET /appointments** (Authenticated, host only)
- Query: `{ status?, from_date?, to_date?, guest_email?, page?, limit?, sort_by?, sort_order? }`
- Response 200:
```json
{
  "appointments": [ ... ],
  "pagination": {
    "page": 1,
    "limit": 20,
    "total": 150,
    "totalPages": 8,
    "hasNextPage": true,
    "hasPrevPage": false
  }
}
```
- Business logic: Filter by status (comma-separated), date range, guest email. Default sort by `start_time` descending. Clamped pagination (page min 1, limit 1-100).
- Only returns appointments where the authenticated user is the host. Only returns non-deleted appointments.

**PATCH /appointments/:id/status**
- Body: `{ status }` (one of: `confirmed`, `completed`, `no_show`)
- Response 200: `{ appointment: { ... } }`
- Business logic: State machine validation. Only host can change status. `completed` only allowed if `end_time` has passed. `no_show` only allowed if `start_time` has passed.
- Triggers appropriate webhook events.
- Errors: 400 (invalid transition, premature completion), 404 (not found / not owner)

**PATCH /appointments/:id/cancel**
- Body: `{ cancellation_reason?, cancelled_by }` (cancelled_by: "host" or "guest")
- Response 200: `{ appointment: { ... } }`
- Business logic: Only pending or confirmed appointments can be cancelled. If `cancelled_by: "host"`, requires auth. If `cancelled_by: "guest"`, optionally requires a guest cancellation token or simply the guest_email in the body (simplified for assessment scope). Invalidates availability cache. Triggers `appointment.cancelled` webhook.
- Errors: 400 (already cancelled/completed), 404 (not found)

**PATCH /appointments/:id/reschedule**
- Body: `{ new_start_time, timezone }`
- Response 200: `{ appointment: { ... } }`
- Business logic: Only confirmed appointments can be rescheduled. Calculate new end_time. Check slot availability (conflict detection with buffer). Update start_time, end_time. Invalidate cache. Trigger `appointment.rescheduled` webhook.
- Errors: 400 (not confirmed, slot unavailable, too soon), 404 (not found / not owner)

### 3.6 Webhook Endpoints

| # | Method | Path | Auth | Description |
|---|---|---|---|---|
| 24 | GET | `/webhooks/subscriptions` | Yes | List host's webhook subscriptions |
| 25 | POST | `/webhooks/subscriptions` | Yes | Create a webhook subscription |
| 26 | DELETE | `/webhooks/subscriptions/:id` | Yes | Soft-delete a subscription |
| 27 | GET | `/webhooks/deliveries` | Yes | List recent webhook deliveries (paginated) |
| 28 | POST | `/webhooks/deliveries/:id/retry` | Yes | Manually retry a failed delivery |

**POST /webhooks/subscriptions**
- Body: `{ url, events }` (events is array of event type strings)
- Response 201: `{ subscription: { ..., secret: "whsec_..." } }` (secret only shown on create)
- Business logic: Validate URL starts with https://, validate event types. Auto-generate signing secret.
- Errors: 400 (invalid URL, invalid event types)

**POST /webhooks/deliveries/:id/retry**
- Response 200: `{ delivery: { ... } }`
- Business logic: Only retry deliveries with status `failed` or `exhausted`. Reset attempt_count, set status to `pending`, calculate next_retry_at.
- Errors: 400 (delivery not retryable), 404 (not found / not owner)

---

## 4. Project Structure

```
src/
  app.js                              -- Express app setup: middleware, routes, swagger, error handler
  server.js                           -- HTTP server start, DB + Redis connection
  seed.js                             -- Database seeder with initial data
  reset.js                            -- Database reset utility

  config/
    env.js                            -- Environment variables (port, mongoUri, redisUrl, jwtSecret)
    db.js                             -- MongoDB connection via mongoose
    redis.js                          -- Redis connection via ioredis
    swagger.js                        -- Swagger/OpenAPI spec definition

  models/
    User.js                           -- User schema, password hashing, soft-delete
    AvailabilityRule.js               -- Recurring weekly availability rules
    AvailabilityOverride.js           -- Date-specific availability exceptions
    BookingLink.js                    -- Shareable booking link with settings
    Appointment.js                    -- Booked meeting, state machine, idempotency
    WebhookSubscription.js            -- Registered webhook endpoints
    WebhookDelivery.js                -- Delivery attempt log

  services/
    auth.service.js                   -- Register, login, JWT generation
    availability.service.js           -- Slot calculation, conflict detection, cache integration
    bookingLink.service.js            -- CRUD for booking links, status transitions
    appointment.service.js            -- Booking, cancellation, rescheduling, status changes
    webhook.service.js                -- Webhook dispatch, delivery, retry logic
    cache.service.js                  -- Redis get/set/invalidate helpers

  controllers/
    auth.controller.js                -- Auth route handlers
    availability.controller.js        -- Availability rules + overrides route handlers
    bookingLink.controller.js         -- Booking link route handlers
    appointment.controller.js         -- Appointment route handlers
    webhook.controller.js             -- Webhook subscription + delivery route handlers

  routes/
    index.js                          -- Route aggregator
    auth.routes.js                    -- /auth/*
    availability.routes.js            -- /availability/*
    bookingLink.routes.js             -- /booking-links/*
    appointment.routes.js             -- /appointments/*
    webhook.routes.js                 -- /webhooks/*

  middleware/
    auth.js                           -- JWT verification, req.user population
    authorize.js                      -- Role-based authorization
    errorHandler.js                   -- Global error handler
    rateLimiter.js                    -- Redis-based rate limiting
    sanitize.js                       -- NoSQL injection prevention

  utils/
    AppError.js                       -- Custom error classes (BadRequest, NotFound, Conflict, etc.)
    helpers.js                        -- roundMoney, email regex, date utils, timezone validation
    softDelete.plugin.js              -- Mongoose plugin: findActive, findOneActive, countActive

  pkg/features/
    webhook_delivery/                 -- Feature Task 1: Webhook delivery with retry
      controller.js
      routes.js
      service.js
      tests/
        integration_test.js
        test_fixtures.js

    appointment_slots/                -- Feature Task 2: Available slot calculation
      controller.js
      routes.js
      service.js
      tests/
        integration_test.js
        test_fixtures.js

    appointment_reschedule/           -- Feature Task 3: Appointment rescheduling
      controller.js
      routes.js
      service.js
      tests/
        integration_test.js
        test_fixtures.js

    appointment_history/              -- Feature Task 4: Paginated appointment history
      controller.js
      routes.js
      service.js
      tests/
        integration_test.js
        test_fixtures.js

test/
  task5/
    app.spec.js                       -- Bug: Appointment state machine pre-save hooks
  task6/
    app.spec.js                       -- Bug: Booking link rate limiting
  task7/
    app.spec.js                       -- Bug: Availability cache invalidation
  task8/
    app.spec.js                       -- Bug: Double-booking prevention (idempotency)
  task9/
    app.spec.js                       -- Bug: Soft-delete leaking into slot calculation
```

**Total source files: ~15 files** (7 models, 6 services, 5 controllers, 5 route files, 4 middleware, 3 utils, 3 config, plus app.js/server.js/seed.js/reset.js).

---

## 5. Status Lifecycles

### 5.1 Appointment Statuses

```
                    +-----------+
                    |  pending  |
                    +-----------+
                   /             \
                  v               v
          +-----------+     +-----------+
          | confirmed |     | cancelled |
          +-----------+     +-----------+
         /      |      \
        v       v       v
+-----------+ +-----------+ +-----------+
| completed | |  no_show  | | cancelled |
+-----------+ +-----------+ +-----------+
```

**Valid transitions:**
```
pending    --> confirmed, cancelled
confirmed  --> completed, cancelled, no_show
completed  --> (terminal)
cancelled  --> (terminal)
no_show    --> (terminal)
```

**Rules:**
- `completed` only allowed if `end_time <= now`
- `no_show` only allowed if `start_time <= now`
- `cancelled` allowed from pending or confirmed (not from completed/no_show)
- Only host can confirm, complete, or mark no_show
- Both host and guest can cancel

### 5.2 Webhook Delivery Statuses

```
+---------+
| pending |
+---------+
    |
    v
+-- attempt delivery -->--+
|                         |
v                         v
+-----------+       +---------+
| delivered |       |  failed |
+-----------+       +---------+
                        |
                        v
               +-- retry? --+
               |            |
               v            v
           +---------+  +-----------+
           | pending |  | exhausted |
           +---------+  +-----------+
```

**Valid transitions:**
```
pending   --> delivered, failed
failed    --> pending (retry), exhausted (max attempts reached)
exhausted --> pending (manual retry)
delivered --> (terminal)
```

### 5.3 BookingLink Statuses

```
+--------+
| active |
+--------+
    |   ^
    v   |
+--------+
| paused |
+--------+
    |
    v
+----------+
| archived |
+----------+
```

**Valid transitions:**
```
active   --> paused, archived
paused   --> active, archived
archived --> (terminal)
```

---

## 6. The 9 Tasks (4 Features + 5 Bugs)

---

### Task 1: Webhook Delivery with Retry (Feature -- Hard)

**Type:** Feature
**Difficulty:** Hard
**Time estimate:** 60 minutes

**Problem Statement:**

MeetSync has a webhook subscription system where hosts can register URLs to receive notifications about appointment events (created, confirmed, cancelled, etc.). Subscriptions can be created and stored, but the actual delivery mechanism does not exist yet.

Currently, when an appointment is created, confirmed, or cancelled, no webhook notifications are sent. The webhook delivery system needs to be built from scratch. This includes dispatching webhooks to subscriber URLs, logging each delivery attempt, handling HTTP failures with exponential backoff retry, and signing payloads with HMAC-SHA256 so subscribers can verify authenticity.

Build the webhook delivery system that: (a) finds all active subscriptions matching the event type, (b) creates a WebhookDelivery record for each, (c) attempts HTTP POST to the subscriber URL with a signed payload, (d) marks successful deliveries (2xx response) as delivered, (e) marks failed deliveries and schedules retries with exponential backoff (1 min, 5 min, 15 min, 60 min, 240 min), (f) after 5 failed attempts marks the delivery as exhausted.

**Which files are affected:**
- `src/pkg/features/webhook_delivery/service.js` -- Main implementation (TODO placeholder)
- `src/pkg/features/webhook_delivery/controller.js` -- Route handler wiring
- `src/pkg/features/webhook_delivery/routes.js` -- Route definitions
- `src/services/webhook.service.js` -- May need stub updates for dispatch trigger

**What the solution involves:**
1. Implement `dispatchWebhook(eventType, appointmentData)` that:
   - Queries `WebhookSubscription.findActive({ host_id, events: eventType, status: 'active' })`
   - For each subscription, creates a `WebhookDelivery` record with `delivery_id` (UUID), status `pending`, payload, and attempt_count 0
   - Calls `attemptDelivery(deliveryId)` for each
2. Implement `attemptDelivery(deliveryId)` that:
   - Loads the delivery and its subscription
   - Computes HMAC-SHA256 signature of the JSON payload using the subscription's secret
   - Makes HTTP POST with headers: `Content-Type: application/json`, `X-MeetSync-Signature: sha256=...`, `X-MeetSync-Delivery-Id: ...`
   - On 2xx: update status to `delivered`, set http_status
   - On non-2xx or network error: increment attempt_count, set http_status (if available), set response_body (truncated)
   - If attempt_count >= max_attempts: set status to `exhausted`
   - Else: set status to `failed`, calculate next_retry_at using backoff schedule
3. Implement `retryDelivery(deliveryId)` for manual retry endpoint
4. Backoff schedule: `[60, 300, 900, 3600, 14400]` seconds (1m, 5m, 15m, 1h, 4h)

**Test cases (12):**
1. Creates delivery records for all matching active subscriptions
2. Sets delivery status to `delivered` on 2xx response
3. Sets delivery status to `failed` on non-2xx response
4. Increments attempt_count on each failed delivery
5. Calculates correct next_retry_at using exponential backoff
6. Sets status to `exhausted` after max_attempts failures
7. Includes correct HMAC-SHA256 signature in X-MeetSync-Signature header
8. Includes delivery_id in X-MeetSync-Delivery-Id header
9. Skips inactive subscriptions (status: inactive)
10. Skips subscriptions not matching the event type
11. Manual retry resets a failed delivery to pending with attempt_count reset
12. Manual retry rejects already-delivered webhooks

**Why this is a real-world issue:** Webhook reliability is a core requirement for any API platform. Stripe, GitHub, and Twilio all implement exactly this pattern: signed payloads, delivery logging, exponential backoff retry.

**Anti-gaming notes:** Tests use a mock HTTP server to verify actual HTTP requests are made with correct headers and signatures. The HMAC computation is verified with a mirror calculation in tests using random secrets. Backoff intervals are checked against the formula, not hardcoded values.

---

### Task 2: Available Slot Calculation (Feature -- Medium)

**Type:** Feature
**Difficulty:** Medium
**Time estimate:** 45 minutes

**Problem Statement:**

MeetSync allows hosts to define their weekly availability and create booking links with specific durations. However, the slot calculation engine does not exist yet. When a guest visits a booking link and requests available times for a specific date, the system should return the open time slots.

Build the slot calculation endpoint `GET /api/v1/booking-links/:slug/slots?date=YYYY-MM-DD&timezone=America/New_York` that: (a) looks up the host's availability rules for the given day of the week, (b) checks for availability overrides on that specific date (overrides take precedence), (c) slices the available window into meeting-duration-sized slots, (d) removes slots that conflict with existing non-cancelled appointments (including buffer time), (e) removes slots that violate the booking link's `min_notice_hours` and `max_advance_days`, (f) enforces `max_bookings_per_day` limit, (g) caches the result in Redis for 60 seconds.

**Which files are affected:**
- `src/pkg/features/appointment_slots/service.js` -- Main implementation (TODO placeholder)
- `src/pkg/features/appointment_slots/controller.js` -- Route handler
- `src/pkg/features/appointment_slots/routes.js` -- Route definitions
- `src/services/availability.service.js` -- May need supporting utility calls

**What the solution involves:**
1. Parse and validate `date` (YYYY-MM-DD format) and `timezone` (valid IANA zone, default to host's timezone)
2. Determine the day_of_week from the date
3. Check for AvailabilityOverride for this host + date:
   - If `is_available: false` -- return empty slots
   - If `is_available: true` -- use override's start_time/end_time
   - If no override -- use AvailabilityRules for that day_of_week
4. Convert window times to UTC using the rule/override timezone
5. Generate slots of `duration_minutes` length within the window
6. Query existing appointments for the host on that date (status not cancelled)
7. Filter out slots that overlap with existing appointments + buffer_before + buffer_after
8. Filter out slots earlier than `now + min_notice_hours`
9. Filter out slots later than `now + max_advance_days * 24 hours`
10. If `max_bookings_per_day` is set and daily count >= limit, return empty
11. Cache result in Redis with key `slots:{slug}:{date}:{timezone}`, TTL 60s
12. Return slot list with start_time and end_time in the requested timezone

**Test cases (12):**
1. Returns correct slots for a normal weekday with rules
2. Returns empty slots for a day with no rules
3. Override `is_available: false` returns empty slots
4. Override `is_available: true` uses override times instead of rules
5. Existing appointment removes its slot from results
6. Buffer time removes adjacent slots
7. Slots before min_notice_hours are excluded
8. Slots beyond max_advance_days are excluded
9. max_bookings_per_day returns empty when limit reached
10. Invalid date format returns 400
11. Invalid timezone returns 400
12. Inactive booking link returns 404

**Why this is a real-world issue:** Slot calculation is the core algorithm of any scheduling system. It requires combining multiple data sources (rules, overrides, existing bookings) and performing correct time arithmetic across timezones.

**Anti-gaming notes:** Tests use dynamically generated availability rules and appointments with random times. Slot boundaries are verified mathematically (start + duration = end). Timezone conversion is tested with non-UTC timezones. Buffer math is tested with randomized buffer values.

---

### Task 3: Appointment Rescheduling (Feature -- Hard)

**Type:** Feature
**Difficulty:** Hard
**Time estimate:** 60 minutes

**Problem Statement:**

MeetSync supports creating and cancelling appointments, but there is no way to reschedule. When a host or guest needs to change the time of a confirmed appointment, they currently must cancel and rebook, which loses the original appointment record and guest information.

Build a reschedule endpoint `PATCH /api/v1/appointments/:id/reschedule` that: (a) only works on confirmed appointments, (b) validates the new time slot is available (no conflicts with other appointments, respects buffer time), (c) validates the new time respects `min_notice_hours` and `max_advance_days` from the booking link, (d) updates the appointment's start_time and end_time while preserving all other fields, (e) invalidates the availability cache for both the old and new dates, (f) triggers an `appointment.rescheduled` webhook with both old and new times in the payload.

**Which files are affected:**
- `src/pkg/features/appointment_reschedule/service.js` -- Main implementation (TODO placeholder)
- `src/pkg/features/appointment_reschedule/controller.js` -- Route handler
- `src/pkg/features/appointment_reschedule/routes.js` -- Route definitions

**What the solution involves:**
1. Load appointment by ID, verify host ownership, verify status is `confirmed`
2. Load the associated BookingLink for constraints (duration, buffer, min_notice, max_advance)
3. Calculate new `end_time = new_start_time + duration_minutes`
4. Validate the new slot:
   - Not in the past
   - Respects `min_notice_hours`
   - Within `max_advance_days`
   - Check host availability (rules + overrides) for the new date
   - Check for conflicts with other non-cancelled appointments (exclude the current one)
   - Apply buffer_before and buffer_after to conflict detection
5. Save old times for webhook payload
6. Update appointment's start_time, end_time
7. Invalidate cache for old date and new date
8. Dispatch `appointment.rescheduled` webhook with `{ old_start_time, old_end_time, new_start_time, new_end_time }`
9. Return updated appointment

**Test cases (12):**
1. Successfully reschedules a confirmed appointment
2. Rejects rescheduling a pending appointment (must be confirmed)
3. Rejects rescheduling a cancelled appointment
4. Rejects when new time conflicts with another appointment
5. Rejects when new time violates buffer (adjacent appointment within buffer window)
6. Rejects when new time is before min_notice_hours
7. Rejects when new time is beyond max_advance_days
8. Rejects when new time is in the past
9. Returns 404 when appointment does not exist
10. Returns 404 when user does not own the appointment
11. Invalidates availability cache for both old and new dates
12. Returns 401 when not authenticated

**Why this is a real-world issue:** Rescheduling is one of the most requested features in any scheduling system. It requires combining conflict detection, time validation, and cache invalidation in a single atomic operation. The "exclude current appointment from conflict check" requirement is a common source of bugs.

**Anti-gaming notes:** Tests create multiple appointments and verify only the specific one is rescheduled. Conflict detection is tested with appointments at exact boundaries (end of one = start of another, with and without buffer). Cache invalidation is verified by checking Redis keys.

---

### Task 4: Paginated Appointment History with Filtering (Feature -- Easy)

**Type:** Feature
**Difficulty:** Easy
**Time estimate:** 30 minutes

**Problem Statement:**

MeetSync allows hosts to view their appointments, but the current listing endpoint returns all appointments in a flat array with no filtering, sorting, or pagination. A host with hundreds of appointments gets an enormous response that is slow and impractical.

Build the appointment listing endpoint `GET /api/v1/appointments` with support for: (a) filtering by status (single or comma-separated), (b) filtering by date range (from_date, to_date), (c) filtering by guest_email, (d) pagination with page and limit parameters (limit clamped to 1-100, page clamped to min 1), (e) sorting by start_time (ascending or descending), (f) proper pagination metadata (page, limit, total, totalPages, hasNextPage, hasPrevPage).

**Which files are affected:**
- `src/pkg/features/appointment_history/service.js` -- Main implementation (TODO placeholder)
- `src/pkg/features/appointment_history/controller.js` -- Route handler
- `src/pkg/features/appointment_history/routes.js` -- Route definitions

**What the solution involves:**
1. Parse query parameters: `status`, `from_date`, `to_date`, `guest_email`, `page`, `limit`, `sort_order`
2. Build MongoDB query:
   - `{ host_id, deleted_at: null }` (base -- soft-delete aware)
   - If `status`: split by comma, use `$in` for multiple or exact match for single
   - If `from_date`: add `start_time: { $gte: new Date(from_date) }`
   - If `to_date`: add `start_time: { $lte: new Date(to_date) }`
   - If `guest_email`: add `guest_email: guest_email.toLowerCase()`
3. Clamp pagination: `page = Math.max(1, parseInt(page) || 1)`, `limit = Math.min(100, Math.max(1, parseInt(limit) || 20))`
4. Sort: `start_time` with direction based on `sort_order` (default `desc`)
5. Execute query with skip/limit
6. Count total matching documents
7. Return `{ appointments, pagination: { page, limit, total, totalPages, hasNextPage, hasPrevPage } }`

**Test cases (10):**
1. Returns all appointments for the host with pagination metadata
2. Filters by single status
3. Filters by comma-separated statuses
4. Filters by date range (from_date + to_date)
5. Filters by guest_email (case-insensitive)
6. Clamps page to 1 when page is 0 or negative
7. Caps limit at 100 when limit exceeds 100
8. Clamps limit to 1 when limit is 0 or negative
9. Returns correct hasNextPage and hasPrevPage
10. Returns only host's own appointments (not other hosts')

**Why this is a real-world issue:** Every API that returns lists needs pagination and filtering. This is a foundational skill that tests understanding of query building, parameter validation, and response structure.

**Anti-gaming notes:** Tests create a known number of appointments with specific statuses and dates, then verify exact counts after filtering. Random guest emails are used to ensure filtering is dynamic. Pagination math (totalPages, hasNextPage) is verified against the total count.

---

### Task 5: Appointment State Machine Pre-Save Hooks (Bug -- Medium)

**Type:** Bug
**Difficulty:** Medium
**Time estimate:** 45 minutes

**Problem Statement:**

Appointments follow a strict lifecycle: they start as `pending`, can be `confirmed`, then `completed` or marked as `no_show`, and can be `cancelled` from either `pending` or `confirmed`. However, the Appointment model currently has no validation at all.

Three issues have been reported:

1. **Invalid status transitions:** Any status change is accepted. An appointment can jump from `pending` directly to `completed`, or a `cancelled` appointment can be reverted to `confirmed`. There are no rules enforcing the correct lifecycle.

2. **Premature completion:** Appointments can be marked as `completed` before their `end_time` has passed. A meeting at 3pm can be marked complete at 1pm.

3. **Premature no-show:** Appointments can be marked as `no_show` before their `start_time` has passed. You cannot know someone is a no-show before the meeting was supposed to start.

The fixes should use pre-save hooks on the Appointment model to enforce valid status transitions, reject premature completion (end_time must be in the past), and reject premature no-show marking (start_time must be in the past).

**Which files are affected:**
- `src/models/Appointment.js` -- Add pre-save hooks and post-init hook

**What the solution involves:**
1. Add `post('init')` hook to capture `this._original_status = this.status`
2. Add `pre('save')` hook that:
   - Validates `end_time > start_time` (cross-field date validation)
   - If not `isNew` and `isModified('status')`:
     - Look up `VALID_APPOINTMENT_TRANSITIONS[this._original_status]`
     - If new status not in allowed list, throw error with message including both statuses
   - If new status is `completed` and `this.end_time > new Date()`, throw error
   - If new status is `no_show` and `this.start_time > new Date()`, throw error

**Test cases (10):**
1. Rejects invalid transition from pending to completed
2. Allows valid transition from pending to confirmed
3. Rejects transition from cancelled (terminal state)
4. Rejects transition from completed (terminal state)
5. Allows valid transition from confirmed to completed (when end_time passed)
6. Rejects premature completion (end_time in future)
7. Allows no_show when start_time has passed
8. Rejects premature no_show (start_time in future)
9. Rejects end_time before start_time
10. Hooks fire when saving directly (bypassing service layer)

**Why this is a real-world issue:** State machine enforcement at the model level is a defense-in-depth pattern. Even if the service layer has validation, the model hooks catch bugs from direct database operations, seed scripts, or admin tools.

**Anti-gaming notes:** Tests create appointments directly via the model (bypassing the service layer) to verify hooks fire regardless of the call path. Both valid and invalid transitions are tested for each state. Time-based validation uses precise timestamps.

---

### Task 6: Booking Link Rate Limiting (Bug -- Easy)

**Type:** Bug
**Difficulty:** Easy
**Time estimate:** 30 minutes

**Problem Statement:**

MeetSync's public booking endpoints (`GET /booking-links/:slug/slots` and `POST /booking-links/:slug/book`) are accessible without authentication to allow guests to browse and book appointments. However, these endpoints have no rate limiting, making them vulnerable to abuse -- a malicious actor could scrape all available slots or spam booking requests.

Two issues have been reported:

1. **No rate limiting on slot queries:** The `GET /booking-links/:slug/slots` endpoint can be called thousands of times per minute without throttling.

2. **No rate limiting on bookings:** The `POST /booking-links/:slug/book` endpoint has no request cap, allowing rapid-fire booking attempts.

The rate limiter middleware exists in `src/middleware/rateLimiter.js` and works correctly for other endpoints. The fix is to apply a stricter rate limit to the public booking routes: 30 requests per minute for slot queries and 10 requests per minute for booking attempts.

**Which files are affected:**
- `src/routes/bookingLink.routes.js` -- Apply rate limiter middleware to specific routes

**What the solution involves:**
1. Import the `rateLimiter` middleware in the booking link routes file
2. Apply `rateLimiter(30, 60)` to the `GET /:slug/slots` route
3. Apply `rateLimiter(10, 60)` to the `POST /:slug/book` route
4. The rate limiter uses Redis with `rate_limit:{ip}:{route}` keys -- the route-specific key must be passed to distinguish from the global limiter

Actually, looking more carefully at the ShowPass pattern, the rateLimiter needs to be made route-aware. The current implementation uses `req.ip` only. The bug is that the rateLimiter function does not include route-specific keys, so all routes share one counter. The fix involves:
1. Update `rateLimiter` to accept an optional `prefix` parameter for route-specific keys
2. Apply route-specific rate limits to the public booking endpoints

**Test cases (10):**
1. Slot query succeeds within rate limit (under 30/min)
2. Slot query returns 429 after exceeding 30 requests per minute
3. Booking attempt succeeds within rate limit (under 10/min)
4. Booking attempt returns 429 after exceeding 10 requests per minute
5. Rate limit resets after the window expires
6. Different IPs have independent rate limits
7. Rate limit on booking is stricter than slot query (10 vs 30)
8. 429 response includes correct error message
9. Rate limit does not affect authenticated host endpoints
10. Global rate limit still applies to non-booking routes

**Why this is a real-world issue:** Public endpoints without rate limiting are a security vulnerability. Calendly, Stripe, and every production API implement route-specific rate limits. This task tests understanding of middleware composition.

**Anti-gaming notes:** Tests make a precise number of requests and verify the exact threshold where 429 kicks in. The window timing is tested by verifying the Redis TTL on the rate limit key.

---

### Task 7: Availability Cache Invalidation (Bug -- Medium)

**Type:** Bug
**Difficulty:** Medium
**Time estimate:** 40 minutes

**Problem Statement:**

MeetSync caches available time slots in Redis to avoid recalculating them on every request. However, several bugs have been reported with stale cache data.

Three issues need to be fixed:

1. **Stale slots after booking:** When a guest books an appointment, the available slots cache is not invalidated. Other guests still see the just-booked slot as available, leading to conflict errors when they try to book it.

2. **Stale slots after cancellation:** When an appointment is cancelled, the freed-up slot does not appear in the available slots because the cache still has the old data.

3. **Stale slots after availability change:** When a host updates their availability rules or creates an override, the cached slots still reflect the old availability.

The fixes should invalidate the relevant Redis cache keys whenever an appointment is created, cancelled, or rescheduled, and whenever availability rules or overrides are modified.

**Which files are affected:**
- `src/services/appointment.service.js` -- Add cache invalidation after booking/cancellation
- `src/services/availability.service.js` -- Add cache invalidation after rule/override changes

**What the solution involves:**
1. After creating an appointment: `invalidateCache('slots:*:{host_id}:*')` or more specifically `invalidateCache('slots:*')` for the host's slots
2. After cancelling an appointment: same invalidation
3. After updating/creating/deleting availability rules: `invalidateCache('slots:*:{host_id}:*')`
4. After creating/updating/deleting availability overrides: same
5. The cache key pattern is `slots:{slug}:{date}:{timezone}` -- invalidation should match all keys for the host's booking links

**Test cases (10):**
1. Cache is populated after first slot query
2. Booking an appointment invalidates the slot cache
3. After booking, next slot query reflects the booked slot as unavailable
4. Cancelling an appointment invalidates the slot cache
5. After cancellation, next slot query shows the freed slot
6. Creating an availability rule invalidates the slot cache
7. Deleting an availability rule invalidates the slot cache
8. Creating an availability override invalidates the slot cache
9. Cache key pattern matches the expected format
10. Cache TTL is 60 seconds

**Why this is a real-world issue:** Cache invalidation is one of the hardest problems in computer science. Stale data in scheduling systems leads to double-bookings, which is one of the most damaging user experiences possible.

**Anti-gaming notes:** Tests verify Redis key existence and content directly. They populate the cache, perform the mutation, then verify the cache keys are gone. New queries after mutation are verified to return correct (fresh) data.

---

### Task 8: Double-Booking Prevention / Idempotency (Bug -- Hard)

**Type:** Bug
**Difficulty:** Hard
**Time estimate:** 45 minutes

**Problem Statement:**

MeetSync's booking endpoint has a critical vulnerability: it does not properly prevent double-bookings. When two guests attempt to book the same time slot simultaneously, both bookings can succeed, creating a scheduling conflict.

Five problems have been reported:

1. **No conflict detection:** The booking service does not check for existing appointments at the requested time before creating a new one.

2. **Buffer time ignored:** Even when adjacent slots are booked, the buffer time (before and after) is not considered when checking for conflicts.

3. **Idempotency key not checked:** The `idempotency_key` field exists on the Appointment model but is never validated. Duplicate submissions create duplicate appointments instead of returning the existing one.

4. **Soft-deleted appointments counted:** Cancelled (soft-deleted) appointments are included in the conflict check, blocking guests from booking freed-up slots.

5. **Race condition on booking count:** The `max_bookings_per_day` check and the appointment creation are not atomic, allowing the daily limit to be exceeded under concurrent requests.

The fixes should add conflict detection (checking for overlapping non-deleted appointments including buffer time), implement idempotency key deduplication, exclude soft-deleted appointments from conflict checks, and use a Redis lock or MongoDB findOneAndUpdate to make the daily count check atomic.

**Which files are affected:**
- `src/services/appointment.service.js` -- Add conflict detection, idempotency, atomic booking

**What the solution involves:**
1. Before creating appointment, check idempotency_key: if exists, return existing appointment
2. Query for conflicting appointments:
   ```javascript
   const conflict = await Appointment.findOneActive({
     host_id,
     status: { $nin: ['cancelled'] },
     start_time: { $lt: newEndTimeWithBuffer },
     end_time: { $gt: newStartTimeWithBuffer },
   });
   ```
   Where buffer is applied: `newStartTimeWithBuffer = start_time - buffer_before`, `newEndTimeWithBuffer = end_time + buffer_after`
3. Use `findOneActive` (soft-delete aware) to exclude cancelled/deleted appointments
4. For max_bookings_per_day atomicity, use Redis INCR with TTL or MongoDB `$inc` with conditional check
5. Return 409 Conflict error with details when a time slot is already taken

**Test cases (12):**
1. Successfully books when slot is available
2. Returns 409 when slot directly overlaps with existing appointment
3. Returns 409 when slot overlaps with buffer zone of existing appointment
4. Idempotency key returns existing appointment without creating duplicate
5. Cancelled (soft-deleted) appointment does not block new booking
6. Daily limit is enforced (returns 400 when max_bookings_per_day reached)
7. Daily limit counts only non-cancelled appointments
8. Adjacent appointments are allowed when no buffer (end of one = start of next)
9. Adjacent appointments blocked when buffer would cause overlap
10. Conflict check uses correct time comparison (< and >, not <= and >=)
11. Returns the original appointment data on idempotent retry
12. Returns 400 when idempotency_key is missing from request

**Why this is a real-world issue:** Double-booking is the most critical bug in any scheduling system. It directly damages user trust. Race conditions under concurrent load are a common source of production incidents. Idempotency is essential for reliable API design.

**Anti-gaming notes:** Tests create appointments at specific times and verify conflict detection with millisecond precision. Buffer calculations use random buffer values. Idempotency is tested with the same key sent twice and different keys for the same slot. The daily count test creates exactly `max_bookings_per_day` appointments and verifies the next one is rejected.

---

### Task 9: Soft-Delete Leaking into Slot Calculation (Bug -- Easy)

**Type:** Bug
**Difficulty:** Easy
**Time estimate:** 30 minutes

**Problem Statement:**

MeetSync uses soft-delete for all records. When an appointment is cancelled, its `deleted_at` field is set but the record stays in the database. However, the slot calculation and booking conflict check are including cancelled/deleted appointments, causing two problems:

1. **Deleted appointments block slots:** When an appointment is cancelled, its time slot should become available again. But the slot calculation query does not filter out cancelled or soft-deleted appointments, so the slot stays blocked.

2. **Deleted availability rules still apply:** When a host deletes an availability rule, it should no longer affect slot calculations. But the availability query does not use `findActive`, so deleted rules still generate slots.

The fix requires changing specific queries from `find()` to `findActive()` and adding `deleted_at: null` conditions where the soft-delete plugin methods are not used.

**Which files are affected:**
- `src/services/availability.service.js` -- Change `find()` calls to `findActive()` for rules/overrides/appointments

**What the solution involves:**
1. In the slot calculation function, change appointment conflict query from:
   ```javascript
   Appointment.find({ host_id, start_time: ..., end_time: ... })
   ```
   to:
   ```javascript
   Appointment.findActive({ host_id, status: { $ne: 'cancelled' }, start_time: ..., end_time: ... })
   ```
2. Change availability rule query from `AvailabilityRule.find({ host_id, day_of_week })` to `AvailabilityRule.findActive({ host_id, day_of_week })`
3. Change availability override query from `AvailabilityOverride.findOne({ host_id, date })` to `AvailabilityOverride.findOneActive({ host_id, date })`

**Test cases (8):**
1. Cancelled appointment's slot appears as available
2. Soft-deleted appointment (deleted_at set) slot appears as available
3. Active (non-deleted) appointment's slot is correctly blocked
4. Deleted availability rule does not generate slots
5. Active availability rule generates correct slots
6. Deleted availability override does not affect slot calculation
7. Active override correctly blocks or customizes slots
8. Multiple cancelled appointments on same day all free up their slots

**Why this is a real-world issue:** Soft-delete bugs are extremely common in production systems. The pattern of forgetting to use soft-delete-aware queries is one of the most frequent sources of data leakage in enterprise applications.

**Anti-gaming notes:** Tests create appointments, cancel them (setting deleted_at), and verify the slots are freed. Active and cancelled appointments are mixed to ensure selective filtering. Availability rules are created, deleted, and verified independently.

---

## 7. Test Strategy

### Framework

- **Mocha** as the test runner (consistent with HackerRank platform support)
- **Chai** with `chai-http` for HTTP assertions
- **supertest** available as an alternative
- **mocha-junit-reporter** + **mocha-multi-reporters** for HackerRank XML output

### Test Structure

Each feature task has tests in `src/pkg/features/{feature_name}/tests/integration_test.js` with a companion `test_fixtures.js`. Each bug task has tests in `test/task{N}/app.spec.js`.

Tests are integration tests that start the Express app, connect to MongoDB and Redis, and make real HTTP requests.

### Test Data Generation

Every test file has a `test_fixtures.js` with factory functions following the ShowPass pattern:

```javascript
// Factory functions with random data and override support
export const createUser = (overrides = {}) =>
  User.create({
    name: 'Test Host',
    email: `host-${new mongoose.Types.ObjectId()}@test.com`,
    password: 'password123',
    role: 'host',
    timezone: 'America/New_York',
    ...overrides,
  });

export const createAvailabilityRule = (overrides = {}) =>
  AvailabilityRule.create({
    host_id: new mongoose.Types.ObjectId(),
    day_of_week: 1,  // Monday
    start_time: '09:00',
    end_time: '17:00',
    timezone: 'America/New_York',
    ...overrides,
  });

export const createBookingLink = (overrides = {}) =>
  BookingLink.create({
    host_id: new mongoose.Types.ObjectId(),
    slug: `link-${new mongoose.Types.ObjectId()}`,
    title: 'Test Meeting',
    duration_minutes: 30,
    buffer_before_minutes: 0,
    buffer_after_minutes: 0,
    max_bookings_per_day: null,
    min_notice_hours: 24,
    max_advance_days: 60,
    status: 'active',
    ...overrides,
  });

export const createAppointment = (overrides = {}) =>
  Appointment.create({
    host_id: new mongoose.Types.ObjectId(),
    booking_link_id: new mongoose.Types.ObjectId(),
    guest_name: 'Test Guest',
    guest_email: `guest-${new mongoose.Types.ObjectId()}@test.com`,
    start_time: new Date(Date.now() + 7 * 24 * 60 * 60 * 1000),
    end_time: new Date(Date.now() + 7 * 24 * 60 * 60 * 1000 + 30 * 60 * 1000),
    duration_minutes: 30,
    timezone: 'America/New_York',
    status: 'confirmed',
    idempotency_key: new mongoose.Types.ObjectId().toString(),
    ...overrides,
  });

// Helpers
export const futureDate = (days) => new Date(Date.now() + days * 24 * 60 * 60 * 1000);
export const pastDate = (hours) => new Date(Date.now() - hours * 60 * 60 * 1000);

export const cleanupModels = async (
  models = [WebhookDelivery, WebhookSubscription, Appointment, BookingLink, AvailabilityOverride, AvailabilityRule, User]
) => {
  await Promise.all(models.map((Model) => Model.deleteMany({})));
};
```

### Test Isolation

- Each test file connects to `mongodb://localhost:27017/meetsync_test`
- The `before` hook verifies the database name includes "test" (safety check)
- `beforeEach` cleans up per-test collections
- `after` cleans up all collections, flushes Redis, and closes connections
- Each test task runs on a different port (8081-8089) via `cross-env PORT=...`

### Anti-Gaming Measures

1. **Random data:** Factory functions use `new mongoose.Types.ObjectId()` for unique emails and slugs, `randomPrice()` for numeric values
2. **Mirror formulas:** Test fixtures include their own copies of business logic (time calculation, buffer math) to verify results independently
3. **Direct DB verification:** Tests query the database directly after API calls to verify side effects (status changes, record creation, cache invalidation)
4. **Multiple valid/invalid cases:** Each transition in a state machine is tested both positively and negatively
5. **Edge cases:** Exact boundaries (e.g., exactly at `min_notice_hours`), zero values, maximum values, empty arrays
6. **No hardcodable responses:** Tests use dynamic data, so returning hardcoded JSON will fail

---

## 8. Key Business Rules

### 8.1 How Availability Slots Are Calculated

The slot calculation algorithm runs in this order:

1. **Input:** `date` (YYYY-MM-DD), `timezone` (IANA), `booking_link` (with duration, buffer, limits)
2. **Determine base window:**
   - Check AvailabilityOverride for `(host_id, date)`:
     - If `is_available: false` => return empty slots
     - If `is_available: true` => window = override's start_time..end_time
   - If no override, check AvailabilityRules for `(host_id, day_of_week)`:
     - If no rules for that day => return empty slots
     - If multiple rules for same day, merge them (union of windows)
3. **Slice into slots:** Starting from window start, generate slots of `duration_minutes`. Each slot's end = slot's start + duration. Stop when end would exceed window end.
4. **Remove conflicts:** For each slot, check if any non-cancelled, non-deleted appointment for this host overlaps with `[slot_start - buffer_before, slot_end + buffer_after]`. If overlap, remove slot.
5. **Apply min_notice_hours:** Remove slots where `slot_start < now + min_notice_hours * 3600 * 1000`.
6. **Apply max_advance_days:** Remove slots where `slot_start > now + max_advance_days * 24 * 3600 * 1000`.
7. **Apply max_bookings_per_day:** Count non-cancelled appointments for host on this date. If count >= max_bookings_per_day, return empty.
8. **Cache result** with TTL 60 seconds.

### 8.2 How Buffer Time Works

Buffer time creates padding around each appointment to give the host preparation/transition time.

- `buffer_before_minutes`: Time blocked BEFORE the appointment. A 30-min meeting at 10:00 with 10-min buffer_before blocks 09:50-10:30.
- `buffer_after_minutes`: Time blocked AFTER the appointment. Same meeting with 10-min buffer_after blocks 10:00-10:40.

When checking for conflicts, the **effective occupied range** of an existing appointment is:
```
effective_start = appointment.start_time - buffer_before_minutes
effective_end   = appointment.end_time + buffer_after_minutes
```

Two time ranges conflict if:
```
range_A_start < range_B_end AND range_A_end > range_B_start
```

### 8.3 How Timezone Handling Works

- All dates stored in MongoDB are **UTC**.
- Availability rules store times as `HH:mm` strings with an explicit `timezone` field. When calculating slots, these times are converted to UTC using the timezone.
- Guests provide a `timezone` query parameter. Slot start/end times in the response are formatted in the guest's timezone.
- The `date` parameter refers to a calendar date in the **guest's timezone** (or the host's timezone if not specified).

**Example:** A host has availability Mon 09:00-17:00 EST (America/New_York). In UTC, this is 14:00-22:00. A guest in London (Europe/London) requesting Monday's slots sees them as 14:00-22:00 GMT (same as UTC in winter, offset in summer due to DST).

**Important:** DST transitions must be handled correctly. A rule for 09:00 EST in winter is 14:00 UTC, but in summer (EDT) it is 13:00 UTC.

For the assessment, timezone conversion should use simple offset calculation. The `Intl.DateTimeFormat` API can determine the UTC offset for a given timezone and date. Alternatively, store a timezone offset utility in `src/utils/helpers.js`.

### 8.4 How Conflict Detection Works

When booking or rescheduling, the system must verify no conflict exists:

```javascript
const conflictQuery = {
  host_id: hostId,
  deleted_at: null,
  status: { $nin: ['cancelled'] },
  start_time: { $lt: newEndWithBuffer },
  end_time: { $gt: newStartWithBuffer },
};

// When rescheduling, exclude the current appointment
if (excludeAppointmentId) {
  conflictQuery._id = { $ne: excludeAppointmentId };
}

const conflict = await Appointment.findOneActive(conflictQuery);
if (conflict) {
  throw new ConflictError('time slot is already booked');
}
```

### 8.5 How Webhook Retry Works (Backoff Schedule)

| Attempt | Delay | Cumulative |
|---|---|---|
| 1 (initial) | 0s | 0s |
| 2 (first retry) | 60s (1 min) | 1 min |
| 3 | 300s (5 min) | 6 min |
| 4 | 900s (15 min) | 21 min |
| 5 | 3600s (60 min) | 81 min |

After attempt 5 fails, status becomes `exhausted`.

**Backoff formula:** `BACKOFF_SCHEDULE = [60, 300, 900, 3600, 14400]`
```javascript
const nextRetryAt = new Date(Date.now() + BACKOFF_SCHEDULE[attempt_count - 1] * 1000);
```

**Payload structure:**
```json
{
  "event": "appointment.created",
  "delivery_id": "del_...",
  "timestamp": "2025-01-15T14:00:00.000Z",
  "data": {
    "appointment_id": "...",
    "host_id": "...",
    "guest_name": "...",
    "guest_email": "...",
    "start_time": "...",
    "end_time": "...",
    "status": "pending"
  }
}
```

**Signing:**
```javascript
const crypto = require('crypto');
const signature = crypto.createHmac('sha256', subscription.secret)
  .update(JSON.stringify(payload))
  .digest('hex');
// Header: X-MeetSync-Signature: sha256={signature}
```

### 8.6 How Rate Limiting Works

Redis-based sliding window counter per IP per route:

```javascript
export const rateLimiter = (maxRequests = 100, windowSeconds = 60, prefix = 'global') => {
  return async (req, res, next) => {
    const key = `rate_limit:${prefix}:${req.ip}`;
    const current = await redisClient.incr(key);
    if (current === 1) {
      await redisClient.expire(key, windowSeconds);
    }
    if (current > maxRequests) {
      return res.status(429).json({ error: 'Too many requests, please try again later' });
    }
    next();
  };
};
```

Route-specific limits:
- Global: 100 requests per 60 seconds per IP
- `GET /booking-links/:slug/slots`: 30 per 60 seconds per IP (prefix: `slots`)
- `POST /booking-links/:slug/book`: 10 per 60 seconds per IP (prefix: `book`)

### 8.7 How Soft-Delete Affects Queries

All models use the `softDeletePlugin` which adds:
- `deleted_at: { type: Date, default: null }` field
- `Model.findActive(filter)` => `Model.find({ ...filter, deleted_at: null })`
- `Model.findOneActive(filter)` => `Model.findOne({ ...filter, deleted_at: null })`
- `Model.countActive(filter)` => `Model.countDocuments({ ...filter, deleted_at: null })`

**Rule:** Every query that returns data to users or is used in business logic MUST use `findActive` or `findOneActive`. The only exception is admin/debug endpoints that explicitly need to see deleted records.

**Common bugs (intentionally planted):**
- Using `find()` instead of `findActive()` in slot calculation => deleted appointments block slots
- Using `findOne()` instead of `findOneActive()` in availability lookups => deleted rules still apply
- Not adding `deleted_at: null` in aggregation pipelines

---

## 9. Swagger / API Documentation

### Setup

Use `swagger-jsdoc` with inline OpenAPI 3.0 annotations in route files, and `swagger-ui-express` to serve the docs at `/api-docs`.

**Configuration (`src/config/swagger.js`):**

```javascript
const swaggerDefinition = {
  openapi: '3.0.0',
  info: {
    title: 'MeetSync API',
    version: '1.0.0',
    description: 'Appointment Scheduling Platform API -- manage availability, booking links, appointments, and webhooks.',
  },
  servers: [{ url: '/api/v1', description: 'API v1' }],
  components: {
    securitySchemes: {
      bearerAuth: { type: 'http', scheme: 'bearer', bearerFormat: 'JWT' },
    },
    schemas: {
      Error: { type: 'object', properties: { error: { type: 'string' } } },
      User: { ... },
      AvailabilityRule: { ... },
      AvailabilityOverride: { ... },
      BookingLink: { ... },
      Appointment: { ... },
      WebhookSubscription: { ... },
      WebhookDelivery: { ... },
      Pagination: {
        type: 'object',
        properties: {
          page: { type: 'integer' },
          limit: { type: 'integer' },
          total: { type: 'integer' },
          totalPages: { type: 'integer' },
          hasNextPage: { type: 'boolean' },
          hasPrevPage: { type: 'boolean' },
        },
      },
    },
  },
};
```

All model schemas should be fully documented with types, enums, required fields, and examples. Every endpoint should have JSDoc `@swagger` annotations documenting:
- Summary and description
- Parameters (path, query)
- Request body schema
- Response schemas (200, 201, 400, 401, 404, 409, 429)
- Security requirements

The Swagger UI is the candidate's primary API reference during the assessment. It must be comprehensive.

---

## 10. Seed Data

The `src/seed.js` script populates the database with realistic test data.

### Users (4)

```javascript
const users = await User.create([
  { name: 'Admin User', email: 'admin@meetsync.com', password: 'password123', role: 'admin', timezone: 'UTC' },
  { name: 'Dr. Sarah Chen', email: 'sarah@meetsync.com', password: 'password123', role: 'host', timezone: 'America/New_York' },
  { name: 'Mark Thompson', email: 'mark@meetsync.com', password: 'password123', role: 'host', timezone: 'America/Chicago' },
  { name: 'Emily Davis', email: 'emily@meetsync.com', password: 'password123', role: 'host', timezone: 'Europe/London' },
]);
```

### Availability Rules (for each host)

```javascript
// Sarah: Mon-Fri 9am-5pm Eastern
for (let day = 1; day <= 5; day++) {
  await AvailabilityRule.create({
    host_id: sarah._id,
    day_of_week: day,
    start_time: '09:00',
    end_time: '17:00',
    timezone: 'America/New_York',
  });
}

// Mark: Mon-Wed 10am-6pm, Thu 10am-2pm Central
for (let day = 1; day <= 3; day++) {
  await AvailabilityRule.create({
    host_id: mark._id,
    day_of_week: day,
    start_time: '10:00',
    end_time: '18:00',
    timezone: 'America/Chicago',
  });
}
await AvailabilityRule.create({
  host_id: mark._id,
  day_of_week: 4,
  start_time: '10:00',
  end_time: '14:00',
  timezone: 'America/Chicago',
});

// Emily: Mon-Fri 8am-4pm London
for (let day = 1; day <= 5; day++) {
  await AvailabilityRule.create({
    host_id: emily._id,
    day_of_week: day,
    start_time: '08:00',
    end_time: '16:00',
    timezone: 'Europe/London',
  });
}
```

### Availability Overrides (2-3 sample)

```javascript
// Sarah is unavailable on a specific future date (holiday)
const inTwoWeeks = futureDate(14).toISOString().split('T')[0];
await AvailabilityOverride.create({
  host_id: sarah._id,
  date: inTwoWeeks,
  is_available: false,
  timezone: 'America/New_York',
  reason: 'Personal day',
});

// Mark has limited hours on a specific date
const inThreeWeeks = futureDate(21).toISOString().split('T')[0];
await AvailabilityOverride.create({
  host_id: mark._id,
  date: inThreeWeeks,
  is_available: true,
  start_time: '10:00',
  end_time: '12:00',
  timezone: 'America/Chicago',
  reason: 'Half day',
});
```

### Booking Links (3-4 sample)

```javascript
const bookingLinks = await BookingLink.create([
  {
    host_id: sarah._id,
    slug: '30-min-consultation',
    title: '30 Minute Consultation',
    description: 'A quick consultation call to discuss your needs.',
    duration_minutes: 30,
    buffer_before_minutes: 5,
    buffer_after_minutes: 10,
    max_bookings_per_day: 8,
    min_notice_hours: 24,
    max_advance_days: 60,
    status: 'active',
  },
  {
    host_id: sarah._id,
    slug: '60-min-deep-dive',
    title: '60 Minute Deep Dive',
    description: 'An in-depth session for complex topics.',
    duration_minutes: 60,
    buffer_before_minutes: 10,
    buffer_after_minutes: 15,
    max_bookings_per_day: 4,
    min_notice_hours: 48,
    max_advance_days: 30,
    status: 'active',
  },
  {
    host_id: mark._id,
    slug: 'quick-chat',
    title: '15 Minute Quick Chat',
    duration_minutes: 15,
    buffer_before_minutes: 0,
    buffer_after_minutes: 5,
    max_bookings_per_day: null,
    min_notice_hours: 4,
    max_advance_days: 14,
    status: 'active',
  },
  {
    host_id: emily._id,
    slug: 'intro-meeting',
    title: 'Introduction Meeting',
    duration_minutes: 45,
    buffer_before_minutes: 5,
    buffer_after_minutes: 5,
    max_bookings_per_day: 6,
    min_notice_hours: 24,
    max_advance_days: 60,
    status: 'active',
  },
]);
```

### Sample Appointments (4-6)

```javascript
const inOneWeek = new Date(Date.now() + 7 * 24 * 60 * 60 * 1000);
// Set to next Monday at 10am Eastern
// ... (compute next Monday)

const appointments = await Appointment.create([
  {
    host_id: sarah._id,
    booking_link_id: bookingLinks[0]._id,
    guest_name: 'John Guest',
    guest_email: 'john@example.com',
    start_time: nextMonday10amUTC,
    end_time: nextMonday1030amUTC,
    duration_minutes: 30,
    timezone: 'America/New_York',
    status: 'confirmed',
    idempotency_key: 'seed_appt_1',
  },
  {
    host_id: sarah._id,
    booking_link_id: bookingLinks[0]._id,
    guest_name: 'Lisa Guest',
    guest_email: 'lisa@example.com',
    start_time: nextMonday11amUTC,
    end_time: nextMonday1130amUTC,
    duration_minutes: 30,
    timezone: 'America/New_York',
    status: 'pending',
    idempotency_key: 'seed_appt_2',
  },
  {
    host_id: mark._id,
    booking_link_id: bookingLinks[2]._id,
    guest_name: 'Bob Guest',
    guest_email: 'bob@example.com',
    start_time: nextTuesday2pmUTC,
    end_time: nextTuesday215pmUTC,
    duration_minutes: 15,
    timezone: 'America/Chicago',
    status: 'confirmed',
    idempotency_key: 'seed_appt_3',
  },
]);
```

### Webhook Subscriptions (1-2 sample)

```javascript
await WebhookSubscription.create([
  {
    host_id: sarah._id,
    url: 'https://httpbin.org/post',
    events: ['appointment.created', 'appointment.confirmed', 'appointment.cancelled'],
    secret: 'whsec_test_secret_sarah',
    status: 'active',
  },
  {
    host_id: mark._id,
    url: 'https://httpbin.org/post',
    events: ['appointment.created'],
    secret: 'whsec_test_secret_mark',
    status: 'active',
  },
]);
```

---

## Summary: Task Matrix

| Task | Name | Type | Difficulty | Time | Files Affected | Key Pattern |
|---|---|---|---|---|---|---|
| 1 | Webhook Delivery with Retry | Feature | Hard | 60m | pkg/features/webhook_delivery/* | Async patterns, retry, HMAC signing |
| 2 | Available Slot Calculation | Feature | Medium | 45m | pkg/features/appointment_slots/* | Time math, conflict detection, caching |
| 3 | Appointment Rescheduling | Feature | Hard | 60m | pkg/features/appointment_reschedule/* | Conflict detection, cache invalidation, state validation |
| 4 | Paginated Appointment History | Feature | Easy | 30m | pkg/features/appointment_history/* | Pagination, filtering, query building |
| 5 | Appointment State Machine Hooks | Bug | Medium | 45m | src/models/Appointment.js | Pre-save hooks, state machine, time validation |
| 6 | Booking Link Rate Limiting | Bug | Easy | 30m | src/routes/bookingLink.routes.js, src/middleware/rateLimiter.js | Middleware, Redis rate limiting |
| 7 | Availability Cache Invalidation | Bug | Medium | 40m | src/services/appointment.service.js, src/services/availability.service.js | Cache invalidation strategy |
| 8 | Double-Booking Prevention | Bug | Hard | 45m | src/services/appointment.service.js | Idempotency, conflict detection, race conditions |
| 9 | Soft-Delete Leaking into Slots | Bug | Easy | 30m | src/services/availability.service.js | Soft-delete awareness, query correctness |

**Difficulty distribution:**
- Features: 1 Easy, 1 Medium, 2 Hard
- Bugs: 2 Easy, 2 Medium, 1 Hard

**Total estimated time:** ~385 minutes (6.4 hours). Candidates are expected to complete a subset; scoring is per-test-case (like ShowPass with 100 test cases total: approximately 8-12 tests per task across 9 tasks = ~100 tests).

---

### Critical Files for Implementation

- `C:/Users/RajkumarM/Desktop/HackerRank/ShowPass/MERN/src/models/Event.js` - Reference pattern for state machine with pre-save hooks, post-init status tracking, and soft-delete plugin usage
- `C:/Users/RajkumarM/Desktop/HackerRank/ShowPass/MERN/src/services/event.service.js` - Reference pattern for service layer with cache integration, query building, pagination, and conflict detection
- `C:/Users/RajkumarM/Desktop/HackerRank/ShowPass/MERN/src/pkg/features/ticket_transfer/tests/integration_test.js` - Reference pattern for feature test structure: mocha/chai setup, test isolation, DB verification, factory fixtures
- `C:/Users/RajkumarM/Desktop/HackerRank/ShowPass/MERN/test/task5/app.spec.js` - Reference pattern for bug test structure: direct model testing, error message assertions, hook bypass verification
- `C:/Users/RajkumarM/Desktop/HackerRank/ShowPass/MERN/hackerrank.yml` - Reference for HackerRank platform configuration: readonly paths, IDE defaults, test commands