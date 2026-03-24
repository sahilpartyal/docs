---
title: "Overview"
description: "Living documentation of every major process in the Here2There platform."
---

# Here2There — Flow Documentation

Living documentation of every major process in the platform. Updated automatically with each feature, fix, or flow change.

> **Powered by Mintlify** — rendered at your docs site. Maintained by Claude Code via post-feature hook.

## Booking Flows

| Flow | Description | Entry Points |
|------|-------------|--------------|
| [Public Booking](booking-public.md) | Customer books directly with instant pricing | SPA, Mobile App |
| [Quote Request](quote-flow.md) | Customer requests quote, admin prices, customer selects | SPA, Widget |
| [Admin Booking](booking-admin.md) | Operator admin creates booking (phone/walk-in) | Admin Portal |
| [Widget Booking](booking-widget.md) | Embedded booking widget on partner sites | Preact Widget, Blade Widget |
| [Corporate Booking](booking-corporate.md) | Corporate portal — employee self-service + admin booking | Corporate Portal |

## Operations

| Flow | Description |
|------|-------------|
| [Dispatch](dispatch.md) | Driver assignment — manual, offer, sequential, auto, tiered |
| [Driver Lifecycle](driver-lifecycle.md) | Job offers → accept → status updates → complete |
| [Driver Earnings](driver-earnings.md) | Commission calculation, bonuses, deductions, payouts |

## Payments & Billing

| Flow | Description |
|------|-------------|
| [Payment Processing](payment.md) | Stripe intents, saved cards, refunds, webhooks |
| [Invoicing](invoicing.md) | Manual + auto invoices, recurring schedules, reminders |
| [Corporate Billing](corporate-billing.md) | Prepaid balance, auto-charge, monthly invoicing |

## Platform

| Flow | Description |
|------|-------------|
| [Operator Registration](operator-registration.md) | Signup → plan → Stripe → widget provisioning |
| [Notifications](notifications.md) | Email/SMS/push triggers, templates, routing |

## Keeping Docs Current

These docs are maintained via two mechanisms:

1. **CLAUDE.md rule** — Claude Code updates relevant flow docs after every feature
2. **Post-commit hook** — Reminds to update docs when `app/` or `routes/` files change

To manually refresh all docs:
```
Update the flow docs to match the current codebase
```
