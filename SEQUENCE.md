# 2. Sequence Diagrams

Reservation system for additional spaces (UC2, UC3, UC4 from `INFRA.md` / `README.md`).

- **Use case U2:** UC2 – Cancel a reservation with partial refund
- **Use case U3:** UC3 – QR code check-in
- **Use case U4:** UC4 – Register a guest

## 2.0 Reference class diagram (names used by the sequences)

Entities and attributes come from `INFRA.md`. Every method that appears in a sequence diagram is declared here, so names stay consistent across all diagrams. The three `*Service` / `AccessScanner` classes are the only additions: they coordinate the flow so the entities keep their own rules (cancel, validate, add guest).

```mermaid
classDiagram
    class User {
        +String id
        +String name
        +String email
        +String phone
        +String role
    }
    class Space {
        +String id
        +String name
        +String type
        +int capacity
        +decimal hourly_rate
        +String cancellation_policy
        +boolean is_active
        +getCapacity() int
        +getRefundPercentage(notice_hours) decimal
        +releaseSlot(start_time, end_time) void
    }
    class Reservation {
        +String id
        +String user_id
        +String space_id
        +DateTime start_time
        +DateTime end_time
        +String status
        +decimal total_amount
        +isCancellable(now) boolean
        +getNoticeHours(now) int
        +cancel() void
        +getPayment() Payment
        +canCheckIn(now) boolean
        +markCheckedIn() void
        +getGuestCount() int
        +canAddGuest() boolean
        +addGuest(guest) void
    }
    class Payment {
        +String id
        +String reservation_id
        +decimal amount
        +String payment_method
        +String status
        +DateTime created_at
    }
    class Refund {
        +String id
        +String reservation_id
        +decimal original_amount
        +decimal penalty_amount
        +decimal refund_amount
        +String status
        +DateTime processed_at
        +calculate(reservation, refund_percentage) Refund$
        +markProcessed() void
        +markFailed() void
    }
    class QRCode {
        +String id
        +String reservation_id
        +String qr_token
        +DateTime expiration_time
        +String status
        +getReservation() Reservation
        +isValid(now) boolean
        +markUsed() void
        +invalidate() void
    }
    class CheckIn {
        +String id
        +String reservation_id
        +DateTime scanned_at
        +String scanned_by
        +String status
        +record(reservation, scanned_by, status) CheckIn$
    }
    class Guest {
        +String id
        +String reservation_id
        +String full_name
        +String id_document
        +String email
        +String access_status
        +register(reservation, full_name, id_document, email) Guest$
    }
    class ReservationService {
        +cancelReservation(reservation_id) Refund
        +registerGuest(reservation_id, full_name, id_document, email) Guest
    }
    class AccessControlService {
        +processCheckIn(qr_token, scanner_id) AccessResult
        +findQRCodeByToken(qr_token) QRCode
    }
    class AccessScanner {
        +String scanner_id
        +scanQR(qr_token) void
        +unlock() void
        +deny(reason) void
    }
    class AccessResult {
        +boolean granted
        +String reason
    }
    class PaymentGateway {
        <<interface>>
        +processRefund(payment, refund_amount) RefundResult
    }
    class NotificationService {
        <<interface>>
        +sendRefundBreakdown(user_id, refund) void
        +sendGuestNotification(guest) void
    }

    User "1" --> "0..*" Reservation : makes
    Space "1" --> "0..*" Reservation : is booked in
    Reservation "1" --> "1" Payment : paid by
    Reservation "1" --> "0..1" Refund : may generate
    Reservation "1" --> "1" QRCode : grants access with
    Reservation "1" --> "0..*" CheckIn : records attempts
    Reservation "1" *-- "0..*" Guest : authorizes

    ReservationService ..> Reservation
    ReservationService ..> Space
    ReservationService ..> PaymentGateway
    ReservationService ..> NotificationService
    AccessControlService ..> QRCode
    AccessControlService ..> Reservation
    AccessScanner ..> AccessControlService
```

Two small changes against `INFRA.md` (to be discussed with the team):

