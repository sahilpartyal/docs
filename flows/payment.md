---
title: "Payment Processing"
description: "Stripe PaymentIntent lifecycle, saved cards, refunds and webhooks."
---

# Payment Processing Flow

Stripe PaymentIntent lifecycle, saved cards CRUD, refunds, Stripe Connect for operators, and webhook handling with idempotency.

## Actors

- **Customer** — pays for bookings via card, views saved cards
- **Operator Admin** — processes refunds, views payment history
- **Stripe** — processes payments, sends webhooks
- **System** — handles webhooks, manages payment state

## Entry Points

| Channel | URL | Controller |
|---------|-----|------------|
| Create PaymentIntent | `POST /api/v1/payments/{reference}/create-intent` | `Api\V1\PaymentController::createIntent()` |
| Confirm payment | `POST /api/v1/payments/{reference}/confirm` | `Api\V1\PaymentController::confirm()` |
| Widget PaymentIntent | `POST /api/v1/widget/payment-intent` | `Widget\WidgetApiController::createPaymentIntent()` |
| Stripe webhook | `POST /api/webhooks/stripe` | `Api\WebhookController::handleStripe()` |
| Process refund | `POST /api/v1/admin/payments/{id}/refund` | `Api\Admin\PaymentController::refund()` |
| List saved cards | `GET /api/v1/saved-cards` | `Api\V1\SavedCardController::index()` |
| Add saved card | `POST /api/v1/saved-cards/setup-intent` | `Api\V1\SavedCardController::createSetupIntent()` |
| Delete saved card | `DELETE /api/v1/saved-cards/{paymentMethodId}` | `Api\V1\SavedCardController::destroy()` |

## Stripe PaymentIntent Lifecycle

```mermaid
stateDiagram-v2
    [*] --> requires_payment_method: PaymentIntent created
    requires_payment_method --> requires_confirmation: Card entered
    requires_confirmation --> requires_action: 3D Secure required
    requires_action --> processing: 3DS completed
    requires_confirmation --> processing: No 3DS needed
    processing --> succeeded: Payment captured
    processing --> canceled: Payment failed
    requires_payment_method --> canceled: Cancelled
    succeeded --> [*]
    canceled --> [*]
```

## Payment Flow — Customer Booking

```mermaid
sequenceDiagram
    participant C as Customer
    participant API as Payment API
    participant PS as PaymentService
    participant SS as StripeService
    participant Stripe as Stripe
    participant WH as Webhook

    C->>API: POST /payments/{reference}/create-intent
    API->>PS: createPaymentIntent(booking, options)
    PS->>PS: Validate booking.status = payment_pending
    PS->>PS: lockForUpdate() on booking (prevent concurrent)
    PS->>PS: Re-check status after lock

    PS->>SS: useKeysForOperator(operator)
    Note over SS: Per-operator Stripe key scoping (test/live mode)

    alt Existing active PaymentIntent
        PS->>SS: retrievePaymentIntent(existing_id)
        SS-->>PS: PaymentIntent still actionable
        PS-->>C: Return existing client_secret
    else New PaymentIntent
        PS->>SS: createPaymentIntent(booking, stripeOptions)
        SS->>Stripe: PaymentIntent.create()
        Note over Stripe: amount in cents, transfer_data for Connect
        Stripe-->>SS: PaymentIntent {id, client_secret}
        PS->>PS: Create Payment record (status: pending)
        PS-->>C: {client_secret, amount, payment_intent_id}
    end

    C->>Stripe: stripe.confirmCardPayment(client_secret)
    Stripe-->>C: Payment confirmed (or 3DS challenge)

    Stripe->>WH: POST /api/webhooks/stripe (payment_intent.succeeded)
    WH->>WH: Verify signature
    WH->>WH: Idempotency check (stripe_webhook_events table)
    WH->>PS: handlePaymentSuccess(paymentIntentId)
    PS->>PS: lockForUpdate() on Payment
    PS->>PS: Skip if already succeeded (idempotent)
    PS->>SS: retrievePaymentIntent() — verify status + amount
    PS->>PS: Update Payment: status=succeeded, card details
    PS->>PS: Update Booking: status=awaiting_approval, payment_status=paid
    PS->>PS: Dispatch PaymentSucceeded event
```

