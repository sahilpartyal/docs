---
title: "Invoicing Flow"
description: "Invoice creation, sending, payment and reconciliation."
---

# Invoice Management Flow

Manual invoice creation, auto-generation from bookings, invoice lifecycle, PDF generation, overdue reminders, and bulk actions.

## Actors

- **Operator Admin** — creates, sends, manages invoices
- **Customer** — receives invoice, pays via payment link
- **System** — auto-generates invoices, sends overdue reminders

## Entry Points

| Channel | URL | Controller |
|---------|-----|------------|
| List invoices | `GET /api/v1/admin/invoices` | `Api\Admin\InvoiceController::index()` |
| Create invoice | `POST /api/v1/admin/invoices` | `Api\Admin\InvoiceController::store()` |
| Update invoice | `PUT /api/v1/admin/invoices/{id}` | `Api\Admin\InvoiceController::update()` |
| Send invoice | `POST /api/v1/admin/invoices/{id}/send` | `Api\Admin\InvoiceController::send()` |
| Send reminder | `POST /api/v1/admin/invoices/{id}/remind` | `Api\Admin\InvoiceController::remind()` |
| Mark as paid | `POST /api/v1/admin/invoices/{id}/mark-paid` | `Api\Admin\InvoiceController::markPaid()` |
| Void invoice | `POST /api/v1/admin/invoices/{id}/void` | `Api\Admin\InvoiceController::void()` |
| Generate PDF | `GET /api/v1/admin/invoices/{id}/pdf` | `Api\Admin\InvoiceController::pdf()` |
| Bulk action | `POST /api/v1/admin/invoices/bulk` | `Api\Admin\InvoiceController::bulk()` |
| Unbilled bookings | `GET /api/v1/admin/invoices/unbilled` | `Api\Admin\InvoiceController::unbilled()` |
| Statistics | `GET /api/v1/admin/invoices/stats` | `Api\Admin\InvoiceController::stats()` |
| Auto-generate (widget) | Internal | `InvoiceManagementService::createAndSendFromBooking()` |

## Invoice Lifecycle

```mermaid
stateDiagram-v2
    [*] --> draft: Invoice created
    draft --> sent: Admin sends (or auto-sent)
    draft --> void: Admin voids
    sent --> paid: Full payment received
    sent --> partial: Partial payment received
    sent --> overdue: Past due_date, not paid
    sent --> void: Admin voids
    partial --> paid: Remaining balance paid
    partial --> overdue: Past due_date
    overdue --> paid: Payment received
    overdue --> void: Admin voids
    paid --> [*]
    void --> [*]
```

## Manual Invoice Creation

```mermaid
sequenceDiagram
    participant Admin as Operator Admin
    participant API as Admin API
    participant IMS as InvoiceManagementService
    participant PDF as PdfGenerationService

    Admin->>API: POST /admin/invoices {customer_id, items[], billing_*, dates}
    API->>IMS: createInvoice(operator, customer, data)

    IMS->>IMS: Create Invoice (status: draft)
    IMS->>IMS: Set issue_date (default: today)
    IMS->>IMS: Set due_date (default: 14 days)
    IMS->>IMS: Set billing_name, billing_email from customer

    loop Each item
        IMS->>IMS: addInvoiceItem(invoice, item, sortOrder)
        Note over IMS: Each item: description, quantity, unit_price, tax_rate
        Note over IMS: Optional: booking_id to link to a booking
    end

    IMS->>IMS: invoice.recalculateTotals()
    Note over IMS: subtotal, tax_amount, total_amount, amount_due

    API-->>Admin: Invoice with items
```

## Auto-Generated from Booking

When a widget booking uses `payment_method: invoice`, the invoice is auto-created and sent:

```mermaid
sequenceDiagram
    participant ES as EmbedService
    participant IMS as InvoiceManagementService
    participant NS as NotificationService
    participant PDF as PdfGenerationService

    ES->>IMS: createAndSendFromBooking(booking)

    IMS->>IMS: Check for existing invoice (prevent duplicates)
    IMS->>IMS: createFromBooking(booking)
    IMS->>IMS: Create Invoice (status: draft)
    IMS->>IMS: Add booking as single line item
    Note over IMS: Description: "Booking #REF - pickup to dropoff (date)"
    Note over IMS: unit_price = booking.subtotal
    Note over IMS: tax_rate from TaxConfig::getDefaultRate(operator_id)
    IMS->>IMS: recalculateTotals()

    IMS->>IMS: sendInvoice(invoice)
    IMS->>PDF: generateInvoicePdf(invoice)
    IMS->>IMS: generatePaymentToken()
    Note over IMS: Signed token for online payment link
    IMS->>IMS: Update status: draft → sent, set sent_at
    IMS->>NS: sendInvoiceNotification(invoice)
    NS->>NS: Dispatch SendEmailNotification job
    Note over NS: Email with PDF attachment + payment link
```

## Sending an Invoice

```mermaid
sequenceDiagram
    participant Admin as Operator Admin
    participant API as Admin API
    participant IMS as InvoiceManagementService
    participant PDF as PdfGenerationService
    participant NS as NotificationService

    Admin->>API: POST /admin/invoices/{id}/send
    API->>IMS: sendInvoice(invoice)

    alt PDF not generated
        IMS->>PDF: generateInvoicePdf(invoice)
        PDF->>PDF: Render invoice Blade template
        PDF->>PDF: DomPDF → save to storage
        PDF-->>IMS: PDF path
    end

    alt No payment token
        IMS->>IMS: invoice.generatePaymentToken()
        Note over IMS: Signed URL for customer to pay online
    end

    IMS->>IMS: Update status → sent, sent_at → now()
    IMS->>NS: sendInvoiceNotification(invoice)
    NS->>NS: Create NotificationLog (channel: EMAIL, type: INVOICE_SENT)
    NS->>NS: Dispatch SendEmailNotification job

    API-->>Admin: Invoice sent
```