1. `Reservation` to `CheckIn` is **1 : 0..\*** instead of 1 : 0..1, because UC3 step 4 logs a `FAILED` check-in for every rejected scan, so one reservation can have several attempts.
2. `Guest` gets an `email` attribute, because UC4 step 2 captures it and step 5 uses it to notify the guest.

---

## 2.1 UC2 – Cancel a reservation with partial refund

**Flow:** the user asks to cancel. The system checks that the reservation can still be cancelled, looks up the refund percentage in the space's cancellation policy, creates the `Refund` (penalty + refund amount), cancels the reservation (which also invalidates its QR), releases the time slot, asks the payment gateway for the partial refund, and finally notifies the user with the breakdown.

```mermaid
sequenceDiagram
    actor User
    participant ReservationService
    participant Reservation
    participant Space
    participant QRCode
    participant PaymentGateway
    participant NotificationService

    User->>ReservationService: cancelReservation(reservation_id)
    ReservationService->>Reservation: isCancellable(now)
    Reservation-->>ReservationService: boolean

    alt Not cancellable (not CONFIRMED or already started)
        ReservationService-->>User: error: reservation cannot be cancelled
    else Cancellable
        ReservationService->>Reservation: getNoticeHours(now)
        Reservation-->>ReservationService: notice_hours
        ReservationService->>Space: getRefundPercentage(notice_hours)
        Space-->>ReservationService: refund_percentage

        create participant Refund
        ReservationService->>Refund: calculate(reservation, refund_percentage)
        Note right of Refund: sets original_amount, penalty_amount, refund_amount
        Refund-->>ReservationService: refund

        ReservationService->>Reservation: cancel()
        Note right of Reservation: status = CANCELLED
        Reservation->>QRCode: invalidate()
        QRCode-->>Reservation: ok
        Reservation-->>ReservationService: ok

        ReservationService->>Space: releaseSlot(start_time, end_time)
        Space-->>ReservationService: ok

        ReservationService->>Reservation: getPayment()
        Reservation-->>ReservationService: payment
        ReservationService->>PaymentGateway: processRefund(payment, refund_amount)
        PaymentGateway-->>ReservationService: RefundResult

        alt Refund accepted
            ReservationService->>Refund: markProcessed()
        else Refund rejected by gateway
            ReservationService->>Refund: markFailed()
        end

        ReservationService->>NotificationService: sendRefundBreakdown(user_id, refund)
        NotificationService-->>ReservationService: ok
        ReservationService-->>User: refund
    end
```

**Design decisions**

- The percentage rule lives in `Space` (it owns `cancellation_policy`); `Refund.calculate` only does the arithmetic. If the policy changes, only `Space` changes.
- `Reservation.cancel()` invalidates its own `QRCode`, so a cancelled reservation can never keep a valid pass.
- The reservation is cancelled **before** calling the gateway: the user's intent to cancel is not lost if the gateway fails; the `Refund` is just left as `FAILED` for retry.

---

## 2.2 UC3 – QR code check-in

**Flow:** the user presents the QR at the scanner. The scanner delegates validation to `AccessControlService`, which finds the QR by token, checks the pass itself (status and expiration) and the reservation (status `CONFIRMED` and allowed time window). On success it records a `SUCCESS` `CheckIn`, moves the reservation to `CHECKED_IN`, marks the QR as `USED`, and the scanner unlocks the door. On failure it records a `FAILED` `CheckIn` and the scanner denies access.

