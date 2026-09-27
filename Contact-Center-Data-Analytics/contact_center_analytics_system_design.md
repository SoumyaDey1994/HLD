# Contact Center Analytics Pipeline & Reporting System Design

## 1. System & Reporting Requirements

A Voice AI Contact Center Desktop App continuously emits agent activity events:

```text
LOGIN
APP_LAUNCHED
MIC_ENABLED / MIC_DISABLED
SCREEN_LOCKED / SCREEN_UNLOCKED
CALL_STARTED / CALL_ENDED
LOGOUT
```

Typical agent session: **6–9 hours**.

### Metrics

Measured values / durations:

- Total Session Duration
- Total Engaged Duration
- Total Call Duration
- Number of Calls Taken
- Average Call Duration
- Agent Status / Current Status

### KPIs

Primarily live counters:

- Total Active Users
- Engaged Users
- Non-Engaged Users
- Users in Call
- Users with Mic Disabled
- Users Logged In

---

## 2. Overall Architecture

```text
                    ┌─────────────────────┐
                    │    Desktop App      │
                    │   Agent Events      │
                    └──────────┬──────────┘
                               │ HTTPS
                               ▼
                    ┌─────────────────────┐
                    │ Event Ingestion     │
                    │    Web Service      │
                    └──────────┬──────────┘
                               │
                               ▼
                          ┌─────────┐
                          │  Kafka  │
                          └────┬────┘
                               │
                 ┌─────────────┼─────────────┐
                 │             │             │
                 ▼             ▼             ▼
          S3 + Iceberg   ClickHouse RAW    Flink
          RAW Source          Events      Stateful
          of Truth                         Processing
                              │               │
                              │               ▼
                              │        hourly_metrics
                              │               │
                              │               ▼
                              │        daily_metrics
                              │
                              ▼
                       ClickHouse MV
                              │
                              ▼
                       Live KPI Rollups
                              │
                 ┌────────────┴────────────┐
                 │                         │
                 ▼                         ▼
          Reporting API ◄──────────── Metrics
                 │
                 ▼
          Reporting Portal
```

**Core storage principle**

- **Desktop App → Web Service → Kafka** → secure event ingestion boundary
- **S3 + Iceberg** → durable RAW source of truth
- **ClickHouse** → reporting/serving database
- **Flink** → stateful/complex metric computation
- **ClickHouse MVs** → near-real-time count KPIs

The Desktop App does **not connect directly to Kafka**. The ingestion Web Service handles authentication, validation, normalization, rate limiting and Kafka publishing.

All event timestamps stored in ClickHouse are **UTC**.

---

## 3. Initial Approach — CRON + SQL

Before introducing Flink, use a simpler batch-oriented approach.

```text
RAW Events in ClickHouse
        │
        ├── CRON + SQL
        │      ↓
        │  Daily Metrics
        │
        └── Current / Partial Day
               ↓
          RAW SQL Query
```

For a report covering multiple days:

```text
Daily Aggregates
     +
Current-day RAW SQL result
     ↓
Combined Report
```

### Why this eventually becomes expensive

- Increasing RAW event volume
- Long reporting ranges
- Many concurrent report requests
- Repeated scans/aggregations
- Increasing ClickHouse CPU/I/O usage

This motivates stateful stream processing with Flink.

---

## 4. Scalable Processing — Flink

Flink consumes Kafka events and maintains state per agent/session.

Example state:

```text
(agent_id, session_id)

current_status
session_start
last_event_time
call_count
total_call_duration
engaged_duration
current_call_start
```

It incrementally computes:

```text
Session Duration
Engaged Duration
Call Count
Total Call Duration
Agent Status
```

Then writes precomputed data into:

```text
ClickHouse
 ├── hourly_metrics
 └── daily_metrics
```

This avoids repeatedly scanning large RAW event datasets.

### Average Call Duration

Do not aggregate averages directly.

Store:

```text
call_count
total_call_duration
```

and derive:

```text
avg_call_duration =
    total_call_duration / call_count
```

