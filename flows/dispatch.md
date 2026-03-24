---
title: "Dispatch Flow"
description: "Manual, offer, auto and tiered driver dispatch modes."
---

# Dispatch Flow

All dispatch modes for assigning drivers to bookings. Supports manual assignment, offer broadcast, sequential (hold timer), auto-dispatch (nearest driver), and tiered cascade (OWN -> PRIVATE -> PUBLIC).

## Actors

- **Operator Admin** — initiates dispatch, manually assigns drivers
- **Driver** — receives offers, accepts/rejects
- **System** — auto-dispatch, offer expiry, reassignment

## Entry Points

| Channel | URL | Controller |
|---------|-----|------------|
| Manual assign | `PATCH /api/v1/admin/bookings/{id}` | `Api\Admin\BookingController::update()` |
| Offer broadcast | `POST /api/v1/admin/bookings/{id}/offer-dispatch` | `Api\Admin\BookingController::offerDispatch()` |
| Auto dispatch | `POST /api/v1/admin/bookings/{id}/auto-dispatch` | `Api\Admin\BookingController::autoDispatch()` |
| Accept offer | `POST /api/v1/driver/job-offers/{id}/accept` | `Api\Driver\JobOfferController::accept()` |
| Reject offer | `POST /api/v1/driver/job-offers/{id}/reject` | `Api\Driver\JobOfferController::reject()` |
| Expire offers | Scheduled command | `ExpireJobOffers` job |

## Dispatch Mode Overview

| Mode | When | How |
|------|------|-----|
| **Manual** | Admin picks a driver | Direct assignment, no offers |
| **Broadcast** | Pre-scheduled bookings | All drivers in group notified simultaneously |
| **Sequential** | ASAP bookings | One driver at a time with hold timer |
| **Auto** | Nearest driver needed | System finds and assigns nearest available |
| **Tiered** | Cascade through groups | OWN -> PRIVATE -> PUBLIC tiers in order |
| **Smart** | Auto-selects mode | Sequential for ASAP, broadcast for scheduled |

---

## 1. Manual Assignment

```mermaid
sequenceDiagram
    participant Admin as Operator Admin
    participant API as Admin API
    participant BS as BookingService
    participant ER as BookingEventRecorder

    Admin->>API: PATCH /admin/bookings/{id} {driver_id, vehicle_id}
    API->>BS: assignDriver()
    BS->>BS: lockForUpdate() on booking
    BS->>BS: Auto-detect supplier from driver
    BS->>ER: recordStatusChange(DRIVER_ASSIGNED)
    ER->>ER: Update booking status + driver_id + vehicle_id
    ER->>ER: Record timeline event
    ER->>ER: Dispatch DriverAssigned event
    API-->>Admin: status: driver_assigned
```

---

## 2. Offer Broadcast (to Group)

All drivers in the group receive the offer simultaneously. First to accept wins.

```mermaid
sequenceDiagram
    participant Admin as Operator Admin
    participant API as Admin API
    participant ODS as OfferDispatchService
    participant D1 as Driver 1
    participant D2 as Driver 2
    participant D3 as Driver 3

    Admin->>API: POST /admin/bookings/{id}/offer-dispatch {driver_group_id}
    API->>ODS: broadcastToGroup(booking, group)

    ODS->>ODS: Create JobOffer (mode: broadcast, status: pending)
    ODS->>ODS: Create DriverJobOffer per available driver (status: pending)

    par Notify all drivers
        ODS->>D1: JobOfferBroadcast event (push notification)
        ODS->>D2: JobOfferBroadcast event (push notification)
        ODS->>D3: JobOfferBroadcast event (push notification)
    end

    API-->>Admin: JobOffer with driver_count

    alt Driver 1 accepts first
        D1->>API: POST /driver/job-offers/{id}/accept
        API->>ODS: acceptOffer(driverOffer, driver)
        ODS->>ODS: lockForUpdate() on DriverJobOffer + JobOffer
        ODS->>ODS: Verify offer and job still pending
        ODS->>ODS: Accept driver offer + job offer
        ODS->>ODS: Reject all other pending driver offers
        ODS->>ODS: Assign driver to booking via BookingEventRecorder
        ODS->>D2: JobTaken notification
        ODS->>D3: JobTaken notification
    else All drivers reject
        D1->>API: POST /driver/job-offers/{id}/reject
        D2->>API: POST /driver/job-offers/{id}/reject
        D3->>API: POST /driver/job-offers/{id}/reject
        ODS->>ODS: pendingCount === 0 → expire JobOffer
        ODS->>ODS: handleReassignment() if auto_reassign enabled
    end
```