## Per-Operator Stripe Key Scoping

Every Stripe call uses operator-specific keys via `StripeService::useKeysForOperator()`:

```mermaid
flowchart TD
    A[Stripe API Call] --> B[StripeService::useKeysForOperator]
    B --> C{Platform settings exist?}
    C -->|No| D[Use default config keys]
    C -->|Yes| E{operator.stripe_mode}
    E -->|test| F[Use test_secret / test_key]
    E -->|live| G[Use live_secret / live_key]
    F --> H[Set currentSecretKey + currentPublishableKey]
    G --> H
    H --> I[All SDK calls use requestOptions with per-request api_key]
    Note over I: Avoids Stripe::setApiKey() race condition under Octane
```

## Stripe Connect (Destination Charges)

For operators with Stripe Connect accounts, payments use destination charges:

```
PaymentIntent.create({
    amount: ...,
    transfer_data: {
        destination: operator.stripe_account_id
    }
})
```

Only added when `stripe_account_id` exists and is NOT a test placeholder (`acct_test_*`).

## Saved Cards CRUD

```mermaid
sequenceDiagram
    participant C as Customer
    participant API as SavedCard API
    participant SCS as SavedCardService
    participant SS as StripeService
    participant Stripe as Stripe

    Note over C: Add a new card

    C->>API: POST /saved-cards/setup-intent
    API->>SCS: createSetupIntent(user)
    SCS->>SCS: configureStripeForUser(user)
    SCS->>SCS: getOrCreateStripeCustomer(user)
    Note over SCS: Per-operator customer via UserStripeCustomer table
    SCS->>SS: createSetupIntent(customerId)
    SS->>Stripe: SetupIntent.create({customer, payment_method_types: [card]})
    Stripe-->>SS: {setup_intent_id, client_secret}
    API-->>C: {client_secret, setup_intent_id, publishable_key}

    C->>Stripe: stripe.confirmCardSetup(client_secret, card element)
    Stripe-->>C: SetupIntent confirmed, payment_method_id

    C->>API: POST /saved-cards {payment_method_id}
    API->>SCS: saveCard(user, paymentMethodId)
    SCS->>SS: attachPaymentMethod(customerId, paymentMethodId)
    API-->>C: Card saved

    Note over C: List cards
    C->>API: GET /saved-cards
    API->>SCS: getSavedCards(user)
    SCS->>SS: listPaymentMethods(customerId, 'card')
    API-->>C: [{brand, last4, exp_month, exp_year, id}]

    Note over C: Delete card
    C->>API: DELETE /saved-cards/{paymentMethodId}
    API->>SCS: deleteCard(user, paymentMethodId)
    SCS->>SS: detachPaymentMethod(paymentMethodId)
    API-->>C: Card deleted

    Note over C: Set default card
    C->>API: PATCH /saved-cards/{paymentMethodId}/default
    API->>SCS: setDefaultCard(user, paymentMethodId)
    SCS->>SS: setDefaultPaymentMethod(customerId, paymentMethodId)
    API-->>C: Default card updated
```

## Refund Flow

```mermaid
sequenceDiagram
    participant Admin as Operator Admin
    participant API as Admin API
    participant PS as PaymentService
    participant SS as StripeService
    participant Stripe as Stripe

    Admin->>API: POST /admin/payments/{id}/refund {amount?, reason?}
    API->>PS: processRefund(payment, amount, reason)
    PS->>PS: lockForUpdate() on Payment
    PS->>SS: useKeysForOperator(operator)
    PS->>PS: Validate payment.status = succeeded
    PS->>PS: Calculate refundable amount (amount - refunded_amount)
    PS->>PS: Validate refund amount <= refundable

    PS->>SS: createRefund(paymentIntentId, amountCents, reason)
    SS->>Stripe: Refund.create({payment_intent, amount, reason})
    Stripe-->>SS: {refund_id, status, amount}

    PS->>PS: Update Payment:
    Note over PS: Full refund: status=refunded
    Note over PS: Partial refund: status=partially_refunded
    PS->>PS: Update booking.payment_status
    PS->>PS: Dispatch RefundProcessed event

    API-->>Admin: {refund_id, refunded_amount, total_refunded}
```

