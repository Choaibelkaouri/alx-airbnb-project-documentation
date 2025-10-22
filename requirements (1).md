
# Backend Requirement Specifications — Airbnb Clone

**Scope:** This document specifies functional and technical requirements for three core backend features: **User Authentication**, **Property Management**, and **Booking System**. It also covers cross‑cutting concerns (validation, security, performance, and error handling). The API is REST-first and JSON-based.

---

## 0. Conventions

- **Base URL:** `/api/v1`
- **Auth:** Bearer JWT in `Authorization: Bearer <token>` unless noted.
- **Content-Type:** `application/json`
- **Timestamps:** ISO 8601 in UTC (e.g., `2025-10-22T16:30:00Z`)
- **IDs:** UUIDv4 strings unless specified.
- **Pagination:** `?page=<int>&limit=<1..100>`; responses include `meta: { page, limit, total }`.
- **Sorting:** `?sort=<field>:<asc|desc>` e.g. `?sort=price:asc`.
- **Filtering:** Explicit query params (see endpoints).

### Error Model
```json
{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Email is invalid",
    "details": [
      {"field": "email", "rule": "format"}
    ],
    "trace_id": "b1e5..."
  }
}
```
- **Common codes:** `UNAUTHORIZED`, `FORBIDDEN`, `NOT_FOUND`, `CONFLICT`, `RATE_LIMITED`, `INTERNAL_ERROR`

---

## 1) User Authentication

### Goals
- Secure registration and login with refresh tokens
- Password reset and email verification (optional)
- Role-based access (GUEST, HOST, ADMIN)

### Data Model (minimum)
- **User**: `id`, `name`, `email (unique)`, `password_hash`, `role`, `phone?`, `email_verified_at?`, `created_at`, `updated_at`
- **RefreshToken**: `id`, `user_id`, `token_hash`, `expires_at`, `revoked_at?`, `ip?`, `ua?`

### Endpoints

#### POST `/auth/register`
**Input**
```json
{ "name": "John Doe", "email": "john@example.com", "password": "StrongP@ssw0rd!", "role": "GUEST" }
```
**Validation**
- `name` 2..80 chars
- `email` RFC 5322 format; unique
- `password` ≥ 8 chars, 1 upper, 1 lower, 1 digit; hash with bcrypt (>= cost 10)
- `role` ∈ {GUEST, HOST} (ADMIN only via admin tooling)
**Output** `201`
```json
{ "id": "uuid", "name": "John Doe", "email": "john@example.com", "role": "GUEST", "created_at": "..." }
```

#### POST `/auth/login`
**Input**
```json
{ "email": "john@example.com", "password": "StrongP@ssw0rd!" }
```
**Output** `200`
```json
{
  "access_token": "jwt_access",
  "expires_in": 3600,
  "refresh_token": "jwt_refresh",
  "token_type": "Bearer",
  "user": { "id": "uuid", "name": "John Doe", "email": "john@example.com", "role": "GUEST" }
}
```
**Errors:** `UNAUTHORIZED` if credentials invalid; backoff after 5 failed attempts (lock for 15 min).

#### POST `/auth/refresh`
**Input**
```json
{ "refresh_token": "jwt_refresh" }
```
**Output** `200` (new access & refresh pair). Revoke old refresh.

#### POST `/auth/logout`
Revokes current access and associated refresh token. Returns `204`.

#### POST `/auth/forgot-password`
Input: `{ "email": "john@example.com" }` → send one-time token (15 min).
Output `204`.

#### POST `/auth/reset-password`
Input: `{ "token": "xxx", "new_password": "StrongP@ssw0rd!" }` → `204`.

#### GET `/users/me` (Auth)
Output: user profile.
#### PATCH `/users/me` (Auth)
Editable fields: `name`, `phone`. Not `email` or `role` (separate flows).

### Performance & Security
- Access JWT TTL: 15–60 min; Refresh TTL: 7–30 days, rotating refresh
- Store password with bcrypt/argon2; never log secrets
- Rate limit: 10 req/min per IP for `/auth/*` (burst 20)
- Brute-force protection: incremental delay; IP + account lock window
- Email verification (optional): block bookings until verified

