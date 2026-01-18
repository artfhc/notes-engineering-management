# Parking Lot System Design

## Problem Statement

Design a parking lot management system with mobile app support that allows users to:
1. Search and view available parking spots
2. Reserve spots with start/end time
3. Make payments (guest or authenticated users)
4. Receive real-time availability updates
5. Check-in and check-out using QR codes

---

## Requirements

### Functional Requirements

- User can enter and exit a parking lot
- System assigns parking spots based on vehicle type
- System tracks availability in real-time
- Payment system with guest checkout support
- Support different vehicle types: car, SUV, truck, bike, bus
- Reservation with time windows (hourly/daily)
- Reservation expires if not confirmed within 1 minute
- Admin can add/remove parking floors/spots
- Multi-level, multi-location support

### Non-Functional Requirements

- Low-latency spot allocation
- High availability
- Scalable to multiple parking lots
- Real-time updates on mobile clients
- Offline support for recent searches
- Privacy-compliant payment processing

---

## High-Level Architecture

### Client (Mobile App)

- Search lots by location (map or list view)
- Filter by time range, vehicle type
- View lot & spot details with real-time availability
- Reservation & checkout (guest or signed-in)
- Push notifications (confirmation, expiry, cancellation)
- Offline cache of recent searches and reservations
- QR code for entry/exit

### Backend Services

| Service | Responsibility |
|---------|---------------|
| **Search Service** | List lots by location + metadata (geohash or PostGIS) |
| **Availability Service** | Tracks current + future reservations, syncs real-time spot status |
| **Reservation Service** | Handles start/end time, temp hold, final booking, 1-min expiry |
| **Payment Service** | Checkout, receipts, guest payment flows |
| **User Service** | Authentication (OAuth, guest token) |
| **Pricing Engine** | Computes hourly/daily charges |
| **Notification Service** | Email/SMS/push reminders |

### Database Schema

| Table | Key Fields |
|-------|-----------|
| **ParkingLot** | id, name, location, totalLevels |
| **ParkingSpot** | id, lotId, level, type, status |
| **Reservation** | id, spotId, startTime, endTime, vehicleType, userId/guestToken, status |
| **User** | id, name, email, phone, optional password |
| **Vehicle** | id, userId, plate, type |
| **Payment** | id, reservationId, amount, method, status |

**Caching:** Redis for spot availability and temporary reservation holds.

---

## Class Design (OOP Layer)

```kotlin
enum class VehicleType { CAR, SUV, TRUCK, BIKE, BUS }

open class Vehicle(val plateNumber: String, val type: VehicleType)

data class ParkingSpot(
    val id: String,
    val type: VehicleType,
    var isOccupied: Boolean = false,
    val level: Int
)

data class Ticket(
    val ticketId: String,
    val spot: ParkingSpot,
    val vehicle: Vehicle,
    val entryTime: Long,
    var exitTime: Long? = null
)

class ParkingLot(
    val id: String,
    val floors: List<ParkingFloor>
) {
    fun getAvailableSpot(vehicleType: VehicleType): ParkingSpot?
    fun assignSpot(vehicle: Vehicle): Ticket
    fun releaseSpot(ticketId: String): Boolean
}
```

### Mobile Data Models

```kotlin
data class ParkingLot(
    val id: String,
    val name: String,
    val location: LatLng,
    val levels: Int,
    val availableSpots: List<SpotSummary>
)

data class SpotSummary(
    val vehicleType: VehicleType,
    val count: Int,
    val hourlyRate: Double
)

data class ReservationRequest(
    val lotId: String,
    val spotType: VehicleType,
    val startTime: Long,
    val endTime: Long,
    val userId: String?,      // Optional for guest
    val guestEmail: String?   // Optional for signed-in
)

data class PaymentInfo(
    val reservationId: String,
    val method: String, // e.g. "google_pay"
    val token: String
)
```

---

## API Design

### 1. Search Parking Lots

**GET** `/api/v1/lots/search?lat=...&lng=...&radius=10km`

**Response:**

```json
{
  "lots": [
    {
      "id": "lot123",
      "name": "Downtown Garage",
      "location": {"lat": 37.7749, "lng": -122.4194},
      "levels": 3,
      "distance": 0.5
    }
  ]
}
```

---

### 2. Check Spot Availability

**GET** `/api/v1/lots/{lotId}/availability?startTime=...&endTime=...&vehicleType=SUV`

**Response:**

```json
{
  "lotId": "abc123",
  "availableSpots": [
    {"vehicleType": "CAR", "count": 12, "ratePerHour": 2.5},
    {"vehicleType": "SUV", "count": 3, "ratePerHour": 4.0}
  ]
}
```

---

### 3. Hold a Spot (Temporary Reservation)

