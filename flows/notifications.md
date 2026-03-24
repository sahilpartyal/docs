---
title: "Notifications"
description: "Email, SMS and push notification delivery flow."
---

# Notification System Flow

Event-driven notification pipeline: Event -> Listener -> NotificationService -> Job -> External Service. Supports email (Zoho SMTP), SMS (Twilio), and push notifications (Firebase FCM).

## Actors

- **System** — dispatches notifications based on booking events
- **Customer** — receives booking updates via email, SMS, push
- **Driver** — receives job offers and assignments via push, email
- **Operator Admin** — receives admin alerts, manages email templates

## Entry Points

| Channel | URL | Controller |
|---------|-----|------------|
| Email templates | `GET /api/v1/admin/email-templates` | `Api\Admin\EmailTemplateController::index()` |
| Create template | `POST /api/v1/admin/email-templates` | `Api\Admin\EmailTemplateController::store()` |
| Update template | `PUT /api/v1/admin/email-templates/{id}` | `Api\Admin\EmailTemplateController::update()` |
| Notification preferences | `PATCH /api/v1/settings/notifications` | `Api\V1\SettingsController` |
| Device token registration | `POST /api/v1/device-tokens` | `Api\V1\DeviceTokenController::store()` |

## Architecture

```mermaid
flowchart LR
    A[Domain Event] --> B[Event Listener]
    B --> C[NotificationService]
    C --> D[NotificationLog created]
    D --> E{Channel}
    E -->|Email| F[SendEmailNotification Job]
    E -->|SMS| G[SendSmsNotification Job]
    E -->|Push| H[SendPushNotification Job]
    F --> I[SendGridService / Zoho SMTP]
    G --> J[TwilioService]
    H --> K[FirebaseService]
    I --> L[NotificationLog.markAsSent]
    J --> L
    K --> L
```

## Notification Pipeline (Detail)

```mermaid
sequenceDiagram
    participant Evt as Domain Event
    participant Lst as Event Listener
    participant NS as NotificationService
    participant RS as NotificationRoutingService
    participant NL as NotificationLog
    participant Q as Queue
    participant Ext as External Service

    Evt->>Lst: Event dispatched (e.g., BookingConfirmed)
    Lst->>NS: sendBookingConfirmedNotification(booking)

    par Email
        NS->>RS: resolveRecipient(booking, type)
        RS-->>NS: {email, target: booker|passenger}
        NS->>NL: Create log (channel: EMAIL, status: PENDING)
        NS->>Q: Dispatch SendEmailNotification job
        Q->>Ext: SendGridService::sendBookingConfirmation(booking)
        Ext-->>Q: {success, external_id}
        Q->>NL: markAsSent(external_id) or markAsFailed(error)
    and SMS
        NS->>RS: resolveRecipient(booking, type)
        RS-->>NS: {phone, target}
        NS->>NL: Create log (channel: SMS, status: PENDING)
        NS->>Q: Dispatch SendSmsNotification job
        Q->>Ext: TwilioService::sendBookingConfirmation(booking)
        Q->>NL: markAsSent or markAsFailed
    and Push
        NS->>NL: Create log (channel: PUSH, status: PENDING)
        NS->>Q: Dispatch SendPushNotification job
        Q->>Ext: FirebaseService::sendToUser(userId, title, body, data)
        Q->>NL: markAsSent or markAsFailed
    end
```

## Full Event-to-Notification Mapping

### Customer Notifications

