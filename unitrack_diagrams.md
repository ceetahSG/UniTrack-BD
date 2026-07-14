# UniTrack BD — Required Diagrams (Mermaid)

---

## TECHNICAL PERSPECTIVE

### 1. Use Case Diagram

```mermaid
graph TB
    Student["👤 Student"]
    Helper["👤 Helper"]
    Admin["👤 Admin"]
    System["UniTrack System"]
    bKash["💳 bKash"]
    Mapbox["🗺️ Mapbox"]
    FCM["🔔 Firebase FCM"]

    Student -->|Register & Login| System
    Student -->|Buy Tickets| System
    Student -->|Top-up Wallet| bKash
    bKash -->|Payment Webhook| System
    Student -->|Track Live Bus| System
    System -->|Request Traffic Data| Mapbox
    Student -->|View ETA| System
    Student -->|Show QR Code| Helper
    Student -->|Raise Emergency Alert| System
    Student -->|View Ride History| System

    Helper -->|Register| System
    System -->|Admin Approval| Helper
    Helper -->|Start/End Trip| System
    Helper -->|Stream GPS Data| System
    Helper -->|Scan QR Code| System
    System -->|Validate QR Offline| Helper
    Helper -->|Report Seats| System
    Helper -->|Raise SOS Alert| System
    Helper -->|Sync Data| System

    Admin -->|Login| System
    Admin -->|CRUD Buses/Routes| System
    Admin -->|Manage Helpers| System
    Admin -->|Manage Students| System
    Admin -->|View Revenue Dashboard| System
    Admin -->|View Ridership Dashboard| System
    Admin -->|View Live Fleet Map| System
    Admin -->|Manage Alerts| System
    Admin -->|View Trip History| System
    Admin -->|Export Reports| System

    System -->|Send Push Notifications| FCM

    style Student fill:#dbeafe,stroke:#2563eb,color:#1e3a5f
    style Helper fill:#dbeafe,stroke:#2563eb,color:#1e3a5f
    style Admin fill:#dbeafe,stroke:#2563eb,color:#1e3a5f
    style System fill:#dcfce7,stroke:#16a34a,color:#14532d
    style bKash fill:#fee2e2,stroke:#dc2626,color:#7f1d1d
    style Mapbox fill:#fee2e2,stroke:#dc2626,color:#7f1d1d
    style FCM fill:#fee2e2,stroke:#dc2626,color:#7f1d1d
```

---

### 2. Sequence Diagram (Complete Boarding & Validation Flow)

```mermaid
sequenceDiagram
    participant SW as Student Web (PWA)
    participant API as FastAPI<br/>REST + WS
    participant Redis as Redis<br/>(Cache)
    participant PG as PostgreSQL<br/>(Auth of Record)
    participant Helper as Helper App<br/>(Flutter)
    participant Mapbox as Mapbox<br/>ETA Engine

    rect rgb(200, 220, 240)
    Note over SW,Helper: STEP 1: Student Buys Ticket & Top-ups Wallet
    SW->>API: POST /auth/login (varsity email)
    API->>PG: Verify email domain & fetch user
    PG-->>API: User record
    API-->>SW: JWT access token
    SW->>API: POST /wallet/topup (bKash amount)
    API->>PG: Create order (status=pending)
    API-->>SW: Payment initiation form
    SW->>Mapbox: Execute bKash Checkout
    Mapbox-->>SW: Payment status
    SW->>API: POST /webhooks/bkash (paymentID, trxID)
    API->>Redis: HMSET order:123 status:completed
    API->>PG: UPDATE orders SET status='completed'
    PG->>PG: CREATE ticket records (rides_remaining=20)
    API-->>SW: Wallet updated
    SW->>Redis: GET tickets:student_id (retrieve ticket list)
    Redis-->>SW: Ticket list + qr_secret (cached)
    end

    rect rgb(200, 240, 220)
    Note over SW,Helper: STEP 2: Student Joins Route & Views Live Tracking
    SW->>API: GET /routes/active
    API->>Redis: MGET bus:pos:* bus:eta:* bus:seats:*
    Redis-->>API: Live bus data
    API-->>SW: Route list with live buses
    SW->>API: WebSocket /ws/track/route?route_id=5
    API->>Redis: SUBSCRIBE route:5:ch
    Redis-->>API: Bus position updates (~5s)
    API-->>SW: Live map frame + ETA
    Mapbox->>API: Traffic-aware ETA (every 2-3 min)
    API->>Redis: HSET bus:eta:10 stops:[...]
    Redis-->>SW: Updated ETA
    end

    rect rgb(240, 220, 200)
    Note over SW,Helper: STEP 3: Helper Starts Trip & Student Boards
    Helper->>API: POST /trips/start (bus_id, route_id)
    API->>PG: INSERT into trips (status='active')
    API->>Redis: SET bus:status:10 active
    API->>Helper: foreground GPS service enabled
    Helper->>API: POST /gps/stream (lat, lon, timestamp)
    Helper-->>API: GPS batch sync (buffered, offline-capable)
    API->>Redis: LPUSH gps_ingest:stream {lat,lon,timestamp}
    SW->>SW: Generate rotating QR (offline, from qr_secret)
    SW-->>Helper: QR visible on PWA
    Helper->>Helper: Scan QR with camera
    Helper->>Helper: Validate QR offline (HMAC + nonce check + manifest lookup)
    Helper->>API: POST /tickets/validate (qr_payload)
    API->>PG: Verify ticket state + check nonce log
    API->>Redis: INCR tickets:student_id:redeemed
    API->>PG: INSERT into redemptions (ticket_id, trip_id, timestamp)
    API-->>Helper: ✓ Validated (offline nonce log updated)
    Helper->>API: POST /seats/report (occupancy='~30/45')
    API->>Redis: HSET bus:seats:10 count:30 capacity:45
    end

    rect rgb(240, 240, 200)
    Note over API,PG: STEP 4: Live Admin Monitoring
    API->>Redis: SUBSCRIBE fleet:ch alerts:ch
    API->>PG: Query live trip_summaries
    PG-->>API: Trip metadata
    API-->>Admin Dashboard: Live fleet map + KPIs
    API-->>Admin Dashboard: Revenue, ridership, alerts
    end

    rect rgb(240, 200, 220)
    Note over Helper,API: STEP 5: Trip Ends & Reconciliation
    Helper->>API: POST /trips/end (final_gps, final_seats)
    API->>PG: UPDATE trips SET status='completed'
    API->>PG: Aggregate redemptions for trip
    PG->>PG: Populate trip_summaries materialized view
    API->>Redis: DEL bus:status:10
    end
```

