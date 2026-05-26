# Uber ETA Computation — System Design Summary

# 1. Trip Starts

Known Inputs:
- Driver current GPS `(lat,long)`
- Rider destination `(lat,long)`

Driver app continuously sends GPS updates every few seconds.

---

# 2. Location Ingestion

## Service: Location Service

### Responsibilities
- Receive driver GPS updates
- Validate coordinates
- Store latest driver location
- Publish location events

### Data Store
- Redis GEO / Redis Cluster
  - Stores latest driver GPS point
  - Enables fast nearby queries

### Communication Flow

```text
Driver App
    |
    | WebSocket/gRPC
    v
Location Service
    |
    +--> Redis GEO (latest driver location)
    |
    +--> Kafka Topic: driver-location-events
```

---

# 3. Map Matching

## Service: Map Matching Service

### Responsibilities
- Consume GPS events from Kafka
- Snap GPS → nearest road segment
- Determine:
  - current road
  - direction
  - lane alignment

### Communication Flow

```text
Kafka(driver-location-events)
            |
            v
Map Matching Service
            |
            +--> Kafka(mapped-location-events)
```

### Example Output

```text
Driver D1 → Road Segment R1023
```

---

# 4. Build Road Graph

## Services:
- Routing Service
- Map Service

### Road Network Representation

```text
Node = Intersection
Edge = Road Segment
Weight = Expected Travel Time
```

### Edge Metadata
- distance
- speed limit
- live traffic speed
- restrictions
- road type

---

# 5. Traffic Computation

## Service: Traffic Service

### Responsibilities
Compute:
```text
Expected Speed per Road Segment
```

### Inputs
- Nearby driver speeds
- Historical traffic
- Time-of-day patterns
- Weather/events
- Signal congestion

### Communication Flow

```text
GPS Streams
     |
     v
Traffic Service
     |
     +--> Redis / In-Memory Cache
     |
     +--> Road Segment Speed Updates
```

### Example

```text
R1023 → 20 km/h
R7781 → 45 km/h
```

---

# 6. Compute Best Route

## Service: Routing Service

### Responsibilities
- Find fastest path
- Use graph algorithms:
  - A*
  - Dijkstra
  - Contraction Hierarchies

### Inputs
- Current road segment
- Destination
- Traffic weights

### Communication Flow

```text
Map Matching Service
        |
        v
Routing Service
        |
        +--> Map Service
        |
        +--> Traffic Service
```

### Goal

```text
Current Road → Destination
```

### Output

```text
R1 → R5 → R8 → R12
```

---

# 7. ETA Computation

## Service: ETA Service

### Responsibilities
Refine final ETA using:
- segment travel time
- signal delays
- driver behavior
- pickup wait
- ML prediction
- weather impact
- city traffic patterns

### Formula

ETA = Σ(distance_i / expectedSpeed_i)

### Communication Flow

```text
Routing Service
      |
      v
ETA Service
      |
      +--> ML Prediction Service
```

### Example Output

```text
ETA = 14 mins
```

---

# 8. Push ETA to Rider

## Service: Notification Gateway

### Communication Flow

```text
ETA Service
     |
     | WebSocket / Push
     v
Rider App
```

### Rider View

```text
Driver arriving in 14 mins
```

---

# 9. Continuous Recalculation Loop

Every GPS update triggers:

```text
Driver GPS Update
        |
        v
Location Service
        |
        +--> Redis GEO Update
        |
        +--> Kafka(driver-location-events)
                    |
                    v
            Map Matching Service
                    |
                    +--> Kafka(mapped-location-events)
                                |
                                v
                        Routing Service
                                |
                                +--> Traffic Service
                                |
                                v
                           ETA Service
                                |
                                v
                     Notification Gateway
                                |
                                v
                            Rider App
```

---

# Key Architectural Separation

| Service | Responsibility |
|---|---|
| Location Service | GPS ingestion |
| Redis GEO | Latest driver location store |
| Kafka | Real-time event streaming |
| Map Matching Service | GPS → road mapping |
| Traffic Service | Road congestion computation |
| Routing Service | Fastest path computation |
| ETA Service | Accurate ETA prediction |
| Notification Gateway | Push ETA updates |

---

# Important Edge Cases

## 1. Driver Route Deviation
- Driver leaves suggested route
- Recompute path from current location

---

## 2. Sudden Traffic Spike
- Accident/heavy congestion
- Traffic service updates edge weights
- Routing recomputes best path

---

## 3. GPS Noise / Tunnel Loss
Handling:
- map matching heuristics
- dead reckoning
- previous direction prediction

---

## 4. Road Closure / Construction
- Invalidate graph edges
- Trigger rerouting

---

## 5. ETA Fluctuation
Prevent poor UX using:
- ETA smoothing
- threshold-based updates
- confidence intervals

---

# Scalability Considerations

System must support:
- millions of GPS updates/sec
- low latency routing
- continuous ETA recomputation

Typical Infra:
- Kafka/Pulsar
- Redis GEO
- H3/S2 spatial indexing
- Distributed routing engines
- WebSockets/gRPC

---

# Interview-Friendly Final Statement

> Uber computes ETA by continuously ingesting driver GPS updates, storing live locations in Redis, streaming events through Kafka, map-matching drivers to road segments, computing the fastest path over a traffic-weighted road graph, refining ETA using live traffic and ML models, and continuously recomputing routes and ETA as new location and traffic updates arrive.
