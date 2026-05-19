# Real-Time Leaderboard System Design

A Real-time Leaderboard system is used in:

    Gaming platforms
    Fantasy sports
    Coding contests
    Sales dashboards
    Fitness apps

Core requirement: Continuously rank users based on score updates with very low latency.


### 1. Functional Requirements
    - Core Features:
        Update player/user score in real time
    - Fetch:
        Top N players
        Rank of a user
        Nearby ranks around a user
    - Handle millions of updates/sec
    - Real-time propagation to clients

### 2. Non-Functional Requirements

    | Requirement     | Target                          |
    | --------------- | ------------------------------- |
    | Read latency    | < 50 ms                         |
    | Write latency   | < 20 ms                         |
    | High throughput | Millions of updates             |
    | Scalability     | Horizontal                      |
    | Availability    | Very high                       |
    | Consistency     | Eventual consistency acceptable |


### 3. High-Level Architecture

                        ┌─────────────────┐
                        │   Client Apps   │
                        └────────┬────────┘
                                │
                        WebSocket/API
                                │
                        ┌────────▼────────┐
                        │  API Gateway    │
                        └────────┬────────┘
                                │
                        ┌────────▼────────┐
                        │ Leaderboard API │
                        └────────┬────────┘
                                │
                ┌────────────────┼────────────────┐
                │                │                │
                │                │                │
        ┌───────▼───────┐ ┌──────▼───────┐ ┌──────▼──────┐
        │ Score Updater │ │ Query Engine │ │ WS Broadcaster│
        └───────┬───────┘ └──────┬───────┘ └──────┬──────┘
                │                │                │
                │                │                │
        ┌──────▼────────────────▼────────────────▼─────┐
        │               Redis Cluster                  │
        │   Sorted Sets (Primary Ranking Engine)       │
        └───────────────────┬──────────────────────────┘
                            │
                    Async Persistence
                            │
                    ┌───────▼────────┐
                    │ Kafka / Queue  │
                    └───────┬────────┘
                            │
                    ┌───────▼────────┐
                    │ Cassandra/DB   │
                    └────────────────┘

### 4. Core Design Idea

The heart of leaderboard systems is:

    Redis Sorted Set (ZSET)

Redis internally maintains: (score, member)

Example:
```
    (9800, user_1)
    (8700, user_2)
    (8600, user_3)
```
It automatically keeps data sorted.

This gives:
```
    Operation	        Complexity
    Update score      	O(log N)
    Get top N     	    O(log N + M)
    Get rank      	    O(log N)
```
Perfect for real-time ranking.