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

### Redis ZSET Internals

Redis Sorted Set internally uses 2 data structures together:
```
    - Hashmap: 
        (member -> score) like [user_1 -> 9800, user_2 -> 8700]
        Purpose:
            O(1) member lookup
            Fetch current score quickly
            Check if member exists
    - Skip List
        Stores sorted data as: (score, member) like [(8700, user_2), (9800, user_1)]
        Purpose:
            Maintain sorted ordering
            Rank calculation
            Top N retrieval
            Nearby users traversal
        Avg Complexity: O(logN)
```
## How Redis Achieves Efficient Operations

| Operation        | Redis Command                    | Internal Working                                                                           | Complexity     |
| ---------------- | -------------------------------- | ------------------------------------------------------------------------------------------ | -------------- |
| Update score     | `ZINCRBY leaderboard +50 user_1` | HashMap finds member quickly → SkipList removes old score node → reinserts new sorted node | `O(log N)`     |
| Get rank         | `ZREVRANK leaderboard user_1`    | SkipList traverses levels & uses span metadata to compute rank                             | `O(log N)`     |
| Get Top N users  | `ZREVRANGE leaderboard 0 99`     | SkipList jumps to highest rank → sequential traversal for next M nodes                     | `O(log N + M)` |
| Get nearby users | `ZRANGE leaderboard start end`   | SkipList finds user position → traverses neighboring nodes                                 | `O(log N + M)` |
| Get user score   | `ZSCORE leaderboard user_1`      | Direct HashMap lookup                                                                      | `O(1)`         |

