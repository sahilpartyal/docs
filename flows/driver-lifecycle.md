---
title: "Driver Lifecycle"
description: "Driver onboarding, activation, jobs and offboarding."
---

# Driver Job Lifecycle

Complete lifecycle of a driver's interaction with a booking, from receiving an offer to completing a trip.

## Actors

- **Driver** — receives offers, updates trip status
- **Customer** — receives status notifications
- **System** — manages offer expiry, releases driver status

## Entry Points

| Channel | URL | Controller |
|---------|-----|------------|
| Pending offers | `GET /api/v1/driver/job-offers` | `Api\Driver\JobOfferController::index()` |
| Accept offer | `POST /api/v1/driver/job-offers/{id}/accept` | `Api\Driver\JobOfferController::accept()` |
| Reject offer | `POST /api/v1/driver/job-offers/{id}/reject` | `Api\Driver\JobOfferController::reject()` |
| Today's jobs | `GET /api/v1/driver/jobs/today` | `Api\Driver\JobController::today()` |
| Active job | `GET /api/v1/driver/jobs/active` | `Api\Driver\JobController::active()` |
| Update status | `POST /api/v1/driver/jobs/{id}/status` | `Api\Driver\JobController::updateStatus()` |
| Job detail | `GET /api/v1/driver/jobs/{id}` | `Api\Driver\JobController::show()` |

## Full Job Lifecycle

```mermaid
stateDiagram-v2
    [*] --> offer_pending: Job offer received
    offer_pending --> offer_accepted: Driver accepts
    offer_pending --> offer_rejected: Driver rejects
    offer_pending --> offer_expired: Hold timer / timeout

    offer_accepted --> driver_assigned: Booking updated
    driver_assigned --> driver_on_way: Driver starts (on_way)
    driver_on_way --> driver_arrived: Driver arrives (arrived)
    driver_arrived --> passenger_on_board: Passenger picked up (passenger_on_board)
    driver_arrived --> no_show: Passenger did not show (no_show)
    passenger_on_board --> completed: Trip finished (completed)

    completed --> [*]
    no_show --> [*]
```

## Receiving and Viewing Offers

```mermaid
sequenceDiagram
    participant Sys as Dispatch System
    participant D as Driver (App/Portal)
    participant API as Driver API
    participant ODS as OfferDispatchService

    Sys->>D: Push notification: "New Job Offer"
    Note over D: Notification includes pickup date/time, expiry countdown

    D->>API: GET /driver/job-offers
    API->>ODS: getPendingOffersForDriver(driver)
    ODS-->>API: offers with booking details, group info
    API-->>D: List of pending offers

    Note over D: Each offer shows:
    Note over D: - Pickup/dropoff addresses
    Note over D: - Pickup date/time
    Note over D: - Vehicle type
    Note over D: - Expiry countdown
```

## Accept / Reject Flow

```mermaid
sequenceDiagram
    participant D as Driver
    participant API as Driver API
    participant ODS as OfferDispatchService
    participant ER as BookingEventRecorder
    participant NS as NotificationService

    alt Accept
        D->>API: POST /driver/job-offers/{id}/accept
        API->>ODS: acceptOffer(driverOffer, driver)
        ODS->>ODS: lockForUpdate() on DriverJobOffer
        ODS->>ODS: Verify driver_id matches
        ODS->>ODS: Verify offer is still pending (after lock)
        ODS->>ODS: lockForUpdate() on JobOffer
        ODS->>ODS: Verify job offer still pending
        ODS->>ODS: Verify booking can transition to driver_assigned
        ODS->>ODS: driverOffer.accept()
        ODS->>ODS: jobOffer.accept(driver)
        ODS->>ODS: Reject all other pending/waiting driver offers
        ODS->>ER: recordStatusChange(DRIVER_ASSIGNED)
        ER->>ER: Update booking: driver_id, vehicle_id, supplier_id, status
        ODS->>ODS: Dispatch JobOfferAccepted event
        NS->>D: Push: "Job Accepted" confirmation
        NS->>NS: Notify customer: driver assigned
        API-->>D: success

    else Reject
        D->>API: POST /driver/job-offers/{id}/reject
        API->>ODS: rejectOffer(driverOffer, driver)
        ODS->>ODS: lockForUpdate() on DriverJobOffer
        ODS->>ODS: driverOffer.reject()

        alt Sequential mode
            ODS->>ODS: advanceSequentialDispatch() → notify next driver
        else Broadcast mode
            ODS->>ODS: Check remaining pending count
            alt All rejected
                ODS->>ODS: jobOffer.expire()
                ODS->>ODS: handleReassignment() if auto_reassign
            end
        end
        API-->>D: success
    end
```

## Status Updates (Trip Progress)

