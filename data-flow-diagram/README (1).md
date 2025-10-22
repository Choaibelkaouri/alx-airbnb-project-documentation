# Data Flow Diagram (DFD) — Airbnb Clone Backend

This folder contains the Level-0/Level-1 style DFD showing how data moves between external entities, backend processes, and data stores.

## Scope (Core Flows)
- User Authentication (register/login)
- Property Management (CRUD, images, availability)
- Search and Booking
- Payments (via Payment Provider)
- Reviews & Notifications (basic)

## External Entities
- User (Guest/Host)
- Admin
- Payment Provider (Stripe/PayPal)
- Email/SMS Service

## Processes (examples)
1. Register/Login User
2. Manage Property
3. Search Properties
4. Create Booking
5. Process Payment
6. Manage Reviews
7. Send Notifications

## Data Stores
- D1: Users
- D2: Properties
- D3: Bookings
- D4: Payments
- D5: Reviews
- D6: Messages / Notifications
- D7: Media (images/objects)

## Files
- `data-flow.drawio` — editable DFD in Draw.io format
- `data-flow.png` — exported image