---

## 5. ClickHouse Storage Model

### `raw_events`

```sql
CREATE TABLE raw_events
(
    event_id       UUID,
    workspace_id   String,
    agent_id       String,
    session_id     UUID,
    event_type     LowCardinality(String),
    event_time     DateTime64(3, 'UTC'),
    metadata       String
)
ENGINE = MergeTree
PARTITION BY toDate(event_time)
ORDER BY (workspace_id, agent_id, event_time, event_id);
```

Example:

```text
event_id     : 8f...
workspace_id : W100
agent_id     : A123
session_id   : S456
event_type   : CALL_STARTED
event_time   : 2026-09-25 09:32:15 UTC
```

### `hourly_metrics`

```sql
CREATE TABLE hourly_metrics
(
    workspace_id            String,
    agent_id                String,
    hour_start              DateTime('UTC'),
    session_duration_sec    UInt64,
    engaged_duration_sec    UInt64,
    call_count              UInt32,
    total_call_duration_sec UInt64,
    agent_status            LowCardinality(String),
    version                 UInt64
)
ENGINE = ReplacingMergeTree(version)
PARTITION BY toDate(hour_start)
ORDER BY (workspace_id, agent_id, hour_start);
```

`ReplacingMergeTree` helps when hourly aggregates need correction/replacement because of late events or recomputation.

### `daily_metrics`

```sql
CREATE TABLE daily_metrics
(
    workspace_id            String,
    agent_id                String,
    metric_date             Date,
    session_duration_sec    UInt64,
    engaged_duration_sec    UInt64,
    call_count              UInt32,
    total_call_duration_sec UInt64,
    agent_status            LowCardinality(String),
    version                 UInt64
)
ENGINE = ReplacingMergeTree(version)
PARTITION BY toYYYYMM(metric_date)
ORDER BY (workspace_id, agent_id, metric_date);
```

All timestamps/dates used for processing are based on **UTC**.

### Hourly → Daily Relationship

`daily_metrics` is derived from `hourly_metrics`:

```text
hourly_metrics
      │
      │ aggregate hourly buckets
      ▼
daily_metrics
```

Additive values such as `call_count`, `total_call_duration_sec`, and `engaged_duration_sec` can be summed. Average values should be recomputed from daily totals:

```text
avg_call_duration =
    daily_total_call_duration / daily_call_count
```

State-based fields such as `agent_status` require appropriate latest/final-state handling rather than summation.

---

## 6. ClickHouse Materialized Views — Live KPIs

Count-based KPIs are good candidates for ClickHouse Materialized Views because they require incremental, near-real-time updates.

```text
RAW Events
    │
    ▼
ClickHouse Materialized View
    │
    ▼
KPI Rollup Table
    │
    ▼
Reporting API
```

Examples:

```text
Active Users
Engaged Users
Non-Engaged Users
Users in Call
Users with Mic Disabled
```

### MV Engine

Use **`AggregatingMergeTree`** for flexible aggregate-state based rollups:

```text
countState()
uniqState(agent_id)
...
```

and query using:

```text
countMerge()
uniqMerge()
```

`SummingMergeTree` can also be used for purely additive counters.

### Important: Don't Overuse MVs

MVs are not free. Every incoming event may trigger processing across multiple MVs.

Too many/complex MVs can cause:

- Higher CPU
- Higher memory usage
- More disk writes
- More background merges
- Higher ingestion latency
- Increased operational complexity

Therefore:

> Use MVs selectively for important, frequently queried, near-real-time KPIs.

Complex/stateful metrics such as session duration and engaged duration are better handled by **Flink**.

### MV vs Hourly/Daily Metrics

These are **separate computation paths**:

```text
RAW Events
   │
   ├──────────────► ClickHouse MV ──► Live KPI Rollups
   │
   └──────────────► Flink ──► hourly_metrics ──► daily_metrics
```

The MV rollups do **not** directly generate `hourly_metrics` or `daily_metrics`.