---

## 3. Sequential Dispatch (Hold Timer)

Offers to one driver at a time. If the hold timer expires or driver rejects, advances to next.

```mermaid
sequenceDiagram
    participant ODS as OfferDispatchService
    participant D1 as Driver 1
    participant D2 as Driver 2
    participant D3 as Driver 3

    ODS->>ODS: Create JobOffer (mode: sequential, current_driver_index: 0)
    ODS->>ODS: Create DriverJobOffer: D1=pending, D2=waiting, D3=waiting
    ODS->>D1: JobOfferBroadcast (notify first driver only)

    alt Driver 1 accepts
        D1->>ODS: acceptOffer()
        ODS->>ODS: Assign driver, reject D2/D3 offers
    else Driver 1 rejects or hold timer expires
        ODS->>ODS: advanceSequentialDispatch()
        ODS->>ODS: D1 offer → expired, D2 offer → pending
        ODS->>ODS: Update current_driver_index = 1, reset hold_expires_at
        ODS->>D2: JobOfferBroadcast (notify next driver)

        alt Driver 2 accepts
            D2->>ODS: acceptOffer()
        else Driver 2 rejects or expires
            ODS->>ODS: Advance to D3 (same pattern)
            alt Driver 3 also declines
                ODS->>ODS: No more drivers → expire JobOffer
                ODS->>ODS: handleReassignment() if auto_reassign
            end
        end
    end
```

**Hold timer:** Configured per driver group via `hold_timer_minutes` (default: 2 minutes).

**Offer timeout:** Configured per group via `offer_timeout_minutes` (default: 5 minutes).

---

## 4. Auto Dispatch (Nearest Driver)

System finds the nearest available driver using Haversine distance calculation and assigns directly.

```mermaid
sequenceDiagram
    participant Admin as Operator Admin
    participant API as Admin API
    participant ADS as AutoDispatchService
    participant ER as BookingEventRecorder

    Admin->>API: POST /admin/bookings/{id}/auto-dispatch
    API->>ADS: dispatch(booking)
    ADS->>ADS: Validate booking has pickup coordinates
    ADS->>ADS: Validate booking status = confirmed

    ADS->>ADS: getAvailableDriversWithDistance()
    Note over ADS: Query drivers: is_active, is_verified, status=available, has coordinates
    ADS->>ADS: Calculate Haversine distance for each driver
    ADS->>ADS: Sort by distance (nearest first)
    ADS->>ADS: Optional: filter by maxDistanceKm

    ADS->>ADS: Select nearest driver
    ADS->>ER: recordStatusChange(DRIVER_AUTO_ASSIGNED, DRIVER_ASSIGNED)
    ADS->>ADS: driver.setBusy()
    ADS->>ADS: Dispatch DriverAutoAssigned event
    API-->>Admin: {driver_name, distance_km, drivers_checked}
```

**Driver availability criteria:**
- `is_active = true`
- `is_verified = true`
- `status = 'available'`
- `current_latitude` and `current_longitude` not null

---

## 5. Tiered Dispatch (OWN -> PRIVATE -> PUBLIC)

Three-tier cascade through driver groups ordered by tier and priority.

```mermaid
flowchart TD
    A[Start Tiered Dispatch] --> B{OWN tier groups?}
    B -->|Yes| C[Get available drivers in OWN groups]
    C -->|Drivers found| D[Broadcast to group]
    C -->|No drivers| E{PRIVATE tier groups?}
    B -->|No groups| E

    E -->|Yes| F[Get available drivers in PRIVATE groups]
    F -->|Drivers found| G[Broadcast to group]
    F -->|No drivers| H{PUBLIC tier groups?}
    E -->|No groups| H

    H -->|Yes| I[Get available drivers in PUBLIC groups]
    I -->|Drivers found| J[Broadcast to group]
    I -->|No drivers| K[No drivers available]
    H -->|No groups| K

    D --> L[Wait for acceptance]
    G --> L
    J --> L
```