```mermaid
sequenceDiagram
    actor User
    participant AccessScanner
    participant AccessControlService
    participant QRCode
    participant Reservation
    participant CheckIn

    User->>AccessScanner: scanQR(qr_token)
    AccessScanner->>AccessControlService: processCheckIn(qr_token, scanner_id)
    AccessControlService->>AccessControlService: findQRCodeByToken(qr_token)

    alt Token not found
        AccessControlService-->>AccessScanner: AccessResult(granted=false, reason=UNKNOWN_TOKEN)
    else Token found
        AccessControlService->>QRCode: getReservation()
        QRCode-->>AccessControlService: reservation
        AccessControlService->>QRCode: isValid(now)
        QRCode-->>AccessControlService: qr_valid
        AccessControlService->>Reservation: canCheckIn(now)
        Reservation-->>AccessControlService: reservation_ok

        alt qr_valid and reservation_ok
            create participant CheckIn
            AccessControlService->>CheckIn: record(reservation, scanner_id, SUCCESS)
            CheckIn-->>AccessControlService: check_in
            AccessControlService->>Reservation: markCheckedIn()
            Note right of Reservation: status = CHECKED_IN
            Reservation-->>AccessControlService: ok
            AccessControlService->>QRCode: markUsed()
            QRCode-->>AccessControlService: ok
            AccessControlService-->>AccessScanner: AccessResult(granted=true)
        else Expired, cancelled or out of schedule
            AccessControlService->>CheckIn: record(reservation, scanner_id, FAILED)
            CheckIn-->>AccessControlService: check_in
            AccessControlService-->>AccessScanner: AccessResult(granted=false, reason)
        end
    end

    alt granted
        AccessScanner->>AccessScanner: unlock()
        AccessScanner-->>User: access granted
    else denied
        AccessScanner->>AccessScanner: deny(reason)
        AccessScanner-->>User: access denied
    end
```

**Design decisions**

- `AccessScanner` only handles the hardware (read, unlock, deny); all business rules stay in `AccessControlService`, `QRCode` and `Reservation`.
- Two separate checks: `QRCode.isValid` asks "is this pass good?", `Reservation.canCheckIn` asks "is this the right moment?". Each object answers about its own data.
- An unknown token creates **no** `CheckIn`, because `CheckIn` requires a `reservation_id` and an unknown token has none. The denial is only returned to the scanner.

---

## 2.3 UC4 – Register a guest

**Flow:** the holder submits the guest data for an active reservation. The system checks that the reservation is `CONFIRMED` and still has capacity. If so, it creates the `Guest`, links it to the reservation and optionally notifies the guest.

```mermaid
sequenceDiagram
    actor User
    participant ReservationService
    participant Reservation
    participant Space
    participant NotificationService

    User->>ReservationService: registerGuest(reservation_id, full_name, id_document, email)
    ReservationService->>Reservation: canAddGuest()
    Reservation->>Space: getCapacity()
    Space-->>Reservation: capacity
    Reservation->>Reservation: getGuestCount()
    Note right of Reservation: allowed if status = CONFIRMED and guest_count < capacity
    Reservation-->>ReservationService: boolean

    alt Not allowed (not CONFIRMED or capacity reached)
        ReservationService-->>User: error: guest cannot be registered
    else Allowed
        create participant Guest
        ReservationService->>Guest: register(reservation, full_name, id_document, email)
        Guest-->>ReservationService: guest
        ReservationService->>Reservation: addGuest(guest)
        Reservation-->>ReservationService: ok

        opt Guest email provided
            ReservationService->>NotificationService: sendGuestNotification(guest)
            NotificationService-->>ReservationService: ok
        end

        ReservationService-->>User: guest
    end
```

**Design decisions**

- `Reservation.canAddGuest()` owns the rule (status + capacity); the service does not read `Space` attributes directly, so the rule has a single home.
- The notification is `opt`: the guest pass is an optional step in UC4 and must not block the registration.

---

## 2.4 Consistency check

| Message in sequences | Declared in class diagram |
|---|---|
| `cancelReservation`, `registerGuest` | `ReservationService` |
| `isCancellable`, `getNoticeHours`, `cancel`, `getPayment`, `canCheckIn`, `markCheckedIn`, `canAddGuest`, `getGuestCount`, `addGuest` | `Reservation` |
| `getRefundPercentage`, `releaseSlot`, `getCapacity` | `Space` |
| `calculate`, `markProcessed`, `markFailed` | `Refund` |
| `invalidate`, `isValid`, `markUsed`, `getReservation` | `QRCode` |
| `record` | `CheckIn` |
| `register` | `Guest` |
| `processCheckIn`, `findQRCodeByToken` | `AccessControlService` |
| `scanQR`, `unlock`, `deny` | `AccessScanner` |
| `processRefund` | `PaymentGateway` |
| `sendRefundBreakdown`, `sendGuestNotification` | `NotificationService` |