---

## 7. ClickHouse Caching Strategy

Use caching as an optimization, not as a replacement for proper pre-aggregation.

### ClickHouse Query Cache

Useful for repeated identical queries.

### Application / Redis Cache

Cache frequently requested reports:

```text
workspace
+ filters
+ date_range
+ timezone
      ↓
   Cache Key
```

### Pre-aggregation

The primary optimization remains:

```text
RAW Events
    ↓
Hourly / Daily Aggregates
    ↓
Reporting Queries
```

rather than repeatedly scanning RAW events.

---

## 8. Timezone Handling — Supervisor/Admin Local Time

There are two valid strategies.

### Strategy 1 — Store UTC + Convert at Query Time

**Store all timestamps in UTC.**

The Portal knows the supervisor/admin's configured timezone. The Reporting API converts the requested local range into UTC and selects the appropriate aggregation level.

#### Case A — Local range crosses 2 UTC calendar days

Example:

```text
Timezone: Asia/Kolkata (UTC +5:30)

Local request:
Sep 25, 00:00 → Sep 26, 00:00 IST
```

UTC equivalent:

```text
Sep 24, 18:30 UTC
        →
Sep 25, 18:30 UTC
```

The range crosses two UTC calendar dates.

Therefore query:

```text
Hourly Metrics
  ├── Partial Sep 24
  ├── Full/partial Sep 25
  └── Partial Sep 25
        ↓
   Consolidated Result
```

Use `hourly_metrics` because `daily_metrics` cannot accurately represent partial UTC days.

#### Case B — Local range maps to complete UTC days

Example:

```text
UTC:
Sep 20, 00:00
     →
Sep 23, 00:00
```

The API can directly query:

```text
daily_metrics
 ├── Sep 20
 ├── Sep 21
 └── Sep 22
       ↓
  Consolidated Result
```

#### Decision Logic

```text
User Local Date/Time Range
          │
          ▼
Convert boundaries → UTC
          │
          ▼
Complete UTC calendar days?
       /       \
     YES        NO
      │          │
      ▼          ▼
daily_metrics  hourly_metrics
       \          /
        ▼        ▼
       Consolidated Result
```

**Principle:**

```text
Storage        → UTC
Query Boundary → Local → UTC
Response       → UTC → Local
```

No timezone-specific aggregate tables are required.

---

### Strategy 2 — Precomputed Timezone-Specific Aggregates

If the product supports a **known and manageable list of timezones**, we can precompute aggregates for each supported timezone.

Because ClickHouse storage is generally cheaper than repeatedly spending compute at query time, this can make timezone-aware reporting faster and simpler.

Flink computes timezone-specific aggregates:

```text
Flink
  │
  ├──► UTC
  ├──► Asia/Kolkata
  ├──► America/New_York
  ├──► Europe/London
  └──► Asia/Singapore
        │
        ▼
Timezone-specific
Hourly / Daily Tables
```

For example:

```text
hourly_metrics_utc
hourly_metrics_ist
hourly_metrics_est
hourly_metrics_london

daily_metrics_utc
daily_metrics_ist
daily_metrics_est
daily_metrics_london
```

The Portal/API can then directly query the table corresponding to the supervisor's configured timezone.

#### Trade-offs

**Advantages**

- Faster timezone-aware queries
- Less runtime timezone conversion
- Lower query-time compute
- Simple reporting queries

**Costs**

- More storage
- More Flink processing
- More tables to maintain
- More complex backfill/reprocessing
- Additional operational overhead

Therefore:

```text
Few / known supported timezones
        → Precompute timezone-specific aggregates

Many / arbitrary timezones
        → Store UTC + convert query boundaries at runtime
```

**Key principle:** Choose between runtime timezone conversion and precomputed timezone aggregates based on the number of supported timezones and the storage-vs-compute trade-off.

---

## 9. RAW Data, Backfill & Metric Evolution

S3 + Iceberg is the durable RAW source of truth.