```mermaid
sequenceDiagram
    participant ODS as OfferDispatchService
    participant DB as Database

    ODS->>DB: Query DriverGroups for operator (tier: own, ordered by priority)
    loop Each OWN group
        ODS->>DB: getAvailableDrivers()
        alt Drivers found
            ODS->>ODS: broadcastToGroup() or dispatchSequential()
            Note over ODS: Done - waiting for acceptance
        end
    end

    Note over ODS: No OWN drivers available

    ODS->>DB: Query DriverGroups (tier: private, ordered by priority)
    loop Each PRIVATE group
        ODS->>DB: getAvailableDrivers()
        alt Drivers found
            ODS->>ODS: broadcastToGroup()
        end
    end

    Note over ODS: No PRIVATE drivers available

    ODS->>DB: Query DriverGroups (tier: public, ordered by priority)
    loop Each PUBLIC group
        ODS->>DB: getAvailableDrivers()
        alt Drivers found
            ODS->>ODS: broadcastToGroup()
        end
    end
```

**Tier enum values:** `own` (order 1), `private` (order 2), `public` (order 3)

Groups within each tier are ordered by `priority ASC`.

---

## 6. Reassignment Cascade

When all drivers in a group decline (or offers expire), the system automatically tries the next group.

```mermaid
flowchart TD
    A[All drivers declined / expired] --> B{auto_reassign enabled?}
    B -->|No| Z[Notify admin: no drivers]
    B -->|Yes| C[handleReassignment]
    C --> D[Get tried group IDs from parent chain]
    D --> E[Find remaining groups ordered by tier + priority]
    E --> F{Group with available drivers?}
    F -->|Yes| G[broadcastToGroup with parentOffer link]
    F -->|No more groups| H[Try AutoDispatchService as fallback]
    H -->|Success| I[Driver auto-assigned]
    H -->|Failure| J[NoDriversAvailable event]
    J --> K[Admin push notification: No drivers available]
```

**Parent chain:** Job offers link via `parent_job_offer_id` to track which groups have already been tried.

## Smart Dispatch

`smartDispatch()` auto-selects the dispatch mode:
- **ASAP bookings** (pickup within ~1 hour): sequential dispatch (hold timer)
- **Scheduled bookings**: broadcast to group

## Driver Group Model

| Field | Description |
|-------|-------------|
| `name` | Group name |
| `tier` | `own`, `private`, `public` |
| `priority` | Sort order within tier (lower = tried first) |
| `offer_timeout_minutes` | How long the overall offer stays open (default: 5) |
| `hold_timer_minutes` | Per-driver hold time in sequential mode (default: 2) |
| `is_shared` | Whether this group is shared across operators |
| `is_active` | Whether this group is available for dispatch |

## Events Fired

| Event | When | Listeners |
|-------|------|-----------|
| `JobOfferBroadcast` | Offer sent to a driver | `SendJobOfferToDrivers` (push notification) |
| `JobOfferAccepted` | Driver accepts an offer | `SendJobAcceptedConfirmation` |
| `JobOfferCancelled` | Admin cancels an offer | `SendJobOfferCancelledNotification` |
| `DriverAssigned` | Driver assigned to booking | `SendDriverAssignedNotification` |
| `DriverAutoAssigned` | Auto-dispatch assigns driver | Notification to customer and driver |
| `NoDriversAvailable` | All dispatch options exhausted | `SendAdminNoDriversNotification` |

## Key Files

| Purpose | File |
|---------|------|
| Offer dispatch service | `app/Dispatch/Services/OfferDispatchService.php` |
| Auto dispatch service | `app/Dispatch/Services/AutoDispatchService.php` |
| Driver job service | `app/Dispatch/Services/DriverJobService.php` |
| Dispatch tier enum | `app/Dispatch/Enums/DispatchTier.php` |
| JobOffer model | `app/Dispatch/Models/JobOffer.php` |
| DriverJobOffer model | `app/Dispatch/Models/DriverJobOffer.php` |
| DriverGroup model | `app/Driver/Models/DriverGroup.php` |
| Expire offers job | `app/Dispatch/Jobs/ExpireJobOffers.php` |
| Events | `app/Dispatch/Events/` |
| Listeners | `app/Dispatch/Listeners/` |