---

## 2) Property Management

### Goals
- Hosts can CRUD properties with images, amenities, rules, pricing, availability
- Guests can search/filter & view

### Data Model (minimum)
- **Property**: `id`, `host_id`, `title`, `description`, `type`, `address`, `city`, `country`, `lat`, `lng`, `base_price`, `cleaning_fee?`, `currency`, `max_guests`, `instant_book` (bool), `created_at`, `updated_at`
- **PropertyImage**: `id`, `property_id`, `url`, `is_cover`
- **Availability**: `id`, `property_id`, `date`, `is_available`, `price_override?`

### Endpoints

#### POST `/properties` (Auth: HOST)
**Input (min)**
```json
{
  "title": "Cozy Loft",
  "description": "Near downtown",
  "type": "APARTMENT",
  "address": "123 King St",
  "city": "Rabat",
  "country": "MA",
  "lat": 34.02,
  "lng": -6.83,
  "base_price": 85.00,
  "currency": "MAD",
  "max_guests": 3,
  "instant_book": false
}
```
**Validation**
- `title` 3..100; `description` ≤ 3000
- `type` ∈ {APARTMENT, HOUSE, ROOM, OTHER}
- `base_price` ≥ 0; `currency` ISO 4217; `max_guests` 1..20
- (lat,lng) within valid ranges
**Output** `201` → created property

#### GET `/properties`
Query params:
- `q` (text), `city`, `country`, `min_price`, `max_price`, `type`, `guests`, `start_date`, `end_date`, `instant_book` (bool), `sort`, `page`, `limit`
- Must exclude properties not available for the full `[start_date, end_date)` window.
**Output** `200` (list + `meta`).

#### GET `/properties/{id}` → property details + images + rating avg

#### PATCH `/properties/{id}` (Auth: HOST owner or ADMIN)
- Partial update; server enforces ownership.

#### DELETE `/properties/{id}` (Auth: HOST owner or ADMIN)
- If future confirmed bookings exist → `CONFLICT` unless admin override.

#### POST `/properties/{id}/images` (Auth: HOST owner)
Multipart or JSON with already-uploaded object storage URLs.
- Limit: ≤ 20 images per property; max 10MB each; MIME check.
- Returns array of images.

#### PUT `/properties/{id}/availability` (Auth: HOST owner)
**Input**
```json
{
  "slots": [
    { "date": "2025-11-01", "is_available": false },
    { "date": "2025-11-02", "is_available": true, "price_override": 99.00 }
  ]
}
```
- Server must prevent overlapping/conflicting rules (one record per date).

### Performance
- Indexes: `(city,country)`, `lat/lng` (geospatial), `base_price`, `host_id`, `(property_id,date)`
- Caching: Search results for common filters (60–120s) + CDN on images
- Pagination required; max 100 per page

---

## 3) Booking System

### Goals
- Guests request bookings; system checks availability; holds inventory; confirms on payment
- Hosts can approve/decline if `instant_book=false`
- Prevent double booking and race conditions

### Data Model (minimum)
- **Booking**: `id`, `property_id`, `guest_id`, `start_date`, `end_date`, `guests`, `status` ∈ {PENDING, AWAITING_PAYMENT, CONFIRMED, CANCELED, EXPIRED}, `total_amount`, `currency`, `created_at`, `updated_at`
- **Payment** (covered by Payments feature; referenced): `booking_id`, `status`, `amount`, etc.

### Workflow (high level)
1. Guest checks availability (search/detail).
2. Create booking (PENDING) + **temporary hold** on dates.
3. Compute price = nightly sum (override if any) + cleaning fee + fees.
4. If instant book → status `AWAITING_PAYMENT`; else `PENDING` → host action.
5. On successful payment → `CONFIRMED` and lock dates. On failure/timeout → `EXPIRED` or `CANCELED`.

### Endpoints