```text
Kafka
  │
  ├──► ClickHouse RAW
  │
  └──► S3 + Iceberg
```

If metric logic changes:

```text
S3 / Iceberg RAW
       ↓
Flink / Batch Processing
       ↓
New Aggregate Version
       ↓
ClickHouse V2
       ↓
Validate
       ↓
Switch Reporting API
       ↓
Retire Old Version
```

Avoid massive in-place updates of large historical ClickHouse datasets.

**Golden rule:**

> Regenerate OLAP aggregates from durable RAW data rather than mass-mutating huge aggregate tables.

This also supports replay, backfill and correction of historical metrics.

---

## 10. End-to-End Reporting Flow

### Complete Reporting Flow

```text
                         Kafka
                           │
             ┌─────────────┼──────────────┐
             │             │              │
             ▼             ▼              ▼
      S3 + Iceberg   ClickHouse RAW     Flink
      RAW Source          Events          │
      of Truth                            ▼
                                   hourly_metrics
                                          │
                                          ▼
                                   daily_metrics
                                          │
                                          │
                                          └──────────┐
                                                     │
ClickHouse RAW Events                                │
        │                                            │
        ▼                                            │
 Materialized View                                  │
        │                                            │
        ▼                                            │
 Live KPI Rollups                                    │
        │                                            │
        └──────────────────┬─────────────────────────┘
                           ▼
                    Reporting API
                           │
              ┌────────────┴────────────┐
              │                         │
       Timezone Handling              Cache
              │                         │
              └────────────┬────────────┘
                           ▼
                    Reporting Portal
```

### Two independent computation paths

**1. Metrics path**

```text
Kafka → Flink → hourly_metrics → daily_metrics → Reporting API
```

Used for:

- Session duration
- Engaged duration
- Call count
- Total call duration
- Average call duration
- Agent status

**2. Live KPI path**

```text
Kafka → ClickHouse RAW → Materialized View → KPI Rollup → Reporting API
```

Used for:

- Active Users
- Engaged Users
- Non-Engaged Users
- Users in Call
- Users with Mic Disabled

The MV KPI path is **independent of** the hourly/daily metrics pipeline.

---

## 11. Responsibility Matrix & Interview Summary

| Component | Core Responsibility |
|---|---|
| Desktop App | Generate agent activity events; send events via HTTPS |
| Event Ingestion Web Service | Authenticate, validate, normalize, rate-limit and publish events to Kafka |
| Kafka | Event transport + buffering |
| S3 + Iceberg | Durable RAW source of truth |
| ClickHouse RAW | Event-level/realtime querying |
| CRON + SQL | Initial/simple daily aggregation |
| Flink | Stateful session + complex metric computation; produces hourly metrics |
| ClickHouse MV | Independent near-real-time count-based KPI rollups |
| Hourly Metrics | Realtime/partial-day metric reporting; source for daily aggregates |
| Daily Metrics | Historical reporting derived from hourly aggregates |
| Redis / Cache | Frequently repeated report results |
| Reporting API | Query correct layer, combine results, timezone conversion |
| Reporting Portal | Supervisor/admin visualization |

### Interview-Ready Summary

The design evolves naturally:

```text
Phase 1
RAW ClickHouse
   ↓
CRON + SQL
   ↓
Daily Aggregates
+
RAW SQL for current/partial day
```

↓

```text
Phase 2
Kafka
   ↓
Flink
   ↓
Hourly + Daily Metrics
   ↓
ClickHouse
```

+

```text
Live Count KPIs
RAW Events
   ↓
ClickHouse MV
   ↓
AggregatingMergeTree
   ↓
Near-real-time KPI
```

+

```text
S3 + Iceberg
   ↓
Long-term RAW Source
   ↓
Replay / Backfill / Reprocessing
```

**Core principle:**

> **The Web Service provides the secure ingestion boundary; Flink computes complex/stateful metrics through the hourly → daily pipeline; ClickHouse MVs independently handle selective live counters; and S3/Iceberg preserves the authoritative RAW event history.**
