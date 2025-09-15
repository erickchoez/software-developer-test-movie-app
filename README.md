# Cinema Booking System — Design Documentation

This document contains the deliverables for the **Software Developer Design Test**. The design is presented in English, using **Mermaid** diagrams.

---

## 1. Entity-Relationship Diagram (ERD)

```mermaid
erDiagram
    USER ||--o{ BOOKING : places
    USER {
        uuid id PK
        string email
        string password_hash
        string full_name
        string phone
        datetime created_at
        datetime updated_at
    }

    MOVIE ||--o{ SCREENING : "has"
    MOVIE {
        uuid id PK
        string title
        text description
        int duration_minutes
        date release_date
        string language
        string genre
        datetime created_at
        datetime updated_at
    }

    ROOM ||--o{ SEAT : contains
    ROOM ||--o{ SCREENING : hosts
    ROOM {
        uuid id PK
        string name
        int capacity
        string layout_type
        text notes
    }

    SEAT {
        uuid id PK
        uuid room_id FK
        string seat_label
        int row
        int number
        string type
        boolean accessible
    }

    SCREENING ||--o{ BOOKING : "is booked in"
    SCREENING {
        uuid id PK
        uuid movie_id FK
        uuid room_id FK
        datetime show_start
        datetime show_end
        int base_price_cents
        enum status
    }

    BOOKING ||--o{ BOOKING_SEAT : "contains"
    BOOKING ||--o{ PAYMENT : "has"
    BOOKING {
        uuid id PK
        uuid user_id FK
        uuid screening_id FK
        enum status
        int total_amount_cents
        datetime created_at
        datetime expires_at
    }

    BOOKING_SEAT {
        uuid id PK
        uuid booking_id FK
        uuid seat_id FK
        int price_cents
        enum status
    }

    PAYMENT {
        uuid id PK
        uuid booking_id FK
        enum method
        enum status
        string provider_txn_id
        int amount_cents
        datetime created_at
        datetime updated_at
    }
```

---

## 2. Class Design

```mermaid
classDiagram
    class User {
        +UUID id
        +String email
        +String passwordHash
        +String fullName
        +String phone
        +register()
        +login()
        +updateProfile()
    }

    class Movie {
        +UUID id
        +String title
        +String description
        +int durationMinutes
        +Date releaseDate
        +create()
        +update()
        +delete()
    }

    class Room {
        +UUID id
        +String name
        +int capacity
        +Seat[] seats
        +create()
        +update()
        +delete()
        +getSeatLayout()
    }

    class Seat {
        +UUID id
        +String seatLabel
        +int row
        +int number
        +SeatType type
        +boolean accessible
    }

    class Screening {
        +UUID id
        +Movie movie
        +Room room
        +DateTime showStart
        +DateTime showEnd
        +schedule()
        +reschedule()
        +cancel()
        +getAvailableSeats()
    }

    class Booking {
        +UUID id
        +User user
        +Screening screening
        +BookingSeat[] seats
        +BookingStatus status
        +holdSeats()
        +confirm(paymentInfo)
        +cancel(reason)
        +releaseSeats()
    }

    class BookingSeat {
        +UUID id
        +Seat seat
        +int priceCents
        +SeatHoldStatus status
    }

    class Payment {
        +UUID id
        +Booking booking
        +PaymentMethod method
        +PaymentStatus status
        +process()
        +refund()
    }

    class AuthService {
        +register(request)
        +login(request)
        +verifyToken(token)
        +logout()
    }

    class ScreeningService {
        +search(title?, date?)
        +getDetails(screeningId)
        +getAvailableSeats(screeningId)
    }

    class BookingService {
        +createBooking(userId, screeningId, seatIds)
        +confirmBooking(bookingId, paymentInfo)
        +cancelBooking(bookingId, userId)
        +autoExpireHolds()
    }

    class PaymentGateway {
        +createPayment(amount, method, metadata)
        +capture(paymentId)
        +refund(paymentId, amount)
    }

    class NotificationService {
        +sendBookingConfirmation(user, booking)
        +sendCancellationNotice(user, booking)
    }

    Booking --> Screening
    Booking --> User
    Booking --> BookingSeat
    BookingSeat --> Seat
    Screening --> Movie
    Screening --> Room
```