#### POST `/bookings` (Auth: GUEST)
**Input**
```json
{
  "property_id": "uuid",
  "start_date": "2025-12-03",
  "end_date": "2025-12-06",
  "guests": 2
}
```
**Validation**
- `start_date < end_date`; stay length 1..30 nights (configurable)
- `guests` ≤ `property.max_guests`
- Availability must be true for **each** night in `[start_date, end_date)`
- Atomic check + hold to avoid race conditions
**Output** `201`
```json
{
  "id": "uuid",
  "status": "AWAITING_PAYMENT",
  "total_amount": 255.00,
  "currency": "MAD"
}
```

#### GET `/bookings/{id}` (Auth: owner guest or host or ADMIN)
Returns booking + property snapshot + payment status if caller is involved.

#### POST `/bookings/{id}/cancel` (Auth: guest owner or host)
- Apply cancellation policy (window, fees).  
- If already `CONFIRMED`, may need refund flow (see payments).

#### POST `/bookings/{id}/approve` (Auth: HOST owner)
- Only if property `instant_book=false` and current status `PENDING` → sets `AWAITING_PAYMENT`.

#### POST `/bookings/{id}/decline` (Auth: HOST owner)
- Sets status `CANCELED`; releases holds.

### Availability & Concurrency
- Use **SELECT ... FOR UPDATE** or advisory locks around availability check + insert
- Unique constraint: `(property_id, date)` in Availability
- Background job to **expire** unpaid bookings after TTL (e.g., 15 min)

### Performance
- Index: `(guest_id)`, `(property_id, start_date, end_date)`, `(status)`
- Limit concurrent holds per user to mitigate abuse

---

## 4) (Reference) Payments Interface (brief)

> Full spec out of scope here, but booking relies on these behaviors.

- **POST `/payments/intent`** with `booking_id` → returns provider client secret
- **Webhook `/webhooks/stripe`** → updates `Payment.status` and `Booking.status`
- Idempotency-Key header for all payment actions
- Refund endpoint: `POST /payments/{bookingId}/refund` (host/admin constraints)

---

## 5) Validation Rules (summary)

- Strings trimmed; prohibit control chars; max lengths enforced
- Money fields stored in minor units (e.g., cents); validate `currency` (ISO 4217)
- Dates are calendar-valid; no past check-in; max stay length; timezone-agnostic (UTC)
- File uploads: allowlist MIME; max 10MB; virus scan hook (optional)
- Request body schema validated (e.g., JSON Schema / zod / Joi)

---

## 6) Security Requirements

- HTTPS only; HSTS; secure cookies if used
- JWT audience/issuer claims; rotate signing keys
- RBAC middleware; ownership checks for host/guest resources
- Protect IDOR: never trust client IDs without access checks
- Rate limit globally (e.g., 100 req/min/IP) + stricter for sensitive routes
- Audit log for privileged/admin actions
- Webhook verification (provider signature header)
- Secrets in env manager (not in repo)

---

## 7) Observability & Ops

- Structured logging (JSON); include `trace_id`
- Metrics: request latency, error rate, booking funnel, payment success rate
- Health check: `GET /healthz` returns db/connectivity probes
- Backups for primary database (daily + PITR if supported)
- CI: run unit + schema validation + migrations dry-run

---

## 8) Non‑Functional Requirements

- **Availability:** 99.5% for core APIs
- **Latency SLO (p95):** Auth < 200ms; Search < 500ms; Booking actions < 300ms (excluding provider roundtrips)
- **Scalability:** horizontal API nodes; read replicas for heavy read endpoints
- **Internationalization:** currency & locale-aware formatting on client; backend stores canonical values

---

## 9) Open Questions / Assumptions

- Payment provider: Stripe (default); PayPal optional later
- Email verification required before booking? (default: required)
- City search via full-text vs. geo-radius? (MVP: city + date window filter)
- Cancellation policy variants to be finalized

---

## 10) API Examples (Happy Paths)

### Register → Login → Create Property → Search → Book
```http
POST /api/v1/auth/register
POST /api/v1/auth/login
POST /api/v1/properties  (HOST)
GET  /api/v1/properties?q=loft&city=Rabat&start_date=2025-12-03&end_date=2025-12-06
POST /api/v1/bookings
```
