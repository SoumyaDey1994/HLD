# Uber/Swiggy Nearby Driver Matching System Design
### Date: 27th May, 2026

# 1. Driver Goes Online & Shares GPS Location

## Workflow
1. Driver app continuously sends GPS coordinates.
2. API Gateway forwards updates to Location Service.
3. Location Service stores latest driver location in Redis.
4. Same event is published to Kafka for downstream consumers.

---

## GPS Storage in Redis (Geohash-based)

Redis internally stores GEO data using Geohash + Sorted Sets.

### Sample Redis Command

```bash
GEOADD drivers_geo 77.5946 12.9716 D123
```

- `77.5946` → Longitude
- `12.9716` → Latitude
- `D123` → Driver ID

---

## Flow Diagram

```text
Driver App
   ↓
API Gateway
   ↓
Location Service
   ↓
Redis GEO Store
   ↓
Kafka Event Bus
```

---

# 2. User Creates Ride / Delivery Request

## Workflow
1. User submits pickup location.
2. Matching Service receives request.
3. Nearby drivers are fetched from Redis.

---

## Nearby Driver Search in Redis

### Sample Redis Command

```bash
GEOSEARCH drivers_geo
FROMLONLAT 77.6387 12.9611
BYRADIUS 3 km
WITHDIST
COUNT 100
```

Returns:
- Nearby driver IDs
- Approximate distance

---

## Flow Diagram

```text
User App
   ↓
API Gateway
   ↓
Matching Service
   ↓
Redis GEOSEARCH
   ↓
Nearby Drivers
```

---

# 3. Distance Calculation & ETA

## Workflow
1. Redis provides approximate geo distance.
2. ETA Service calculates precise travel distance/time.
3. Routing APIs may be used for road-based ETA.
4. Haversine Formula Used for accurate earth-distance calculation.
---

## Flow Diagram

```text
Redis Distance
   ↓
ETA Service
   ↓
Road Distance Calculation
   ↓
Final ETA
```

---

# 4. Driver Filtering

## Top 5 Filtering Criteria

| Criteria | Purpose |
|---|---|
| Online Status | Driver must be active |
| Availability | Driver should not be busy |
| Vehicle Type | Match Bike/Car/Auto |
| Distance Threshold | Remove far-away drivers |
| Driver Rating | Maintain service quality |

Filtering data may come from:
- Redis Cache
- Driver Service
- Profile DB

---

## Flow Diagram

```text
Nearby Drivers
   ↓
Availability Check
+ Vehicle Match
+ Distance Filter
+ Rating Filter
   ↓
Eligible Drivers
```

---

# 5. Driver Ranking & Notification

## Workflow
1. Eligible drivers are ranked.
2. Closest/highest-score drivers selected.
3. Notifications sent to top drivers.

---

## Ranking Factors

- Distance
- ETA
- Acceptance Rate
- Idle Time
- Driver Score

---

## Notification Channels

- Push Notification (FCM/APNS)
- WebSocket
- MQTT (optional)

---

## Flow Diagram

```text
Eligible Drivers
   ↓
Ranking Engine
   ↓
Top Drivers Selected
   ↓
Notification Service
   ↓
Driver Apps
```

---

# 6. Driver Accepts Request

## Workflow
1. Driver clicks ACCEPT.
2. Assignment Service receives acceptance.
3. Distributed lock prevents duplicate assignment.

---

## Redis Distributed Lock

### Sample Redis Command

```bash
SET trip:T900:lock D101 NX EX 10
```

Meaning:
- `NX` → Create only if not exists
- `EX 10` → Expire in 10 sec

Only first successful driver gets the lock.

---

## Flow Diagram

```text
Driver Accepts
   ↓
Assignment Service
   ↓
Redis Distributed Lock
   ↓
Winner Driver Selected
```

---

# 7. Final Assignment & Persistence

## Workflow
1. Assignment Service verifies trip state.
2. SQL transaction assigns driver.
3. Driver marked BUSY.
4. User & driver notified.

---

## Sample SQL Transaction

```sql
BEGIN;

UPDATE trips
SET driver_id = 'D101',
    status = 'ASSIGNED'
WHERE trip_id = 'T900';

UPDATE drivers
SET status = 'BUSY'
WHERE driver_id = 'D101';

COMMIT;
```

---

## Flow Diagram

```text
Assignment Service
   ↓
SQL Transaction
   ↓
Trip Assigned
   ↓
User & Driver Notification
```

---

# Final High-Level Architecture

```text
Driver App
   ↓
Location Service
   ↓
Redis GEO Store
   ↓
Kafka


User Request
   ↓
Matching Service
   ↓
Redis GEOSEARCH
   ↓
Filtering + Ranking
   ↓
Notification Service
   ↓
Driver Accept
   ↓
Redis Lock
   ↓
Assignment Service
   ↓
SQL DB
   ↓
Trip Assigned
```

---

# Technologies Used

| Use Case | Technology |
|---|---|
| GPS Tracking | Mobile GPS SDK |
| Nearby Search | Redis GEO |
| Event Streaming | Kafka |
| Notifications | FCM / APNS |
| Real-Time Updates | WebSocket |
| Persistent Storage | PostgreSQL/MySQL |
| Distributed Locking | Redis |
| ETA Calculation | Google Maps / OpenStreetMap |

---

## Interview-Friendly Summary

This system uses Redis GEO for ultra-fast nearby driver discovery, Kafka for real-time event streaming, and distributed locking to ensure only one driver gets assigned to a request atomically. The overall architecture is optimized for low latency, high scalability, real-time tracking, and concurrent request handling, which are critical for platforms like Uber and Swiggy.