```mermaid
sequenceDiagram
    participant D as Driver
    participant API as Driver API
    participant DJS as DriverJobService
    participant ER as BookingEventRecorder
    participant NS as NotificationService

    D->>API: POST /driver/jobs/{id}/status {status: "on_way", latitude, longitude}
    API->>DJS: updateJobStatus(booking, "on_way", notes, lat, lng)

    DJS->>DJS: Map "on_way" → BookingStatus::DRIVER_ON_WAY
    DJS->>DJS: Validate transition: driver_assigned → driver_on_way (allowed)
    DJS->>DJS: Map status → BookingEventType::DRIVER_ON_WAY
    DJS->>ER: recordStatusChange(DRIVER_ON_WAY, bookingUpdates: {driver_latitude, driver_longitude})

    DJS->>NS: sendDriverOnWayNotification(booking, etaMinutes: 15)
    NS->>NS: SMS to customer: "Driver on the way, ETA 15 min"
    API-->>D: "Status updated: On the way to pickup"

    D->>API: POST /driver/jobs/{id}/status {status: "arrived"}
    DJS->>ER: recordStatusChange(DRIVER_ARRIVED)
    DJS->>NS: sendDriverArrivedNotification()
    NS->>NS: SMS to customer: "Driver arrived"

    D->>API: POST /driver/jobs/{id}/status {status: "passenger_on_board"}
    DJS->>ER: recordStatusChange(PASSENGER_ON_BOARD)

    D->>API: POST /driver/jobs/{id}/status {status: "completed"}
    DJS->>ER: recordStatusChange(TRIP_COMPLETED)
    DJS->>DJS: releaseDriver() — set driver status back to 'available'
```

## Allowed Status Transitions

| Current Status | Allowed Next Status |
|---------------|-------------------|
| `driver_assigned` | `driver_on_way`, `cancelled` |
| `driver_on_way` | `driver_arrived`, `cancelled` |
| `driver_arrived` | `passenger_on_board`, `no_show`, `cancelled` |
| `passenger_on_board` | `completed` |

## Status Input Mapping

The driver API accepts short status names that map to `BookingStatus` enum values:

| Driver Input | BookingStatus | BookingEventType |
|-------------|---------------|-----------------|
| `on_way` | `DRIVER_ON_WAY` | `DRIVER_ON_WAY` |
| `arrived` | `DRIVER_ARRIVED` | `DRIVER_ARRIVED` |
| `passenger_on_board` | `PASSENGER_ON_BOARD` | `PASSENGER_ON_BOARD` |
| `completed` | `COMPLETED` | `TRIP_COMPLETED` |
| `no_show` | `NO_SHOW` | `NO_SHOW` |

## No-Show Handling

```mermaid
sequenceDiagram
    participant D as Driver
    participant API as Driver API
    participant DJS as DriverJobService

    Note over D: Driver has status: driver_arrived
    Note over D: Passenger does not appear

    D->>API: POST /driver/jobs/{id}/status {status: "no_show", notes: "Waited 15 min"}
    API->>DJS: updateJobStatus(booking, "no_show", notes)
    DJS->>DJS: Validate: driver_arrived → no_show (allowed)
    DJS->>DJS: recordStatusChange(NO_SHOW)
    DJS->>DJS: releaseDriver()
    Note over DJS: Driver status → available (if no other active bookings)
    API-->>D: "Status updated: Passenger no-show"
```

## Active Job Tracking

```mermaid
sequenceDiagram
    participant D as Driver
    participant API as Driver API
    participant DJS as DriverJobService

    D->>API: GET /driver/jobs/active
    API->>DJS: getActiveJob(driverId)
    DJS->>DJS: Query bookings where driver_id = X
    DJS->>DJS: Status IN (driver_on_way, driver_arrived, passenger_on_board)
    DJS-->>API: Active booking with customer, vehicle, operator details
    API-->>D: Active job (or null)
```

## Driver Release Logic

On terminal statuses (`completed`, `no_show`):

1. Lock driver row (`lockForUpdate`)
2. Check if driver has other active bookings (`driver_assigned`, `driver_on_way`, `driver_arrived`, `passenger_on_board`)
3. If no other active bookings: set `driver.status = 'available'`
4. If other active bookings remain: keep `driver.status = 'busy'`

## Customer Notifications Per Status

| Status | Email | SMS | Push |
|--------|-------|-----|------|
| `driver_assigned` | Yes | Yes | Yes |
| `driver_on_way` | No | Yes (with ETA) | No |
| `driver_arrived` | No | Yes | No |
| `passenger_on_board` | No | No | No |
| `completed` | No | No | No |
| `no_show` | No | No | No |

## Events Fired

| Event Type | When |
|------------|------|
| `DRIVER_ON_WAY` | Driver starts heading to pickup |
| `DRIVER_ARRIVED` | Driver reaches pickup location |
| `PASSENGER_ON_BOARD` | Passenger picked up, trip starts |
| `TRIP_COMPLETED` | Trip finished successfully |
| `NO_SHOW` | Passenger did not show up |

## Key Files

| Purpose | File |
|---------|------|
| Driver job service | `app/Dispatch/Services/DriverJobService.php` |
| Offer dispatch service | `app/Dispatch/Services/OfferDispatchService.php` |
| Job offer controller (driver) | `app/Http/Controllers/Api/Driver/JobOfferController.php` |
| Job controller (driver) | `app/Http/Controllers/Api/Driver/JobController.php` |
| Booking event recorder | `app/Booking/Services/BookingEventRecorder.php` |
| Notification service | `app/Notification/Services/NotificationService.php` |
| BookingStatus enum | `app/Booking/Enums/BookingStatus.php` |
| BookingEventType enum | `app/Booking/Enums/BookingEventType.php` |