| Event / Trigger | Email | SMS | Push | Service Method |
|----------------|-------|-----|------|---------------|
| Quote sent to customer | Yes (`sendQuote`) | Yes (`sendQuoteReady`) | No | `sendQuoteNotification()` |
| Booking received (payment confirmed) | Yes (`sendPaymentConfirmation`) | Yes | No | `sendBookingReceivedNotification()` |
| Booking confirmed | Yes (`sendBookingConfirmation`) | Yes | Yes | `sendBookingConfirmedNotification()` |
| Booking cancelled | Yes (`sendBookingCancellation`) | Yes (`sendBookingCancelledSms`) | Yes | `sendBookingCancelledNotification()` |
| Booking rejected | Yes (`sendBookingRejection`) | Yes (`sendBookingRejectedSms`) | No | `sendBookingRejectedNotification()` |
| Driver assigned | Yes (`sendDriverAssigned`) | Yes | Yes | `sendDriverAssignedNotification()` |
| Driver on the way | No | Yes (`sendDriverOnTheWay` + ETA) | No | `sendDriverOnWayNotification()` |
| Driver arrived | No | Yes (`sendDriverArrived`) | No | `sendDriverArrivedNotification()` |
| Payment received | Yes (`sendPaymentConfirmation`) | Yes | No | `sendPaymentReceivedNotification()` |
| Refund processed | Yes (`sendRefundProcessed`) | No | No | `sendRefundProcessedNotification()` |
| Booking reminder | Yes (`sendBookingReminder`) | Yes (`sendBookingReminderSms`) | Yes | `sendBookingReminderNotification()` |
| Modification approved | Yes (`sendModificationApproved`) | No | No | `sendModificationApprovedNotification()` |
| Modification rejected | Yes (`sendModificationRejected`) | No | No | `sendModificationRejectedNotification()` |

### Driver Notifications

| Event / Trigger | Email | SMS | Push | Service Method |
|----------------|-------|-----|------|---------------|
| Job offer broadcast | No | No | Yes | `sendJobOfferNotification()` |
| Job accepted confirmation | No | No | Yes | `sendJobAcceptedConfirmation()` |
| Job taken by another driver | No | No | Yes | `sendJobTakenNotification()` |
| Job offer cancelled | No | No | Yes | `sendJobOfferCancelledNotification()` |
| Job assigned (manual/auto) | Yes (if enabled) | No | Yes | `sendJobAssignedToDriverNotification()` |

### Admin Notifications

| Event / Trigger | Email | SMS | Push | Service Method |
|----------------|-------|-----|------|---------------|
| New quote request | Yes (`sendNewQuoteToAdmin`) | No | No | `sendQuoteRequestNotification()` |
| No drivers available | No | No | Yes | `sendAdminNoDriversNotification()` |
| Customer message reply | No | No | Yes | `sendCustomerReplyNotification()` |

### Invoice Notifications

| Event / Trigger | Email | SMS | Push | Service Method |
|----------------|-------|-----|------|---------------|
| Invoice sent | Yes (with PDF attachment) | No | No | `sendInvoiceNotification()` |
| Invoice overdue | Yes | No | No | `sendInvoiceOverdueNotification()` |

### Corporate CC Notifications

| Event / Trigger | Email | SMS | Push | Service Method |
|----------------|-------|-----|------|---------------|
| Booking confirmed (CC to company) | Yes | No | No | `sendBookingConfirmedCcNotification()` |

## Notification Routing

The `NotificationRoutingService` determines **who** receives each notification:

```mermaid
flowchart TD
    A[resolveRecipient booking, type] --> B{Operator routing mode?}
    B -->|booker_receives_all| C[Return booker email/phone]
    B -->|passenger_receives_updates| D{Is update notification?}
    D -->|Yes| E[Return passenger email/phone]
    D -->|No| F[Return booker email/phone]
    B -->|default| G[Return booker email/phone]
```

- **Booker** = `booking.user` (the person who made the booking)
- **Passenger** = `booking.passenger_email` / `booking.passenger_phone` (the person riding)

This matters for "booking for others" scenarios where booker and passenger are different.

## Email Template System

Operators can customize email templates per notification type:

```mermaid
sequenceDiagram
    participant Admin as Operator Admin
    participant API as Admin API
    participant ETS as EmailTemplateService

    Admin->>API: POST /admin/email-templates
    API->>ETS: createTemplate(data, operatorId)
    ETS->>ETS: Create EmailTemplate record
    Note over ETS: Fields: name, template_type, subject, body_html, body_text, is_active

    Note over ETS: When sending email:
    ETS->>ETS: Check for operator-specific template
    alt Custom template exists
        ETS->>ETS: Render with BookingVariableResolver
        Note over ETS: Variables: {{booking_reference}}, {{customer_name}}, etc.
    else No custom template
        ETS->>ETS: Use default system template
    end
```

**Template types** match notification types (e.g., `booking_confirmed`, `quote_sent`, `driver_assigned`).

## Push Notification via Firebase

