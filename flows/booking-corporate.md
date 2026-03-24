---
title: "Corporate Booking Flow"
description: "Corporate employee bookings with company billing and pricing."
---

# Corporate Booking Flow

Corporate bookings made by employees or portal admins of a Company entity. Companies have configurable payment terms, pricing adjustments, cost centre tracking, and automated billing cycles.

## Actors

- **Employee** — customer with `company_id` set, books via widget or SPA
- **Portal Admin** — company user with `portal_admin_user_id`, manages employees and views reports
- **Operator Admin** — sets up companies, manages billing, approves bookings

## Entry Points

| Channel | URL | Controller |
|---------|-----|------------|
| Employee self-service (widget) | `POST /api/v1/widget/bookings` | `Widget\WidgetApiController::createBooking()` |
| Employee self-service (SPA) | `POST /api/v1/bookings` | `Api\V1\BookingController::store()` |
| Admin creates for company | `POST /api/v1/admin/bookings` | `Api\Admin\BookingController::store()` |
| Corporate booking URL | `https://bookmytransfer.net/?corp={company_code}` | Widget with corp parameter |
| Company CRUD | `POST /api/v1/admin/companies` | `Api\Admin\CompanyController` |
| Employee management | `POST /api/v1/admin/companies/{id}/employees` | `Api\Admin\CompanyController` |

## Company Setup

```mermaid
sequenceDiagram
    participant Admin as Operator Admin
    participant API as Admin API
    participant CS as CompanyService

    Admin->>API: POST /admin/companies (name, billing, payment_terms, pricing_mode)
    API->>CS: create()
    CS->>CS: Create Company record
    CS->>CS: Set pricing_mode (discount / markup / custom_rate)
    CS->>CS: Set payment_terms (on_booking, prepaid, weekly, net_30, etc.)
    CS->>CS: Set payment_collection_method (auto_charge_card / send_invoice)
    CS->>CS: Calculate next_billing_date for recurring terms
    API-->>Admin: Company created with company_code

    Admin->>API: POST /admin/companies/{id}/employees (user_id)
    API->>CS: assignEmployee()
    CS->>CS: Set user.company_id
    CS->>CS: Create OperatorCustomer pivot (source: company_assignment)
    API-->>Admin: Employee assigned
```

## Payment Terms

| Term | Value | Due Date | Billing Cycle |
|------|-------|----------|---------------|
| Pay on Booking | `on_booking` | Immediate | None |
| Prepaid Balance | `prepaid` | Immediate (from balance) | None |
| Weekly | `weekly` | 7 days | Every Monday |
| Fortnightly | `fortnightly` | 14 days | Every other Monday |
| Net 15 | `net_15` | 15 days | 15-day cycle |
| Net 30 | `net_30` | 30 days | 30-day cycle |
| Net 45 | `net_45` | 45 days | 45-day cycle |
| Net 60 | `net_60` | 60 days | 60-day cycle |

## Pricing Modes

| Mode | Value | Effect |
|------|-------|--------|
| Discount | `discount` | Reduces standard price by `pricing_percentage`% |
| Markup | `markup` | Increases standard price by `pricing_percentage`% |
| Custom Rate | `custom_rate` | Uses `pricing_percentage` as a custom multiplier |

## Payment Collection Methods

| Method | Value | Behavior |
|--------|-------|----------|
| Auto-charge card | `auto_charge_card` | Charges `stripe_payment_method_id` on billing date |
| Send invoice | `send_invoice` | Generates and emails invoice with payment link |

## Employee Self-Service Booking

```mermaid
sequenceDiagram
    participant E as Employee
    participant W as Widget / SPA
    participant API as Booking API
    participant BS as BookingService

    E->>W: Access corporate booking URL (corp=COMPANY_CODE)
    W->>API: Standard booking flow (see booking-widget.md)

    Note over API: Employee identified by user.company_id
    API->>BS: Create booking with company_id set

    alt Payment Terms: on_booking
        Note over API: Standard card/invoice payment flow
    else Payment Terms: prepaid
        API->>API: Deduct from company.prepaid_balance
    else Payment Terms: weekly/net_30/etc.
        Note over API: No immediate payment required
        API->>API: Booking created, billed on next billing cycle
    end

    API-->>E: Booking confirmed

    opt CC on Booking Confirm enabled
        API->>API: NotificationService::sendBookingConfirmedCcNotification()
        Note over API: CC company billing_email and contact_email
    end
```

