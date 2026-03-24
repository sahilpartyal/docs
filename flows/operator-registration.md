---
title: "Operator Registration"
description: "Operator signup, subscription and onboarding flow."
---

# Operator Registration Flow

Self-service operator signup: registration form, plan selection, Stripe Connect onboarding, and post-registration activation.

## Actors

- **New Operator** — fills registration form, completes Stripe setup
- **System** — creates operator, subscription, provisions widgets
- **Stripe** — handles payment onboarding
- **Super Admin** — can manage operators after registration

## Entry Points

| Channel | URL | Controller |
|---------|-----|------------|
| Registration page | `GET /register` | `Web\RegistrationController` (Inertia) |
| Submit registration | `POST /api/register` | `OperatorRegistrationController::register()` |
| Stripe callback | `GET /register/callback` | `OperatorRegistrationController::callback()` |
| Setup payment | `POST /api/register/subscription-payment` | `OperatorRegistrationController::setupPayment()` |
| Resume onboarding | `POST /api/register/resume-onboarding` | `OperatorRegistrationController::resumeOnboarding()` |
| Available plans | `GET /api/register/plans` | `OperatorRegistrationController::plans()` |
| Check email | `POST /api/register/check-email` | `OperatorRegistrationController::checkEmail()` |

## Registration Flow

```mermaid
sequenceDiagram
    participant Op as New Operator
    participant SPA as Registration SPA
    participant API as Registration API
    participant ORS as OperatorRegistrationService
    participant Stripe as Stripe
    participant Email as Email Service

    Op->>SPA: Fill registration form
    SPA->>API: POST /register
    API->>ORS: registerOperator(data)

    Note over ORS: 8-step transaction begins

    ORS->>ORS: 1. createOperator()
    Note over ORS: status=pending, generate unique slug/subdomain
    Note over ORS: Set onboarding_checklist (6 items, all false)

    ORS->>ORS: 2. Create default "In-House" Supplier
    Note over ORS: Every operator gets a default supplier

    ORS->>ORS: 3. Save OperatorCoverageRegion per service_state

    ORS->>ORS: 4. createAdminUser()
    Note over ORS: role=admin, operator_id set, email_verified_at=null

    ORS->>ORS: 5. createSubscription()
    Note over ORS: Find SubscriptionPlan by plan_id
    Note over ORS: status: trialing (if plan.trial_days > 0) or active
    Note over ORS: billing_cycle: monthly or annual
    Note over ORS: period_end based on billing_cycle

    ORS->>ORS: 6. Apply feature defaults from plan
    Note over ORS: featureToggleService.applyPlanDefaults(operator)

    ORS->>ORS: 7. Provision widgets from plan
    Note over ORS: widgetService.provisionWidgetsForOperator(operator, plan)

    ORS->>ORS: 8. initiateStripeOnboarding()
    ORS->>Stripe: Create Connect Express account
    Stripe-->>ORS: account_id
    ORS->>ORS: Save stripe_account_id on operator
    ORS->>Stripe: Create AccountLink (onboarding)
    Stripe-->>ORS: onboarding_url

    ORS->>ORS: Audit log: operator.self_registered

    ORS->>Email: SendOperatorWelcomeEmail job

    API-->>SPA: {operator, user, subscription, stripe_onboarding_url}
    SPA->>Op: Redirect to Stripe onboarding
```

## Stripe Connect Onboarding

```mermaid
sequenceDiagram
    participant Op as Operator
    participant Stripe as Stripe Dashboard
    participant CB as Callback URL
    participant ORS as OperatorRegistrationService
    participant SCS as StripeConnectService

    Op->>Stripe: Complete onboarding (business details, bank account)

    alt Onboarding complete
        Stripe->>CB: Redirect to /register/callback?operator_id=X
        CB->>ORS: completeStripeOnboarding(operator)
        ORS->>SCS: getOnboardingStatus(operator)
        SCS->>Stripe: Retrieve Account
        Stripe-->>SCS: charges_enabled=true, payouts_enabled=true

        alt Was pending
            ORS->>ORS: activateOperator()
            Note over ORS: status → trial (if trialing) or active
            Note over ORS: activated_at → now()
        end

        CB-->>Op: Redirect to /admin/dashboard
    else Onboarding incomplete (refresh)
        Stripe->>CB: Redirect with refresh=true
        CB->>ORS: resumeStripeOnboarding(operator)
        ORS->>SCS: Create new AccountLink
        SCS-->>CB: New onboarding_url
        CB-->>Op: Redirect back to Stripe
    end
```

## Subscription Payment Setup

After Stripe Connect onboarding, operator sets up subscription billing:

```mermaid
sequenceDiagram
    participant Op as Operator Admin
    participant API as Registration API
    participant ORS as OperatorRegistrationService
    participant Stripe as Stripe

    Op->>API: POST /register/subscription-payment {payment_method_id}
    API->>ORS: setupSubscriptionPayment(operator, paymentMethodId)

    ORS->>Stripe: Create or update Stripe Customer
    Note over Stripe: Attach payment_method, set as default

    ORS->>ORS: getOrCreateStripePrice(plan, billingCycle)
    Note over ORS: Create Stripe Product if needed
    Note over ORS: Create Stripe Price (monthly or annual interval)

    ORS->>Stripe: Subscription.create()
    Note over Stripe: customer, price, trial_period_days (if plan has trial)

    ORS->>ORS: Update local subscription
    Note over ORS: stripe_subscription_id, stripe_customer_id

    ORS->>ORS: Activate operator
    Note over ORS: stripe_onboarding_complete = true
    Note over ORS: status = trial or active

    ORS->>ORS: Update onboarding_checklist: stripe_setup = true

    API-->>Op: {subscription_id, status, trial_end}
```

## Post-Registration State

After successful registration, the operator has:

| Item | Status |
|------|--------|
| Operator record | `pending` (until Stripe complete) → `trial` or `active` |
| Admin user | Created, email not verified |
| Default supplier | "In-House" supplier created |
| Subscription | `trialing` or `active` based on plan |
| Feature toggles | Plan defaults applied |
| Widget configs | Provisioned from plan templates |
| Stripe Connect | Account created, onboarding link provided |
| Onboarding checklist | 6 items, `stripe_setup` marked after payment |

## Onboarding Checklist

```json
{
    "stripe_setup": false,
    "add_vehicle_type": false,
    "add_vehicle": false,
    "add_driver": false,
    "configure_pricing": false,
    "test_booking": false
}
```

Each item is checked off as the operator completes setup in their admin portal.

## Operator Status Transitions

```mermaid
stateDiagram-v2
    [*] --> pending: Registration submitted
    pending --> trial: Stripe complete + plan has trial
    pending --> active: Stripe complete + no trial
    trial --> active: Trial ends, subscription active
    trial --> suspended: Trial expired, no payment
    active --> suspended: Payment failed
    suspended --> active: Payment resumed
    active --> cancelled: Admin cancels
    cancelled --> [*]
```

## Email Validation

| Check | Service Method |
|-------|---------------|
| Operator email available | `isOperatorEmailAvailable(email)` — checks `operators.email` |
| Admin email available | `isAdminEmailAvailable(email)` — checks `users.email` where role IN (admin, super_admin) |

Note: Customer emails are per-operator unique, so a customer email in another operator does NOT conflict.

## Super Admin Operator Management

After registration, super admins can manage operators through the Super Admin SPA:

| Action | Description |
|--------|-------------|
| View operators | List all operators with status, plan, Stripe status |
| Cancel operator | Set status to cancelled |
| Suspend operator | Set status to suspended |
| Delete operator | Soft delete |
| Restore operator | Restore soft-deleted operator |
| Force delete | Permanent deletion (cascade) |
| Impersonate | Login as operator admin |

## Events Fired

| Event | When |
|-------|------|
| `operator.self_registered` | Registration transaction complete (audit log) |
| `operator.stripe_connect.account_created` | Stripe Connect account created |
| `operator.stripe_connect.onboarding_started` | Onboarding link generated |
| `operator.stripe_connect.onboarding_completed` | Stripe onboarding verified |
| `operator.activated_self_service` | Operator status changed to active |
| `operator.subscription_payment_setup` | Subscription payment configured |

## Key Files

| Purpose | File |
|---------|------|
| Registration service | `app/Services/OperatorRegistrationService.php` |
| Registration controller | `app/Http/Controllers/OperatorRegistrationController.php` |
| Registration request | `app/Http/Requests/OperatorRegistrationRequest.php` |
| Stripe Connect service | `app/Payment/Services/StripeConnectService.php` |
| Widget provisioning | `app/Embed/Services/WidgetProvisioningService.php` |
| Feature toggle service | `app/Services/FeatureToggleService.php` |
| Subscription management | `app/Services/SubscriptionManagementService.php` |
| Operator management | `app/Services/OperatorManagementService.php` |
| Welcome email job | `app/Jobs/SendOperatorWelcomeEmail.php` |
| Operator model | `app/Models/Operator.php` |
| Subscription model | `app/Models/OperatorSubscription.php` |
| Registration test | `tests/Feature/OperatorRegistrationTest.php` |