**POST** `/api/v1/reservations/hold`

**Request:**

```json
{
  "lotId": "abc123",
  "vehicleType": "SUV",
  "startTime": 1718848200000,
  "endTime": 1718855400000,
  "userId": null,
  "guestEmail": "guest@example.com"
}
```

**Response:**

```json
{
  "reservationId": "rsv123",
  "status": "HELD",
  "expiresAt": 1718848260000
}
```

**Mobile Use:** Show countdown timer (60 seconds) for user to complete payment.

---

### 4. Complete Reservation (Checkout)

**POST** `/api/v1/payments/complete`

**Request:**

```json
{
  "reservationId": "rsv123",
  "method": "google_pay",
  "paymentToken": "tok_xxx"
}
```

**Response:**

```json
{
  "status": "CONFIRMED",
  "qrCode": "data:image/png;base64,...",
  "reservationId": "rsv123"
}
```

---

### 5. Cancel Reservation

**POST** `/api/v1/reservations/cancel`

**Request:**

```json
{
  "reservationId": "rsv123"
}
```

---

### 6. Get User Reservations

**GET** `/api/v1/users/{userId}/reservations`

---

## Mobile Architecture

Follow clean MVVM (or MVI) pattern:

```
UI (Jetpack Compose / SwiftUI)
  ↓
ViewModel (UI logic, state mgmt)
  ↓
UseCase Layer (business rules)
  ↓
Repository (abstracts network/local)
  ↓
Data Layer:
  |-- RemoteDataSource (Retrofit, Ktor, Apollo)
  |-- LocalDataSource (Room, CoreData)
  |-- CacheDataSource (in-memory, LRU cache)
```

### Key Screens

| Screen | Features |
|--------|----------|
| **Home / Search** | Location-based search (map or list), filters (vehicle type, time), saved locations |
| **Lot Detail** | Lot metadata, spot availability by level and type, price calculation |
| **Reservation Form** | Select vehicle type, start/end time, guest info |
| **Checkout** | Payment summary, payment method (Apple Pay / Google Pay / card) |
| **Confirmation / Ticket** | QR code or ID, cancellation option, calendar add |
| **My Reservations** | Active, past, and upcoming reservations |
| **Guest Flow** | Skip login, temporary token for actions |

---

## Real-Time Updates

### Recommended: WebSocket

| Criteria | Why WebSocket Wins |
|----------|-------------------|
| **Real-time needs** | Spot availability must reflect in near real-time as users compete to reserve |
| **Bi-directional** | Enables future features like live support, cancel confirmations, QR gate interactions |
| **Low-latency** | Instant updates without polling wait-time |
| **Efficient** | Only sends changes; no repeated request overhead |
| **Widely supported** | Standard in both Android and iOS; works well with backend load balancers |

**Usage Pattern:**
- Open WebSocket when user views a parking lot detail screen
- Receive spot availability updates
- Be notified if user's held spot is released due to timeout
- Close socket when user navigates away or app goes to background

**WebSocket Payload Example:**

```json
{
  "lotId": "abc123",
  "availableSpots": {
    "CAR": 9,
    "SUV": 2
  }
}
```

### Fallback Strategy

```kotlin
if (webSocketConnectionFails) {
    startPollingAvailability(lotId, interval = 5s)
}
```

### Alternatives

| Option | When to Use |
|--------|-------------|
| **Firebase Realtime DB** | Startup/MVP phase, fast time-to-market |
| **HTTP Polling** | Very basic or low-traffic app, extreme simplicity |
| **WebSocket + Polling Fallback** | ✅ Production-level app with dynamic spot competition |

---

## Data Sync & Offline Strategy

| Feature | Approach |
|---------|----------|
| **Recent lots** | Cache last 3–5 search results using Room or SQLite |
| **Spot availability** | Show stale data with a "refreshing..." indicator |
| **In-progress reservation** | Save locally if user switches apps |
| **Retry logic** | Use WorkManager or background queues for failed payments or confirmations |

---

## Performance and Scaling

### Optimization Strategies

| Concern | Solution |
|---------|----------|
| **Performance** | Redis cache for availability; bulk queries; pagination |
| **Read-heavy traffic** | Redis caching, CDN for static assets |
| **Write-heavy traffic** | Async processing with Kafka for ticketing and billing |
| **Concurrency** | Distributed locks on ParkingSpot ID during booking |
| **Sharding** | Split by parking lot or region |
| **Reservation expiry** | TTL or background worker to clean stale HOLD reservations |

---

## Edge Cases and Handling

### Time and Timezone