---

### 3. Entity Relationship Diagram (ERD)

```mermaid
erDiagram
    USERS ||--o{ STUDENTS : has
    USERS ||--o{ HELPERS : has
    STUDENTS ||--o{ ORDERS : places
    STUDENTS ||--o{ TICKETS : owns
    STUDENTS ||--o{ REDEMPTIONS : creates
    STUDENTS ||--o{ ALERTS : raises
    HELPERS ||--o{ TRIPS : conducts
    HELPERS ||--o{ GPS_POINTS : logs
    HELPERS ||--o{ SEAT_REPORTS : submits
    HELPERS ||--o{ ALERTS : creates
    BUSES ||--o{ TRIPS : assigned
    BUSES ||--o{ ROUTES : "on"
    ROUTES ||--o{ STOPS : has
    ROUTES ||--o{ SCHEDULES : has
    TRIPS ||--o{ REDEMPTIONS : contains
    TRIPS ||--o{ GPS_POINTS : generates
    TRIPS ||--o{ SEAT_REPORTS : "has"
    TRIPS ||--o{ ALERTS : "raises"
    TRIPS ||--o{ TRIP_SUMMARIES : "summarizes"
    TICKET_PRODUCTS ||--o{ ORDERS : "sold_as"
    TICKETS ||--o{ REDEMPTIONS : "redeemed_via"
    ORDERS ||--o{ TRANSACTIONS : "paid_via"
    ALERTS ||--o{ AUDIT_LOGS : "logged_in"

    USERS {
        string user_id PK
        string email UK
        string password_hash
        string role
        datetime created_at
        datetime updated_at
    }

    STUDENTS {
        string student_id PK, FK
        string student_id_no UK
        string phone
        string photo_url
        string university_email
        string status
        float wallet_balance
        datetime verified_at
    }

    HELPERS {
        string helper_id PK, FK
        string phone
        string bus_assigned FK
        string status
        datetime approved_at
        datetime last_sync_at
    }

    BUSES {
        string bus_id PK
        string registration_no UK
        string helper_id FK
        int capacity
        string status
        float latitude
        float longitude
        datetime last_gps_at
    }

    ROUTES {
        string route_id PK
        string name
        string polyline
        int distance_km
        datetime created_at
    }

    STOPS {
        string stop_id PK
        string route_id FK
        int sequence
        string name
        float latitude
        float longitude
        int dwell_time_sec
    }

    SCHEDULES {
        string schedule_id PK
        string route_id FK
        string day_of_week
        string departure_time
        string expected_arrival_time
    }

    TRIPS {
        string trip_id PK
        string bus_id FK
        string helper_id FK
        string route_id FK
        datetime start_time
        datetime end_time
        string status
        int total_redemptions
        float delay_minutes
    }

    TICKET_PRODUCTS {
        string product_id PK
        string name
        int rides_included
        float price_taka
        string status
        datetime created_at
    }

    ORDERS {
        string order_id PK
        string student_id FK
        string product_id FK
        float amount_taka
        string status
        string payment_id UK
        string trx_id
        datetime created_at
        datetime completed_at
    }

    TICKETS {
        string ticket_id PK
        string student_id FK
        string order_id FK
        string product_id FK
        int rides_remaining
        int rides_used
        string status
        string qr_secret
        datetime created_at
        datetime expires_at
    }

    REDEMPTIONS {
        string redemption_id PK
        string ticket_id FK
        string trip_id FK
        string helper_id FK
        string student_id FK
        datetime redeemed_at
        string validation_method
        string nonce
    }

    GPS_POINTS {
        string gps_id PK
        string trip_id FK
        string helper_id FK
        float latitude
        float longitude
        float accuracy_m
        string map_matched_segment
        datetime recorded_at
        datetime synced_at
    }

    SEAT_REPORTS {
        string seat_report_id PK
        string trip_id FK
        string helper_id FK
        int seated_count
        int standing_count
        string occupancy_status
        datetime reported_at
    }

    ALERTS {
        string alert_id PK
        string trip_id FK
        string creator_type
        string creator_id FK
        string alert_type
        string severity
        string status
        string description
        datetime raised_at
        datetime resolved_at
        string resolved_note
    }

    TRANSACTIONS {
        string transaction_id PK
        string order_id FK
        string bkash_payment_id UK
        string bkash_trx_id UK
        float amount_taka
        string status
        datetime created_at
    }

    TRIP_SUMMARIES {
        string trip_summary_id PK
        string trip_id FK
        string route_id FK
        string bus_id FK
        string helper_id FK
        datetime date
        int total_validations
        float revenue_taka
        float delay_minutes
        int peak_occupancy
    }

    AUDIT_LOGS {
        string audit_id PK
        string admin_id FK
        string resource_type
        string resource_id
        string action
        string old_value
        string new_value
        datetime logged_at
    }
```