## Payment Failure Handling

```mermaid
sequenceDiagram
    participant Stripe as Stripe
    participant WH as Webhook
    participant PS as PaymentService

    Stripe->>WH: payment_intent.payment_failed
    WH->>WH: Verify signature + idempotency check
    WH->>PS: handlePaymentFailure(paymentIntentId, failureReason)
    PS->>PS: lockForUpdate() on Payment
    PS->>PS: Skip if already succeeded (don't overwrite success)
    PS->>PS: Update Payment: status=failed, failure_reason
```

## Webhook Handling with Idempotency

```mermaid
sequenceDiagram
    participant Stripe as Stripe
    participant WH as WebhookController
    participant DB as stripe_webhook_events

    Stripe->>WH: POST /api/webhooks/stripe
    WH->>WH: Verify Stripe-Signature header
    WH->>DB: INSERT stripe_event_id (claim event)

    alt Duplicate entry (already claimed)
        DB-->>WH: UNIQUE constraint error
        WH->>DB: SELECT processed_at
        alt processed_at IS NOT NULL
            WH-->>Stripe: 200 already_processed
        else processed_at IS NULL (another worker processing)
            WH-->>Stripe: 200 processing
        end
    else New event (claimed successfully)
        WH->>WH: Process event handler
        alt Success
            WH->>DB: UPDATE processed_at = now()
            WH-->>Stripe: 200 Webhook handled
        else Handler error
            WH->>DB: DELETE claimed row (release for retry)
            WH-->>Stripe: 500 (Stripe will retry)
        end
    end
```

**Handled webhook events:**

| Event | Handler |
|-------|---------|
| `payment_intent.succeeded` | `PaymentService::handlePaymentSuccess()` |
| `payment_intent.payment_failed` | `PaymentService::handlePaymentFailure()` |
| `charge.refunded` | Logged (refunds handled synchronously) |

## Payment Status Enum

| Status | Value | Description |
|--------|-------|-------------|
| Pending | `pending` | PaymentIntent created, awaiting confirmation |
| Processing | `processing` | Being processed by Stripe |
| Requires Action | `requires_action` | 3D Secure or similar required |
| Succeeded | `succeeded` | Payment captured successfully |
| Failed | `failed` | Payment failed |
| Cancelled | `cancelled` | PaymentIntent cancelled |
| Refunded | `refunded` | Fully refunded |
| Partially Refunded | `partially_refunded` | Partially refunded |

## Events Fired

| Event | When | Listeners |
|-------|------|-----------|
| `PaymentSucceeded` | Card payment confirmed | `SendPaymentConfirmation` |
| `PaymentFailed` | Payment failed | Logged |
| `RefundProcessed` | Refund completed | `SendRefundProcessedNotification` |

## Key Files

| Purpose | File |
|---------|------|
| Payment service | `app/Payment/Services/PaymentService.php` |
| Stripe service | `app/Services/External/StripeService.php` |
| Stripe Connect service | `app/Payment/Services/StripeConnectService.php` |
| Saved card service | `app/Payment/Services/SavedCardService.php` |
| Webhook controller | `app/Http/Controllers/Api/WebhookController.php` |
| Payment model | `app/Payment/Models/Payment.php` |
| Payment status enum | `app/Payment/Enums/PaymentStatus.php` |
| Payment events | `app/Payment/Events/` |
| Payment listeners | `app/Payment/Listeners/` |
| User Stripe customer | `app/Payment/Models/UserStripeCustomer.php` |
