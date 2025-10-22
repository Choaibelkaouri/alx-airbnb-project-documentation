# Airbnb Clone Backend — Features and Functionalities

## Actors
- Guest  
- Host  
- Admin  
- Payment Provider (Stripe/PayPal)  
- External Services (Email/SMS, Maps, Storage)

---

## Authentication & Authorization
- Register and login with JWT + refresh tokens  
- Forgot/reset password  
- Roles: GUEST, HOST, ADMIN  
- Route protection and ownership checks  
- Optional: OAuth2/Google login

---

## User Profile
- View and update profile, photo, and phone number  
- Manage notification, language, and timezone preferences

---

## Property Management
- Create, read, update, delete properties  
- Include geolocation, multiple images, amenities, and house rules  
- Pricing with cleaning fee and currency support  
- Availability calendar  
- Cancellation policy  
- Instant booking or host approval options

---

## Search & Filters
- Text-based and location-based search  
- Filters: price, rating, type, capacity, and availability  
- Sorting, pagination, and caching for common searches

---

## Booking System
- Check property availability and prevent conflicts  
- Create booking with statuses (PENDING / CONFIRMED / CANCELED)  
- Payment hold or TTL for pending bookings  
- Cancellation policies for guests and hosts  
- Generate booking invoices or summaries

---

## Payments
- Stripe integration: Payment Intent, Confirm, Refund, and Webhooks  
- Platform fee and host payout using Stripe Connect  
- Multi-currency support (with default currency)  
- Payment reconciliation and reporting summaries

---

## Reviews
- Guest reviews for properties (star rating and text comments)  
- Report abuse and calculate average property rating

---

## Wishlists
- Create, view, and manage personalized wishlists  
- Add or remove properties from lists

---

## Messaging & Notifications
- Basic host ↔ guest messaging system  
- Email or push notifications for key events (booking, confirmation, cancellation)  
- User preferences for notification channels

---

## Admin
- Manage users, properties, and bookings  
- Handle moderation, content reports, and complaints  
- Generate basic statistics and performance reports

---

## Data Model (Summary)
Entities include:  
- **User**  
- **Property**  
- **PropertyImage**  
- **Availability**  
- **Booking**  
- **Payment**  
- **Review**  
- **Message**  
- **Wishlist**  
- **WishlistItem**

---

## REST API Examples
- `POST /auth/register`  
- `POST /auth/login`  
- `POST /auth/refresh`  
- `GET /users/me`  
- `PATCH /users/me`  
- `POST /properties`  
- `GET /properties`  
- `GET /properties/{id}`  
- `PATCH /properties/{id}`  
- `DELETE /properties/{id}`  
- `POST /bookings`  
- `GET /bookings/{id}`  
- `POST /bookings/{id}/cancel`  
- `POST /payments/intent`  
- `POST /payments/{bookingId}/confirm`  
- `POST /payments/{bookingId}/refund`  
- `POST /webhooks/stripe`  
- `POST /properties/{id}/reviews`  
- `GET /properties/{id}/reviews`  
- `POST /wishlists`

---

## Diagram
Refer to the diagram file: `airbnb-clone-features.png` (exported from Draw.io)

---

## Checklist
- `features-and-functionalities/README.md`  
- `features-and-functionalities/airbnb-clone-features.png`  
- (Optional) `features-and-functionalities/airbnb-clone-features.drawio`