## Admin Booking for Company

```mermaid
sequenceDiagram
    participant Admin as Operator Admin
    participant API as Admin API
    participant BS as BookingService

    Admin->>API: POST /admin/bookings (company_id, customer_id or new customer)
    API->>BS: createAdminBooking()
    BS->>BS: Set booking.company_id
    BS->>BS: Apply company pricing adjustment (discount/markup)
    BS-->>Admin: Booking created

    opt Cost Centre / PO Number
        Admin->>API: Include cost_centre, po_number in booking data
        Note over API: Stored on booking for reporting and invoicing
    end
```

## Corporate Billing Cycle (Automated)

```mermaid
sequenceDiagram
    participant Cron as Scheduler
    participant Cmd as ProcessCorporateBilling
    participant CIS as CorporateInvoiceService
    participant Stripe as Stripe
    participant NS as NotificationService

    Cron->>Cmd: corporate:process-billing (daily)
    Cmd->>Cmd: Find companies where next_billing_date <= today

    loop Each company due for billing
        Cmd->>CIS: generateMonthlyInvoice(company, periodStart, periodEnd)
        CIS->>CIS: Lock company row (prevent concurrent generation)
        CIS->>CIS: Check for existing invoice (prevent duplicates)
        CIS->>CIS: Query completed, unbilled bookings in period
        CIS->>CIS: Create Invoice with company billing details
        CIS->>CIS: Add each booking as InvoiceItem
        CIS->>CIS: recalculateTotals()
        CIS-->>Cmd: Invoice created

        alt Auto-charge card (shouldAutoCharge)
            Cmd->>Stripe: PaymentIntent.create(off_session, confirm: true)
            Stripe-->>Cmd: succeeded
            Cmd->>Cmd: Create InvoicePayment record
            Note over Cmd: Invoice auto-marked as paid
        else Send invoice
            Cmd->>Cmd: Set invoice status to 'sent'
            Cmd->>NS: sendInvoiceNotification()
            Note over Cmd: Customer receives email with payment link
        end

        Cmd->>Cmd: Advance next_billing_date
    end
```

**Artisan command:** `php artisan corporate:process-billing` (runs daily via scheduler)

**Dry run:** `php artisan corporate:process-billing --dry-run`

## Commission Tracking

| Commission Type | Value | Calculation |
|-----------------|-------|-------------|
| None | `none` | No commission tracked |
| Percentage | `percentage` | `total_amount * commission_rate / 100` |
| Flat | `flat` | `commission_flat` per booking |

Commissions are tracked per booking via the `Company` model's `commission_type`, `commission_rate`, and `commission_flat` fields.

## Events Fired

| Event Type | When |
|------------|------|
| `BOOKING_CREATED` or `ADMIN_BOOKING_CREATED` | Booking saved |
| `BOOKING_CONFIRMED` | CC notification sent to company if `cc_on_booking_confirm` |
| `PAYMENT_SUCCEEDED` | Card charged (auto-charge or on-booking) |

## Key Files

| Purpose | File |
|---------|------|
| Company model | `app/Corporate/Models/Company.php` |
| Company service | `app/Corporate/Services/CompanyService.php` |
| Corporate invoice service | `app/Corporate/Services/CorporateInvoiceService.php` |
| Corporate billing command | `app/Console/Commands/ProcessCorporateBilling.php` |
| Balance transaction model | `app/Corporate/Models/BalanceTransaction.php` |
| Corporate commission model | `app/Corporate/Models/CorporateCommission.php` |
| Company controller | `app/Http/Controllers/Api/Admin/CompanyController.php` |
