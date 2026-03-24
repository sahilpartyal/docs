---
title: "Quote Flow"
description: "Quote request, pricing, customer selection and confirmation."
---

# Quote Request Flow

Customer requests pricing for multiple vehicles. Admin prices each option. Customer selects and pays.

## Actors

- **Customer** — requests quote, selects vehicle, pays
- **Operator Admin** — prices vehicles, sends quote

## Entry Points

| Channel | URL | Controller |
|---------|-----|------------|
| SPA | `POST /api/v1/bookings/quote` | `Api\V1\BookingController::storeQuote()` |
| Widget | `POST /api/v1/widget/quotes` | `Widget\WidgetApiController::createQuote()` |
| Embed | `POST /api/v1/embed/quotes` | `Embed\EmbedController::createQuote()` |

## Flow Diagram

```mermaid
sequenceDiagram
    participant C as Customer
    participant API as Booking API
    participant Admin as Operator Admin
    participant Email as Email Service
    participant Stripe as Stripe

    C->>API: POST /bookings/quote (addresses, vehicle_type_ids[])
    API->>API: BookingService::createQuote()
    API-->>C: status: quote_requested
    API->>Email: Notify admin of new quote

    Admin->>API: PATCH /admin/quotes/{id}/pricing (price each vehicle)
    Admin->>API: PATCH /admin/bookings/{id} (action: send_quote)
    API->>Email: Send quote options to customer
    API-->>Admin: status: quote_sent

    C->>API: POST /bookings/{id}/select-vehicle (quote_vehicle_id)
    API->>API: Copy pricing to booking
    API-->>C: status: payment_pending

    C->>API: POST /payments/{ref}/create-intent
    API->>Stripe: createPaymentIntent()
    C->>Stripe: Confirm payment
    Stripe->>API: Webhook: payment_intent.succeeded
    API-->>C: status: awaiting_approval

    Admin->>API: Approve booking
    API-->>C: status: confirmed
```

## Status Progression

```mermaid
stateDiagram-v2
    [*] --> quote_requested: Customer requests quote
    quote_requested --> quote_sent: Admin prices & sends
    quote_sent --> payment_pending: Customer selects vehicle
    payment_pending --> awaiting_approval: Payment succeeded
    awaiting_approval --> confirmed: Admin approves
    confirmed --> driver_assigned: Driver assigned
    driver_assigned --> completed: Trip completed
```

## Step-by-Step

### 1. Customer Requests Quote

```
POST /api/v1/bookings/quote
```

| Field | Required | Description |
|-------|----------|-------------|
| `vehicle_type_ids` | Yes | Array of vehicle type IDs to compare (2-4 typical) |
| All trip fields | Yes | Same as direct booking (addresses, datetime, passengers) |

**Service:** `BookingService::createQuote($data, $user)`
- Creates booking with `booking_type = 'quote'`, `status = 'quote_requested'`
- Creates `QuoteVehicle` record per vehicle (pricing = 0, placeholder)
- Records `QUOTE_REQUESTED` event

### 2. Admin Prices Each Vehicle

```
PATCH /api/v1/admin/quotes/{booking_id}/pricing
```

| Field | Description |
|-------|-------------|
| `vehicles[].quote_vehicle_id` | ID of quote vehicle |
| `vehicles[].base_fare` | Base price |
| `vehicles[].distance_fare` | Distance charge |
| `vehicles[].time_fare` | Time charge |
| `vehicles[].extras_fare` | Extras charge |
| `vehicles[].total_amount` | Total for this vehicle |

### 3. Admin Sends Quote

```
PATCH /api/v1/admin/bookings/{id}  (action: send_quote)
```

- Validates at least one vehicle is priced
- Transitions: `quote_requested` → `quote_sent`
- Sends email to customer with pricing options

### 4. Customer Selects Vehicle

```
POST /api/v1/bookings/{id}/select-vehicle
```

| Field | Required | Description |
|-------|----------|-------------|
| `quote_vehicle_id` | Yes | ID of selected QuoteVehicle |

**Process:**
- Locks booking (`lockForUpdate`)
- Marks QuoteVehicle as selected
- Copies pricing into booking fields
- Transitions: `quote_sent` → `payment_pending`

### 5. Payment

Same as [Public Booking payment flow](booking-public.md#3-payment-if-card).

## Events Fired

| Event Type | When |
|------------|------|
| `QUOTE_REQUESTED` | Quote created |
| `QUOTE_SENT` | Admin sends to customer |
| `QUOTE_VEHICLE_SELECTED` | Customer picks vehicle |
| `PAYMENT_SUCCEEDED` | Payment confirmed |

## Key Files

| Purpose | File |
|---------|------|
| Quote creation | `app/Booking/Services/BookingService.php` → `createQuote()` |
| Quote pricing | `app/Http/Controllers/Api/Admin/QuoteController.php` |
| Vehicle selection | `app/Http/Controllers/Api/V1/BookingController.php` → `selectVehicle()` |
| QuoteVehicle model | `app/Booking/Models/QuoteVehicle.php` |