---

## 3. Sequence Diagram — Reservation

```mermaid
sequenceDiagram
    participant Customer as Customer
    participant Auth as AuthService
    participant ScreenSvc as ScreeningService
    participant BookingSvc as BookingService
    participant PaymentGW as PaymentGateway
    participant DB as Database
    participant Notify as NotificationService

    Customer->>ScreenSvc: search(title, date)
    ScreenSvc->>DB: query screenings
    DB-->>ScreenSvc: screenings list
    ScreenSvc-->>Customer: present screenings

    Customer->>BookingSvc: createBooking(screeningId, seatIds)
    BookingSvc->>DB: hold seats (PENDING)
    DB-->>BookingSvc: bookingId
    BookingSvc-->>Customer: return bookingId

    Customer->>PaymentGW: pay(paymentDetails)
    PaymentGW-->>BookingSvc: payment success
    BookingSvc->>DB: confirm booking
    BookingSvc->>Notify: send confirmation
    Notify-->>Customer: booking confirmation
```

---

## 4. Sequence Diagram — Cancel a Reservation

```mermaid
sequenceDiagram
    participant Customer as Customer
    participant Auth as AuthService
    participant BookingSvc as BookingService
    participant DB as Database
    participant PaymentGW as PaymentGateway
    participant Notify as NotificationService

    Customer->>Auth: authenticate()
    Customer->>BookingSvc: requestCancel(bookingId)
    BookingSvc->>DB: fetch booking
    DB-->>BookingSvc: booking details

    alt refundable
        BookingSvc->>PaymentGW: refund(paymentId)
        PaymentGW-->>BookingSvc: refund success
        BookingSvc->>DB: set CANCELLED + RELEASED
        BookingSvc->>Notify: send cancellation notice
        Notify-->>Customer: cancellation confirmation
    else non-refundable
        BookingSvc->>DB: set CANCELLED
        BookingSvc->>Notify: send notice
        Notify-->>Customer: cancellation without refund
    end
```

---

## 5. State Diagrams

### 5.1 Booking

```mermaid
stateDiagram-v2
    [*] --> PENDING
    PENDING --> CONFIRMED : payment success
    PENDING --> CANCELLED : payment failed / user cancel
    PENDING --> EXPIRED : hold expired

    CONFIRMED --> CANCELLED : cancellation
    CONFIRMED --> REFUNDED : refund issued
    CONFIRMED --> COMPLETED : show ended

    CANCELLED --> [*]
    REFUNDED --> [*]
    EXPIRED --> [*]
    COMPLETED --> [*]
```

### 5.2 Seat Hold

```mermaid
stateDiagram-v2
    [*] --> AVAILABLE
    AVAILABLE --> HELD : holdSeats()
    HELD --> CONFIRMED : booking confirmed
    HELD --> RELEASED : cancel or expire
    RELEASED --> AVAILABLE
```

### 5.3 Screening

```mermaid
stateDiagram-v2
    [*] --> SCHEDULED
    SCHEDULED --> IN_PROGRESS : show_start
    IN_PROGRESS --> COMPLETED : show_end
    SCHEDULED --> CANCELLED : admin cancel
    IN_PROGRESS --> CANCELLED : emergency cancel
    COMPLETED --> [*]
    CANCELLED --> [*]
```

---

## 6. Technical Considerations

* **Concurrency Control:** Seats locked via DB transactions and UNIQUE constraints `(screening_id, seat_id)`.
* **Performance:** Cache read-heavy data (screenings, seat maps). Scale horizontally at peak demand.
* **Resilience:** Automatic seat release after TTL expiration for unconfirmed bookings.
* **Security:** JWT-based auth, password hashing, PCI-compliant payments.
* **Monitoring:** Metrics for bookings, expirations, payment failures.

---