```mermaid
sequenceDiagram
    participant NS as NotificationService
    participant NL as NotificationLog
    participant Job as SendPushNotification
    participant FS as FirebaseService
    participant DT as DeviceToken Model
    participant FCM as Firebase Cloud Messaging

    NS->>NL: Create log (channel: PUSH)
    NS->>Job: Dispatch(logId, userId, title, body, data)

    Job->>DT: Query device_tokens WHERE user_id = X
    loop Each device token
        Job->>FS: sendNotification(token, title, body, data)
        FS->>FCM: POST to FCM API
        FCM-->>FS: Success or failure
    end

    Job->>NL: Update status (SENT or FAILED)
```

**Push notification data payload:**

| Field | Description |
|-------|-------------|
| `booking_id` | Booking ID for deep linking |
| `type` | Notification type (e.g., `job_offer`, `booking_confirmed`) |
| `job_offer_id` | Job offer ID (for offer notifications) |
| `message_id` | Message ID (for reply notifications) |

## SMS via Twilio

```mermaid
sequenceDiagram
    participant NS as NotificationService
    participant Job as SendSmsNotification
    participant TS as TwilioService
    participant Twilio as Twilio API

    NS->>Job: Dispatch(logId, method, params)
    Job->>TS: $method(booking, ...params)
    TS->>Twilio: Send SMS to customer phone
    Twilio-->>TS: {success, sid}
    Job->>Job: Update NotificationLog
```

## Notification Preferences

Customers can configure notification preferences via settings:

| Preference | Description |
|-----------|-------------|
| Email notifications | Enable/disable all email notifications |
| SMS notifications | Enable/disable all SMS notifications |
| Push notifications | Enable/disable all push notifications |
| Booking reminders | Enable/disable reminder notifications |

Drivers have a separate `email_notifications_enabled` flag on their Driver model.

## NotificationType Enum Values

| Category | Types |
|----------|-------|
| Quote | `quote_requested`, `quote_sent` |
| Booking | `booking_received`, `booking_confirmed`, `booking_rejected`, `booking_cancelled` |
| Driver | `driver_assigned`, `driver_on_way`, `driver_arrived` |
| Payment | `payment_received`, `payment_failed`, `refund_processed` |
| Invoice | `invoice_sent`, `invoice_paid`, `invoice_overdue` |
| Push-specific | `booking_reminder`, `job_assigned`, `job_offer` |
| Admin | `no_drivers_available` |
| Modification | `modification_approved`, `modification_rejected` |
| Messaging | `booking_message`, `customer_reply` |

## NotificationLog Model

| Field | Description |
|-------|-------------|
| `user_id` | Recipient user ID |
| `booking_id` | Related booking (nullable) |
| `channel` | `email`, `sms`, `push` |
| `type` | NotificationType enum value |
| `recipient` | Email address, phone number, or `push:{user_id}` |
| `subject` | Email subject or push title |
| `status` | `pending`, `sent`, `failed` |
| `external_id` | External service reference (SendGrid ID, Twilio SID) |
| `error` | Error message if failed |

## Key Files

| Purpose | File |
|---------|------|
| Notification service | `app/Notification/Services/NotificationService.php` |
| Routing service | `app/Notification/Services/NotificationRoutingService.php` |
| Email template service | `app/Notification/Services/EmailTemplateService.php` |
| Variable resolver | `app/Notification/Services/BookingVariableResolver.php` |
| NotificationType enum | `app/Notification/Enums/NotificationType.php` |
| NotificationChannel enum | `app/Notification/Enums/NotificationChannel.php` |
| NotificationStatus enum | `app/Notification/Enums/NotificationStatus.php` |
| NotificationLog model | `app/Notification/Models/NotificationLog.php` |
| DeviceToken model | `app/Notification/Models/DeviceToken.php` |
| EmailTemplate model | `app/Notification/Models/EmailTemplate.php` |
| Send email job | `app/Notification/Jobs/SendEmailNotification.php` |
| Send SMS job | `app/Notification/Jobs/SendSmsNotification.php` |
| Send push job | `app/Notification/Jobs/SendPushNotification.php` |
| SendGrid service | `app/Services/External/SendGridService.php` |
| Twilio service | `app/Services/External/TwilioService.php` |
| Firebase service | `app/Services/External/FirebaseService.php` |