## Mark as Paid

```mermaid
sequenceDiagram
    participant Admin as Operator Admin
    participant API as Admin API
    participant IMS as InvoiceManagementService

    Admin->>API: POST /admin/invoices/{id}/mark-paid {amount?}
    API->>IMS: markAsPaid(invoice, amount?)

    IMS->>IMS: Validate not void and not already paid
    IMS->>IMS: paymentAmount = amount ?? invoice.amount_due

    IMS->>IMS: Create InvoicePayment record
    Note over IMS: InvoicePayment created hook → invoice.recordPayment()
    Note over IMS: recordPayment() atomically updates:
    Note over IMS: - amount_paid += payment amount
    Note over IMS: - amount_due = total_amount - amount_paid
    Note over IMS: - status → paid (if amount_due <= 0)
    Note over IMS: - paid_at → now() (if fully paid)

    API-->>Admin: Updated invoice
```

## Void Invoice

```mermaid
sequenceDiagram
    participant Admin as Operator Admin
    participant API as Admin API
    participant IMS as InvoiceManagementService

    Admin->>API: POST /admin/invoices/{id}/void
    API->>IMS: voidInvoice(invoice)
    IMS->>IMS: Validate not paid (must refund first)
    IMS->>IMS: Validate not already void
    IMS->>IMS: Update status → void
    API-->>Admin: Invoice voided
```

## Overdue Reminders

```mermaid
sequenceDiagram
    participant Cron as Scheduler
    participant Job as SendInvoiceOverdueReminders
    participant IMS as InvoiceManagementService
    participant NS as NotificationService

    Cron->>Job: Run daily
    Job->>IMS: getOverdueInvoices(operatorId)
    IMS-->>Job: Invoices where status IN (sent, partial) AND due_date < today

    loop Each overdue invoice
        Job->>IMS: sendReminder(invoice)
        IMS->>IMS: Validate not already paid
        IMS->>NS: sendInvoiceOverdueNotification(invoice)
        NS->>NS: Create NotificationLog (type: INVOICE_OVERDUE)
        NS->>NS: Dispatch SendEmailNotification job
        Note over NS: Subject: "Payment Reminder - Invoice #INV-YYYYMM-XXXX is Overdue"
    end
```

## Bulk Actions

```mermaid
sequenceDiagram
    participant Admin as Operator Admin
    participant API as Admin API
    participant IMS as InvoiceManagementService

    Admin->>API: POST /admin/invoices/bulk {invoice_ids[], action}
    API->>IMS: bulkAction(invoiceIds, action, operatorId)

    IMS->>IMS: Load invoices (scoped to operator_id)
    loop Each invoice
        IMS->>IMS: Execute action
        Note over IMS: Actions: send, mark_paid, void, generate_pdf, send_reminder
        alt Success
            IMS->>IMS: Add to success[]
        else Error
            IMS->>IMS: Add to failed[] with error message
        end
    end

    API-->>Admin: {success: [ids], failed: [{id, error}]}
```

**Available bulk actions:**

| Action | Description |
|--------|-------------|
| `send` | Send all selected invoices |
| `mark_paid` | Mark all as paid |
| `void` | Void all selected |
| `generate_pdf` | Regenerate PDFs |
| `send_reminder` | Send overdue reminders |

## Invoice Statistics

```
GET /api/v1/admin/invoices/stats
```

Returns:

| Metric | Description |
|--------|-------------|
| `total_invoices` | Total invoice count |
| `draft` | Count in draft status |
| `sent` | Count sent but unpaid |
| `paid` | Count fully paid |
| `overdue` | Count past due |
| `total_outstanding` | Sum of unpaid amount_due |
| `total_paid_this_month` | Revenue from paid invoices this month |

## Unbilled Bookings

```
GET /api/v1/admin/invoices/unbilled?customer_id=X&from=YYYY-MM-DD&to=YYYY-MM-DD&group_by=service_type
```

Returns completed bookings that have no associated InvoiceItems, optionally grouped by service type.

## PDF Generation

Invoices use DomPDF via `PdfGenerationService::generateInvoicePdf()`:

- Template: `resources/views/pdf/invoice.blade.php`
- Includes: operator logo, billing details, line items, totals, payment terms
- Stored at: `storage/app/invoices/{operator_id}/{invoice_number}.pdf`
- Re-generated on invoice update (old PDF path + payment token cleared)

## Auto-Generated Invoice Numbers

Format: `INV-YYYYMM-XXXX` (e.g., `INV-202603-0042`)

Generated by the Invoice model's `generating` event, using a counter scoped to operator + month.

## Events Fired

| Event Type | When |
|------------|------|
| `INVOICE_SENT` | Invoice emailed to customer |
| `INVOICE_OVERDUE` | Overdue reminder sent |
| `INVOICE_PAID` | Invoice fully paid |

## Key Files

| Purpose | File |
|---------|------|
| Invoice management service | `app/Payment/Services/InvoiceManagementService.php` |
| PDF generation | `app/Services/PdfGenerationService.php` |
| Invoice model | `app/Payment/Models/Invoice.php` |
| InvoiceItem model | `app/Payment/Models/InvoiceItem.php` |
| InvoicePayment model | `app/Payment/Models/InvoicePayment.php` |
| Invoice controller | `app/Http/Controllers/Api/Admin/InvoiceController.php` |
| Overdue reminders job | `app/Payment/Jobs/SendInvoiceOverdueReminders.php` |
| PDF template | `resources/views/pdf/invoice.blade.php` |
| Notification service | `app/Notification/Services/NotificationService.php` |
