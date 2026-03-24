---
title: "Widget Booking Flow"
description: "Embeddable widget booking flow with auth and payment."
---

# Widget/Embed Booking Flow

Booking or quote request submitted through the Preact micro-frontend widget or legacy Blade embed widgets, embedded on third-party websites via `<here2there-booking>` custom element.

## Actors

- **Visitor** — unauthenticated user on a third-party website
- **Authenticated Customer** — logged-in widget user (optional)
- **Operator Admin** — receives quote requests, approves bookings

## Entry Points

| Channel | URL | Controller |
|---------|-----|------------|
| Widget config | `GET /api/v1/widget/config/{identifier}` | `Widget\WidgetApiController::config()` |
| Create session | `POST /api/v1/widget/sessions` | `Widget\WidgetApiController::createSession()` |
| Update session | `PATCH /api/v1/widget/sessions/{token}` | `Widget\WidgetApiController::updateSession()` |
| Calculate pricing | `POST /api/v1/widget/pricing` | `Widget\WidgetApiController::pricing()` |
| Create booking | `POST /api/v1/widget/bookings` | `Widget\WidgetApiController::createBooking()` |
| Create quote | `POST /api/v1/widget/quotes` | `Widget\WidgetApiController::createQuote()` |
| Create PaymentIntent | `POST /api/v1/widget/payment-intent` | `Widget\WidgetApiController::createPaymentIntent()` |
| Validate promo | `POST /api/v1/widget/promo-code/validate` | `Widget\WidgetApiController::validatePromoCode()` |
| Login | `POST /api/v1/widget/auth/login` | `Widget\WidgetAuthController::login()` |
| Register | `POST /api/v1/widget/auth/register` | `Widget\WidgetAuthController::register()` |
| Dashboard | `GET /api/v1/widget/dashboard` | `Widget\WidgetDashboardController::index()` |
| Saved cards | `GET /api/v1/widget/saved-cards` | `Widget\WidgetSavedCardController::index()` |

## Session Lifecycle

```mermaid
stateDiagram-v2
    [*] --> in_progress: POST /sessions (short_code)
    in_progress --> in_progress: PATCH /sessions/{token}
    in_progress --> in_progress: POST /pricing
    in_progress --> completed: POST /bookings or /quotes
    in_progress --> expired: Session TTL exceeded
    completed --> [*]
    expired --> [*]
```

## Flow Diagram — Direct Booking Path

```mermaid
sequenceDiagram
    participant W as Widget (Preact)
    participant API as Widget API
    participant ES as EmbedService
    participant GM as Google Maps
    participant PS as PricingService
    participant Stripe as Stripe
    participant Admin as Operator Admin

    W->>API: GET /config/{slug|short_code}
    API-->>W: operator, theme, vehicles, stripe_key, config flags

    W->>API: POST /sessions (short_code, trip details)
    API->>ES: createSession()
    ES-->>API: session_token, expires_at
    API-->>W: session_token

    W->>API: PATCH /sessions/{token} (update addresses, datetime)
    API->>ES: updateSession()
    API-->>W: updated session

    W->>API: POST /pricing (session_token)
    API->>ES: calculatePricing()
    ES->>GM: getDistance() or getDirections() with waypoints
    GM-->>ES: distance_km, duration_minutes
    ES->>PS: calculateAdvanced() per vehicle type
    PS-->>ES: pricing breakdown per vehicle
    ES-->>API: vehicles with prices or needs_quote flags
    API-->>W: vehicles array (total or needs_quote per vehicle)

    Note over W: Vehicle shows "Book Now" (has price) or "Get Quote" (needs_quote)

    alt Direct Booking (has price)
        W->>API: POST /payment-intent (session_token, vehicle_type_id)
        API->>Stripe: PaymentIntent.create() with operator keys
        Stripe-->>API: client_secret
        API-->>W: client_secret, amount

        Note over W: Stripe Elements in light DOM (Shadow DOM workaround)
        W->>Stripe: confirmCardPayment(client_secret)
        Stripe-->>W: payment confirmed

        W->>API: POST /bookings (session_token, vehicle_type_id, customer, payment_method)
        API->>ES: createDirectBooking()
        ES->>ES: findOrCreateCustomer()
        ES->>ES: Create Booking (status: payment_pending)
        ES->>ES: Create Payment record
        ES->>ES: Record EMBED_BOOKING_CREATED event
        ES->>ES: session.markCompleted()
        API-->>W: booking_id, reference_number, status

        Stripe->>API: Webhook: payment_intent.succeeded
        API->>API: PaymentService::handlePaymentSuccess()
        API->>API: status → awaiting_approval, payment_status → paid
    else Invoice Payment
        W->>API: POST /bookings (payment_method: invoice)
        API->>ES: createDirectBooking()
        ES->>ES: Create Booking (status: awaiting_approval)
        ES->>ES: InvoiceManagementService::createAndSendFromBooking()
        API-->>W: booking_id, reference_number
    end
```

