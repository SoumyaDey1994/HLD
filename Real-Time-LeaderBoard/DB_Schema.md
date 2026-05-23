## High-Level DB Schema

### - SQL DB is mainly used for:

    - user metadata
    - contest metadata
    - durable score storage
    - audit/history
    - analytics/reporting

    NOT for realtime ranking queries.

### - Tables:

#### 1. users

    | Column       | Type               |
    | ------------ | ------------------ |
    | user_id      | BIGINT / UUID (PK) |
    | username     | VARCHAR            |
    | display_name | VARCHAR            |
    | country      | VARCHAR            |
    | avatar_url   | TEXT               |
    | created_at   | TIMESTAMP          |

#### 2. leaderboards: Represent a contest/game/event leaderboard 

    | Column           | Type               |
    | ---------------- | ------------------ |
    | leaderboard_id   | BIGINT / UUID (PK) |
    | leaderboard_name | VARCHAR            |
    | leaderboard_type | VARCHAR            |
    | start_time       | TIMESTAMP          |
    | end_time         | TIMESTAMP          |
    | status           | VARCHAR            |
    | created_at       | TIMESTAMP          |

#### 3. leaderboard_scores: User for latest persisted score snapshot

    | Column         | Type          |
    | -------------- | ------------- |
    | leaderboard_id | BIGINT / UUID |
    | user_id        | BIGINT / UUID |
    | score          | BIGINT        |
    | updated_at     | TIMESTAMP     |

#### 4. score_events: Immutable event log for auditing/replay (Append only, leveragin Kafka-connect JDBC sink)

    | Column         | Type               |
    | -------------- | ------------------ |
    | event_id       | BIGINT / UUID (PK) |
    | leaderboard_id | BIGINT / UUID      |
    | user_id        | BIGINT / UUID      |
    | score_delta    | INT                |
    | event_type     | VARCHAR            |
    | source         | VARCHAR            |
    | created_at     | TIMESTAMP          |


#### 5. leaderboard_rank_snapshots (Optional): Used only if business needs historical rankings.

    | Column         | Type      |
    | -------------- | --------- |
    | snapshot_id    | BIGINT    |
    | leaderboard_id | BIGINT    |
    | user_id        | BIGINT    |
    | rank           | INT       |
    | score          | BIGINT    |
    | snapshot_time  | TIMESTAMP |

### Recommended Indexes

    - leaderboard_scores: 
        - PRIMARY KEY (leaderboard_id, user_id)
        - INDEX (leaderboard_id, score DESC)

    - score_events:
        - INDEX (leaderboard_id, user_id)
        - INDEX (created_at)
