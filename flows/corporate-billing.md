---
title: "Corporate Billing"
description: "Automated billing cycles, prepaid balance and auto-charge."
---

# Corporate Billing Flow

Automated billing cycles for corporate companies, including prepaid balance management, auto-charge card, monthly invoice generation, commission tracking, and low balance alerts.

## Actors

- **System** — runs billing cycle daily via scheduler
- **Operator Admin** — configures companies, manages balances, views commissions
- **Company Portal Admin** — views invoices, manages payment methods

## Entry Points

| Channel | URL / Command | Handler |
|---------|--------------|---------|
| Process billing | `php artisan corporate:process-billing` | `ProcessCorporateBilling` command |
| Dry run | `php artisan corporate:process-billing --dry-run` | Same (preview mode) |
| Company settings | `PATCH /api/v1/admin/companies/{id}` | `Api\Admin\CompanyController::update()` |
| Balance top-up | `POST /api/v1/admin/companies/{id}/balance` | `Api\Admin\CompanyController::topUp()` |

## Payment Terms and Collection Methods

```mermaid
flowchart TD
    A[Company Payment Configuration] --> B{payment_terms}
    B -->|on_booking| C[Pay at time of booking]
    B -->|prepaid| D[Deduct from prepaid_balance]
    B -->|weekly/fortnightly/net_X| E[Recurring billing cycle]

    E --> F{payment_collection_method}
    F -->|auto_charge_card| G[Charge stripe_payment_method_id]
    F -->|send_invoice| H[Email invoice with payment link]
```

## Automated Billing Cycle

```mermaid
sequenceDiagram
    participant Cron as Scheduler (daily)
    participant Cmd as ProcessCorporateBilling
    participant CIS as CorporateInvoiceService
    participant IMS as InvoiceManagementService
    participant SS as StripeService
    participant NS as NotificationService
    participant Stripe as Stripe

    Cron->>Cmd: corporate:process-billing

    Cmd->>Cmd: Query active companies WHERE next_billing_date <= today
    Cmd->>Cmd: Filter by recurring terms (weekly, fortnightly, net_15-60)

    loop Each company due for billing
        Cmd->>Cmd: Calculate billing period
        Note over Cmd: period_start = last_billed_date + 1 day
        Note over Cmd: period_end = next_billing_date - 1 day

        Cmd->>CIS: generateMonthlyInvoice(company, periodStart, periodEnd)

        CIS->>CIS: lockForUpdate() on company row
        CIS->>CIS: Check for existing invoice for this period (prevent duplicates)
        CIS->>CIS: Query completed, unbilled bookings in period
        Note over CIS: Bookings WHERE company_id AND status=completed
        Note over CIS: AND pickup_datetime BETWEEN period AND no InvoiceItems

        alt No unbilled bookings
            CIS-->>Cmd: null
            Cmd->>Cmd: Advance billing date (skip period)
        else Has bookings
            CIS->>CIS: Create Invoice (type: corporate_monthly)
            Note over CIS: billing_name, billing_email from Company
            Note over CIS: due_date from company.getDueDateFromTerms()
            CIS->>CIS: Add each booking as InvoiceItem
            Note over CIS: Description: "Booking #REF - pickup to dropoff (date)"
            Note over CIS: unit_price = booking.subtotal, tax from TaxConfig
            CIS->>CIS: recalculateTotals()
            CIS-->>Cmd: Invoice created

            alt Zero amount invoice
                Cmd->>Cmd: Set status=sent, send email, advance date
            else shouldAutoCharge() is true
                Cmd->>SS: useKeysForOperator(operator)
                Cmd->>Stripe: PaymentIntent.create(off_session, confirm: true)
                Note over Stripe: customer = company.stripe_customer_id
                Note over Stripe: payment_method = company.stripe_payment_method_id
                Note over Stripe: metadata includes invoice_id, company_id

                alt Charge succeeds
                    Stripe-->>Cmd: status = succeeded
                    Cmd->>Cmd: Create InvoicePayment record
                    Note over Cmd: InvoicePayment hook → invoice.recordPayment()
                    Note over Cmd: Invoice auto-transitions to paid
                else Card declined
                    Cmd->>Cmd: Fallback: set status=sent, send invoice email
                    Note over Cmd: Company can pay via payment link
                end
            else Send invoice (default)
                Cmd->>Cmd: Set invoice status = sent
                Cmd->>NS: sendInvoiceNotification(invoice)
                Note over NS: Email with PDF + payment link
            end

            Cmd->>Cmd: Advance billing date
            Note over Cmd: last_billed_date = current next_billing_date
            Note over Cmd: next_billing_date = calculateNextBillingDate()
        end
    end
```