| Case | Mobile Handling |
|------|----------------|
| **Device time skew** | Use `SystemClock.elapsedRealtime()` (Android) or `ProcessInfo.systemUptime` (iOS) for countdowns. Always validate against server UTC time. |
| **Cross-timezone bookings** | Show both parking lot timezone and user's local time. Store all times in UTC. |
| **DST transitions** | Handle timezones via `java.time` or `JodaTime`. Avoid device time parsing. |
| **Zero-minute duration** | Enforce minimum duration (e.g., 15 minutes) on frontend and backend. |

### Reservation and Availability

| Case | Handling |
|------|----------|
| **Race condition in booking** | Use database-level transactions or distributed locks |
| **Expired holds** | Use Redis TTL or backend job to expire reservations after 60 seconds |
| **Spot unavailable during checkout** | Reconfirm availability before payment. Cancel or reassign spot as fallback |
| **Early or late check-in** | Define grace period (e.g., ±10 mins) with server-side checks |
| **Reservation expires after 1 min** | Show countdown timer on checkout screen; auto-cancel locally |

### Payment

| Case | Handling |
|------|----------|
| **Network loss post-payment** | Use idempotent payment endpoints tied to reservation ID. Persist status server-side |
| **Payment failure with held spot** | Tie hold to payment success; cancel hold on failure |
| **Unsupported payment methods** | Filter and display supported methods dynamically based on region and platform |
| **Refunds and pro-rata** | Define clear refund policies and enforce server-side checks for used time |

### User Session

| Case | Handling |
|------|----------|
| **Guest clears app data** | Store guest reservations server-side keyed by email or temporary token |
| **Expired token during socket** | Implement auto-refresh mechanism and reconnect flow |
| **Multi-device login** | Use consistent session syncing or display session logs in profile |
| **No internet at checkout** | Warn user, disable "Confirm" button until reconnected |
| **App backgrounded mid-checkout** | Resume with saved state from local DB or ViewModel Store |

### Location and Localization

| Case | Handling |
|------|----------|
| **No GPS / Geolocation** | Allow manual address entry fallback. Cache last known location |
| **Country border regions** | Normalize backend data and display localized policies and fees |
| **Language support** | Use i18n libraries for all static text. Support key locales with graceful fallback |
| **Currency display** | API returns both lot currency and user currency with conversion rate |
| **RTL layout** | Use `android:supportsRtl` and layout mirroring. Test with pseudo-RTL builds |

### Platform-Specific

| Case | Handling |
|------|----------|
| **Backgrounded WebSocket disconnect** | Reconnect on foreground and refresh state |
| **App killed mid-flow** | Persist in-progress state locally using Room/CoreData |
| **Device locale vs language** | Let user choose preferred language in-app |

### Accessibility

| Case | Handling |
|------|----------|
| **Screen readers** | Add semantic labels, `contentDescription`, and logical tab order |
| **Colorblind-friendly UI** | Use color + icon pairing; test with colorblind simulators |
| **Small tap targets** | Follow platform guidelines: 48dp minimum for tap areas |

---

## Technologies & SDKs

| Layer | Android | iOS |
|-------|---------|-----|
| **Networking** | Retrofit / Ktor | Alamofire |
| **Real-time** | Firebase, Socket.IO | Firebase, Starscream |
| **Local DB** | Room / SQLite | CoreData / Realm |
| **Payments** | Stripe / Google Pay | Stripe / Apple Pay |
| **Maps** | Google Maps SDK | Apple Maps SDK |
| **Analytics** | Firebase / Mixpanel | Firebase / Mixpanel |
| **State Management** | Jetpack Compose + ViewModel / MVI | SwiftUI + Combine / Redux-style |

---

## Localization Code Examples

### Language Switching

**Android:**

```kotlin
fun setLocale(context: Context, languageCode: String): Context {
    val locale = Locale(languageCode)
    Locale.setDefault(locale)
    val config = context.resources.configuration
    config.setLocale(locale)
    return context.createConfigurationContext(config)
}
```

**iOS:**

```swift
UserDefaults.standard.set(["fr"], forKey: "AppleLanguages")
UserDefaults.standard.synchronize()
```

### Currency Formatting

**Kotlin:**

```kotlin
fun formatCurrency(amount: Double, locale: Locale): String {
    val formatter = NumberFormat.getCurrencyInstance(locale)
    return formatter.format(amount)
}
```

**Swift:**

```swift
func formatCurrency(amount: Double, locale: Locale) -> String {
    let formatter = NumberFormatter()
    formatter.numberStyle = .currency
    formatter.locale = locale
    return formatter.string(from: NSNumber(value: amount)) ?? ""
}
```

---

## Future Extensions

- Dynamic pricing (demand-based)
- Loyalty program
- Admin dashboard for parking lot operators
- Entry validation using camera or NFC
- License Plate Recognition (LPR) for automatic entry/exit
- EV charging station integration
- Corporate/fleet accounts
- Multi-vehicle reservations