## Flow Diagram — Quote Path

```mermaid
sequenceDiagram
    participant W as Widget (Preact)
    participant API as Widget API
    participant ES as EmbedService
    participant Admin as Operator Admin

    Note over W: Vehicle shows "Get Quote" (needs_quote or force_quote_only)

    W->>API: POST /quotes (session_token, vehicle_type_ids[], customer)
    API->>ES: createQuoteRequest()
    ES->>ES: findOrCreateCustomer()
    ES->>ES: Create Booking (type: quote, status: quote_requested)
    ES->>ES: Create QuoteVehicle per vehicle_type_id
    ES->>ES: Record EMBED_QUOTE_REQUESTED event
    ES->>ES: session.markCompleted()
    ES->>ES: NotificationService::sendQuoteRequestNotification()
    API-->>W: quote_id, message

    Note over Admin: Quote follows standard quote flow from here
    Note over Admin: See quote-flow.md for pricing and vehicle selection
```

## Widget Auth (Login/Register within Widget)

```mermaid
sequenceDiagram
    participant W as Widget
    participant API as Widget Auth API

    alt Register
        W->>API: POST /auth/register (name, email, phone, password, operator_id)
        API->>API: Check email unique per operator
        API->>API: Create User (role: customer, operator_id)
        API->>API: Create OperatorCustomer pivot (source: widget)
        API->>API: createToken('widget', ['widget:read', 'widget:write'])
        API-->>W: token, user
        Note over W: Store token in localStorage (key: h2t_token)
    else Login
        W->>API: POST /auth/login (email, password, operator_id)
        API->>API: Find User by email + operator_id + role=customer
        API->>API: Verify password, check is_active
        API->>API: createToken('widget', ['widget:read', 'widget:write'])
        API-->>W: token, user
    end

    Note over W: Subsequent requests include Bearer token header
    W->>API: GET /dashboard (Authorization: Bearer {token})
    API-->>W: stats, upcoming_bookings
```

## Stripe Elements in Shadow DOM

Stripe Elements cannot mount inside Shadow DOM. The widget uses a **light DOM slot pattern**:

1. Widget creates `<div slot="stripe-card-xxx">` in the **host element's light DOM**
2. Shadow DOM template contains `<slot name="stripe-card-xxx" />`
3. Stripe Elements mount on the light DOM container
4. Browser projects the element into the shadow DOM slot visually
5. On unmount: remove light DOM element + `cardElement.destroy()`

## Hybrid Mode

Each vehicle independently shows **"Book Now"** or **"Get Quote"** based on pricing availability:

- `total > 0` and `needs_quote: false` — "Book Now" (direct booking path)
- `needs_quote: true` — "Get Quote" (quote path)
- `force_quote_only: true` on config — all vehicles forced to quote path

## Widget Versions

| Version | Technology | `widget_version` value |
|---------|-----------|----------------------|
| Blade V1 | Alpine.js + Tailwind (iframe) | `blade_v1` |
| Blade V2 | Alpine.js + Tailwind (iframe) | `blade_v2` |
| Preact V1 | Preact web component (Shadow DOM) | `preact_v1` |

All versions share `EmbedService` for backend logic.

## Rate Limiting

| Tier | Limit | Scope |
|------|-------|-------|
| `widget-config` | 120/min | Config lookups |
| `widget-session` | 30/min | Session operations |
| `widget-submit` | 5/min | Booking/quote submissions |

## CORS

`WidgetCors` middleware registered as **global middleware before HandleCors** in Kernel.php. Only affects `api/v1/widget/*` routes. Route middleware alone does not work because Laravel's global `HandleCors` intercepts OPTIONS preflight before route middleware runs.

## Events Fired

| Event Type | When |
|------------|------|
| `EMBED_BOOKING_CREATED` | Direct booking saved |
| `EMBED_QUOTE_REQUESTED` | Quote request saved |
| `PAYMENT_SUCCEEDED` | Stripe webhook confirms card payment |

## Key Files

| Purpose | File |
|---------|------|
| Widget API controller | `app/Http/Controllers/Api/V1/Widget/WidgetApiController.php` |
| Widget auth controller | `app/Http/Controllers/Api/V1/Widget/WidgetAuthController.php` |
| Widget dashboard controller | `app/Http/Controllers/Api/V1/Widget/WidgetDashboardController.php` |
| Widget saved cards controller | `app/Http/Controllers/Api/V1/Widget/WidgetSavedCardController.php` |
| Embed service | `app/Embed/Services/EmbedService.php` |
| Widget provisioning | `app/Embed/Services/WidgetProvisioningService.php` |
| Embed session model | `app/Embed/Models/EmbedSession.php` |
| Embed config model | `app/Embed/Models/EmbedConfiguration.php` |
| CORS middleware | `app/Http/Middleware/WidgetCors.php` |
| Preact widget source | `widget/` |