## Billing Period Calculation

| Payment Terms | Period Length | Next Billing Date |
|--------------|-------------|-------------------|
| `weekly` | 7 days | Next Monday |
| `fortnightly` | 14 days | Monday + 1 week |
| `net_15` | 15 days | +15 days |
| `net_30` | 30 days | +30 days |
| `net_45` | 45 days | +45 days |
| `net_60` | 60 days | +60 days |

## Prepaid Balance Flow

```mermaid
sequenceDiagram
    participant Admin as Operator Admin
    participant API as Admin API
    participant CS as CompanyService

    Note over Admin: Top up balance
    Admin->>API: POST /admin/companies/{id}/balance {amount, type: credit}
    API->>CS: Create BalanceTransaction (type: credit)
    CS->>CS: company.prepaid_balance += amount
    API-->>Admin: New balance

    Note over Admin: When booking is created for prepaid company
    API->>CS: Create BalanceTransaction (type: debit, booking_id)
    CS->>CS: company.prepaid_balance -= booking.total_amount

    Note over Admin: Refund returns to balance
    API->>CS: Create BalanceTransaction (type: refund, booking_id)
    CS->>CS: company.prepaid_balance += refund_amount
```

## Auto-Charge Card Flow

Requires:
- `payment_collection_method = 'auto_charge_card'`
- `stripe_customer_id` is set
- `stripe_payment_method_id` is set

```mermaid
flowchart TD
    A[shouldAutoCharge?] --> B{stripe_customer_id?}
    B -->|No| F[Send invoice instead]
    B -->|Yes| C{stripe_payment_method_id?}
    C -->|No| F
    C -->|Yes| D[PaymentIntent.create off_session + confirm]
    D -->|succeeded| E[InvoicePayment created → Invoice marked paid]
    D -->|CardException| G[Fallback: send invoice email]
```

## Commission Tracking

Each Company has optional commission tracking:

| Field | Description |
|-------|-------------|
| `commission_type` | `none`, `percentage`, `flat` |
| `commission_rate` | Percentage rate (e.g., 15.00 for 15%) |
| `commission_flat` | Flat amount per booking |

Commission is tracked per booking via the `CorporateCommission` model and appears on reports.

## Low Balance Alerts

When `prepaid_balance` drops below `low_balance_threshold`:
- Alert sent to company portal admin
- Alert sent to operator admin
- Tracked via `BalanceTransaction` history

## Due Date Calculation

`Company::getDueDateFromTerms()` maps payment terms to due dates:

| Term | Days Added |
|------|-----------|
| `on_booking` | 0 |
| `prepaid` | 0 |
| `weekly` | 7 |
| `fortnightly` | 14 |
| `net_15` | 15 |
| `net_30` | 30 |
| `net_45` | 45 |
| `net_60` | 60 |

## Key Files

| Purpose | File |
|---------|------|
| Billing command | `app/Console/Commands/ProcessCorporateBilling.php` |
| Corporate invoice service | `app/Corporate/Services/CorporateInvoiceService.php` |
| Company service | `app/Corporate/Services/CompanyService.php` |
| Company model | `app/Corporate/Models/Company.php` |
| Balance transaction model | `app/Corporate/Models/BalanceTransaction.php` |
| Corporate commission model | `app/Corporate/Models/CorporateCommission.php` |
| Invoice management service | `app/Payment/Services/InvoiceManagementService.php` |
| Stripe service | `app/Services/External/StripeService.php` |
