## NFR Considerations

### 1. Redis Crash Recovery Strategy
    - Goal: Recover leaderboard with minimal downtime

    - Stategies:
        1. Redis Replication: If primary crashes, replica promoted & takes over (Redis Senitel & Redis Cluster)
        2. Full Cluster Failure Recovery: Use Kafka as durable & replayable score log to construct the leaderboard (Leaderboard Rebuilder using Redis sorted set - ZINCBY)
        3. Faster Recovery Optimization: Redis Snapshot + Replay recent kafka events (delta changes post last snapshot)

    - Overall Architecture Insight
        Redis  -> realtime ranking
        Kafka -> durability & replay
        WebSocket -> live updates
        DB -> long-term persistence
        Flink/Aggregator -> write optimization

### 2. Common Bottlenecks & Solutions

    | Problem                 | Solution              |
    | ----------------------- | --------------------- |
    | Redis memory issue      | Sharding              |
    | Hot leaderboard         | Partitioning          |
    | Too many writes         | Batching              |
    | Real-time updates       | WebSockets            |
    | Redis crash             | Kafka replay          |
    | Millions of users       | Redis Cluster         |
    | Rank recalculation cost | Redis ZSET handles it |