---

## PRODUCT PERSPECTIVE

### 4. User Journey Map (Student — From Discovery to Arrival)

```mermaid
journey
    title Student Journey: Download App → Buy Ticket → Board Bus
    section Awareness
      Discovers UniTrack: 3: Student
      Reads about features: 3: Student
    section Onboarding
      Sign up (varsity email): 5: Student
      Email verification: 5: Student
      Create profile: 4: Student
      Download PWA: 5: Student
    section Financial
      Navigate to wallet: 3: Student
      Top-up via bKash: 4: Student
      Confirm payment: 5: Student
      Wallet balance updates: 5: Student
    section Booking & Discovery
      Browse available routes: 4: Student
      Check live bus position: 5: Student
      View real-time ETA: 5: Student
      Check seat availability: 4: Student
      Select bus & confirm: 4: Student
    section Active Trip
      Receive QR code (offline-ready): 5: Student
      Wait for bus at stop: 3: Student
      Scan QR at boarding: 5: Student
      Instant fare deduction: 4: Student
      View journey on live map: 5: Student
      Track ETA to destination: 5: Student
    section Completion
      Arrive at destination: 5: Student
      View ride in history: 4: Student
      Rate trip (future): 3: Student
      Receive receipt/summary: 4: Student
```

---

### 5. User Flow (Core UI Paths)

