# Domain model and infrastructure: Reservation system

## Main entities

### `User`
Represents the user or resident who requests reservations and manages guests.
- Attributes: `id`, `name`, `email`, `phone`, `role`.

### `Space`
Facility or amenity available for booking (for example, terrace, court, hall, or BBQ area).
- Attributes: `id`, `name`, `type`, `capacity`, `hourly_rate`, `cancellation_policy`, `is_active`.

### `Reservation`
Links a user to a space for a specific time window.
- Attributes: `id`, `user_id`, `space_id`, `start_time`, `end_time`, `status` (PENDING, CONFIRMED, CHECKED_IN, CANCELLED), `total_amount`.

### `Payment`
Financial record of the initial reservation charge.
- Attributes: `id`, `reservation_id`, `amount`, `payment_method`, `status`, `created_at`.

### `Refund`
Refund issued after a cancellation subject to penalties.
- Attributes: `id`, `reservation_id`, `original_amount`, `penalty_amount`, `refund_amount`, `status`, `processed_at`.

### `QRCode`
Access pass with a dynamic token to validate physical entry to the space.
- Attributes: `id`, `reservation_id`, `qr_token`, `expiration_time`, `status` (VALID, USED, EXPIRED).

### `CheckIn`
Validation event recorded when scanning the pass at the access point.
- Attributes: `id`, `reservation_id`, `scanned_at`, `scanned_by`, `status` (SUCCESS, FAILED).

### `Guest`
Guest authorized by the reservation holder to enter the space.
- Attributes: `id`, `reservation_id`, `full_name`, `id_document`, `access_status`.

## Relationships

- `User` to `Reservation` (1:N): a user can create multiple reservations.
- `Space` to `Reservation` (1:N): a space supports multiple non-overlapping reservations.
- `Reservation` to `Payment` (1:1): each confirmed reservation has an associated initial payment transaction.
- `Reservation` to `Refund` (1:0..1): canceling a reservation under applicable policies generates a partial refund.
- `Reservation` to `QRCode` (1:1): every confirmed reservation issues a unique access QR code.
- `Reservation` to `CheckIn` (1:0..1): records the entry event when the QR code is scanned at the entrance.
- `Reservation` to `Guest` (1:N): the holder can attach guests up to the space's maximum capacity.

## Use cases

### UC1: Book an amenity space
- Actor: User.
- Preconditions: Authenticated user and available space in the selected time slot.
- Flow:
  1. The user checks space availability.
  2. Selects the schedule and confirms details.
  3. Pays the total reservation fee.
  4. The system confirms the reservation and generates the access `QRCode`.

### UC2: Cancel a reservation with partial refund
- Actor: User or administrator.
- Preconditions: Reservation in `CONFIRMED` status before its start time.
- Flow:
  1. The user requests reservation cancellation.
  2. The system checks the space's cancellation policy based on notice time, calculating the penalty fee (`penalty_amount`) and the refund amount (`refund_amount`).
  3. The reservation status changes to `CANCELLED`.
  4. The associated `QRCode` is invalidated.
  5. The reserved time slot is released for new bookings.
  6. The payment gateway processes the partial refund.
  7. The system notifies the user with the refund breakdown.

### UC3: QR code check-in
- Actor: User, access kiosk, or security staff.
- Preconditions: Reservation in `CONFIRMED` status and arrival within the permitted time window (for example, from 15 minutes before start until the end of the booking).
- Flow:
  1. The user presents their `QRCode` at the access scanner.
  2. The system verifies token authenticity, validity, and schedule.
  3. If valid, it logs the `CheckIn` timestamp, updates the reservation status to `CHECKED_IN`, and unlocks physical access.
  4. If invalid (expired, cancelled, or out of schedule), access is denied and a failure log is created.

### UC4: Register a guest
- Actor: Reservation holder.
- Preconditions: Reservation in `CONFIRMED` status and available capacity (`guests_count < space.capacity`).
- Flow:
  1. The user opens the details for their active reservation.
  2. Enters guest information (full name, ID document, email).
  3. The system ensures the maximum space capacity is not exceeded.
  4. A `Guest` record linked to the reservation is created.
  5. Optional: the system sends a secondary access pass or notification to the guest.