```mermaid
flowchart TD
    A["🚀 App Launch"] --> B{"Logged In?"}
    B -->|No| C["📝 Login / Sign-up Screen"]
    C --> C1{"User Type?"}
    C1 -->|Student| C2["Email Verification<br/>varsity domain check"]
    C1 -->|Helper| C3["Awaiting Admin<br/>Approval"]
    C2 --> D["✅ Dashboard"]
    C3 --> C3A["Helper Status:<br/>Pending"]

    B -->|Yes| D

    D --> D1{"Role?"}

    D1 -->|Student| S["📱 STUDENT FLOW"]
    D1 -->|Helper| H["🚗 HELPER FLOW"]
    D1 -->|Admin| AD["⚙️ ADMIN FLOW"]

    subgraph S["STUDENT FLOW"]
        S1["🏠 Home Dashboard<br/>- Active routes<br/>- Wallet balance"]
        S1 --> S2{"Next Action?"}
        S2 -->|View Route| S3["🗺️ Live Route Screen<br/>- Bus position<br/>- ETA per stop<br/>- Seat availability"]
        S2 -->|Top-up Wallet| S4["💳 Wallet Top-up<br/>- Enter amount<br/>- bKash redirect<br/>- Confirm payment"]
        S2 -->|History| S5["📋 Ride History<br/>- Past trips<br/>- Receipts<br/>- Fare details"]
        S3 --> S6["🎫 Ticket & QR<br/>- Display QR code<br/>- Works offline<br/>- Auto-rotate"]
        S4 --> S7["✅ Payment Success<br/>- Wallet updated<br/>- Tickets added"]
        S6 --> S8["🚍 Board Bus<br/>- Helper scans QR<br/>- Fare deducted<br/>- Ride active"]
        S8 --> S9["🚗 In Transit<br/>- Live GPS tracking<br/>- ETA countdown<br/>- SOS button"]
        S9 --> S10["🏁 Arrived<br/>- Trip complete<br/>- Receipt generated"]
        S10 --> S1
        S2 -->|Emergency| S11["🆘 Report Emergency<br/>- Type incident<br/>- Send to admins<br/>- Rate-limited"]
    end

    subgraph H["HELPER FLOW"]
        H1["🏠 Helper Dashboard<br/>- Today's trips<br/>- Bus assignment"]
        H1 --> H2{"Start Trip?"}
        H2 -->|Yes| H3["🚗 Trip Start<br/>- Select route<br/>- GPS enabled<br/>- Foreground service"]
        H3 --> H4["📡 GPS Streaming<br/>- Continuous location<br/>- Buffered offline<br/>- ~5s updates"]
        H4 --> H5["🎫 QR Validation<br/>- Scan QR code<br/>- Offline HMAC check<br/>- Nonce log"]
        H5 --> H6{"Valid?"}
        H6 -->|Yes| H7["✅ Fare Deducted<br/>- Sync later if offline<br/>- Nonce recorded"]
        H6 -->|No| H8["❌ Invalid Ticket<br/>- Student ID fallback<br/>- Manual redemption"]
        H4 --> H9["👥 Report Seats<br/>- Tap occupancy<br/>- Seated / Standing / Full"]
        H9 --> H10["💾 Seat Report Sent"]
        H7 --> H11["🆘 Emergency Button<br/>- One-tap SOS<br/>- Trip attached<br/>- Priority queue"]
        H11 --> H12["📤 Offline Alert Sync<br/>- Queued if offline"]
        H4 --> H13{"End Trip?"}
        H13 -->|Yes| H14["🏁 Trip End<br/>- Finalize GPS<br/>- Final seat report<br/>- Close trip"]
        H14 --> H1
    end

    subgraph AD["ADMIN FLOW"]
        AD1["📊 Admin Dashboard<br/>- KPIs<br/>- Revenue today<br/>- Active buses<br/>- Open alerts"]
        AD1 --> AD2{"Module?"}
        AD2 -->|Live Fleet| AD3["🗺️ Live Fleet Map<br/>- All buses + heading<br/>- Delay vs schedule<br/>- Occupancy estimate<br/>- GPS freshness"]
        AD2 -->|Revenue| AD4["💰 Revenue Dashboard<br/>- By day/week/month<br/>- By route/product<br/>- Package uptake"]
        AD2 -->|Ridership| AD5["📈 Ridership Analytics<br/>- Occupancy trends<br/>- Peak-hour curves<br/>- Sold vs validated gap"]
        AD2 -->|Management| AD6["⚙️ Management CRUD<br/>- Buses<br/>- Routes & stops<br/>- Schedules<br/>- Ticket products<br/>- Helpers<br/>- Students"]
        AD2 -->|Alerts| AD7["🚨 Emergency Console<br/>- Open alerts<br/>- Acknowledge<br/>- Resolve with note<br/>- Critical → email"]
        AD2 -->|Transactions| AD8["💳 Wallet & Transactions<br/>- Order browser<br/>- Refund initiation<br/>- Reconciliation queue"]
        AD2 -->|Trip History| AD9["🚌 Trip History<br/>- Searchable trips<br/>- GPS playback<br/>- Redemption list<br/>- Seat timeline"]
        AD3 --> AD10["📍 Click Bus<br/>- Trip drawer<br/>- Helper info<br/>- Route progress<br/>- Recent redemptions"]
        AD6 --> AD11["✏️ Edit Resource<br/>- Form submission<br/>- Audit log entry"]
        AD7 --> AD12["✅ Resolve Alert<br/>- Add resolution note<br/>- Broadcast to all"]
    end

    style S fill:#e0f2ff,stroke:#0284c7,color:#003d82
    style H fill:#e0ffe0,stroke:#16a34a,color:#143d2d
    style AD fill:#fff0e0,stroke:#ea580c,color:#5c2405
```

---

## USAGE

All diagrams are **Mermaid v11.12+** compatible. Use them in:
- **GitHub markdown** (auto-renders)
- **HTML/docs** (include mermaid.min.js + `<div class="mermaid">...</div>`)
- **Confluence, Notion, GitLab** (native Mermaid support)
- **Your team wiki/design doc**

Each diagram is self-contained and can be extracted independently.
