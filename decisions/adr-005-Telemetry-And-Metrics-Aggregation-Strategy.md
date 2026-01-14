# ADR-005: Telemetry and Metrics Aggregation Strategy

## Status
Accepted (Updated 2026-01-14)

## Date
Original: 2025-12-17
Updated: 2026-01-05, 2026-01-06, 2026-01-07, 2026-01-14

## Context

The system ingests real-time market data from Binance via two C++ feed handlers and publishes them into a kdb+/KDB-X pipeline (Tickerplant -> RDB -> RTE):
- **Trade Feed Handler**: Processes individual trade executions
- **Quote Feed Handler**: Maintains L5 order book snapshots from depth updates

To understand system behaviour, diagnose issues, and validate latency targets, we need:
- Visibility into latency at each pipeline stage (per handler type)
- Throughput metrics to understand load
- Health indicators to detect problems (per handler)
- Aggregated views suitable for dashboards and alerting
- Ability to compare trade vs quote handler performance

ADR-001 defines raw timestamp fields captured per event:
- `fhParseUs`, `fhSendUs` - feed handler segment latencies (both handlers)
- `fhRecvTimeUtcNs`, `tpRecvTimeUtcNs`, `rdbApplyTimeUtcNs` - wall-clock timestamps for cross-process correlation

A decision is required on **how these raw measurements are aggregated, stored, and consumed**, with support for multiple handler types.

## Notation

| Acronym | Definition |
|---------|------------|
| FH | Feed Handler |
| IPC | Inter-Process Communication |
| RDB | Real-Time Database |
| RTE | Real-Time Engine |
| SLO | Service-Level Objective |
| TEL | Telemetry Process |
| TP | Tickerplant |

## Decision

Telemetry will be collected as **raw per-event measurements** and **aggregated in a dedicated TEL process** that queries RDB and RTE using **persistent IPC handles**, and subscribes to health updates via TP.

### Persistent IPC Handles (Added 2026-01-07)

TEL queries RDB and RTE every 5 seconds for telemetry data. To minimize overhead, TEL maintains **persistent IPC connections** rather than opening/closing connections each query cycle.

**Handle Management:**
```q
/ Handle dictionary: port -> handle
.tel.h:(`int$())!`int$();

/ Check if handle exists and is valid
.tel.hasValidH:{[p]
  (p in key .tel.h) and not null .tel.h[p]
  };

/ Get or create handle (lazy connection)
.tel.getH:{[p]
  if[not .tel.hasValidH[p];
    h:@[hopen; `$"::",string p; 0N];
    if[not null h; .tel.h[p]:h; -1 "TEL: Connected to port ",string p];
  ];
  .tel.h[p]
  };

/ Safe query with automatic reconnect on error
.tel.safeQuery:{[port;query]
  h:.tel.getH[port];
  if[null h; :.tel.queryError];
  r:@[h; query; {.tel.h[x]_:.tel.h[x]; .tel.queryError}[port]];
  r
  };
```

**Benefits:**
- Eliminates TCP handshake overhead (4 handshakes/tick → 0)
- Reduces latency ~4-12ms per 5-second tick
- Reduces file descriptor churn
- Automatic reconnection on connection loss

**Connection Lifecycle:**
- Pre-warm connections at TEL startup
- Lazy reconnection on query failure
- Graceful close on TEL shutdown (`.z.exit`)
- Diagnostic function: `.tel.handleStatus[]`

### Telemetry Categories

| Category | Metrics | Source |
|----------|---------|--------|
| FH segment latencies | `fhParseUs`, `fhSendUs` per handler | Per-event fields from FH (via RDB query) |
| Pipeline latencies | `fh_to_tp_ms`, `tp_to_rdb_ms`, `e2e_system_ms` | Derived from wall-clock timestamps (via RDB query, trades and quotes) |
| Throughput | Events per second, per symbol, per handler | Derived from `cnt` field in latency tables |
| FH health | `uptimeSec`, `msgsReceived`, `connState` per handler | Published by FH → TP → TEL (subscription) |
| System metrics | Memory usage per process | Queried from RDB, RTE, captured from TEL |

### Aggregation Location

**Raw per-event latencies are sent by the FH; aggregation happens in a dedicated TEL process.**

| Component | Responsibility |
|-----------|----------------|
| Trade FH | Captures `fhParseUs`, `fhSendUs` per trade; publishes trades and health to TP |
| Quote FH | Captures `fhParseUs`, `fhSendUs` per L5 snapshot; publishes quotes and health to TP |
| TP | Distributes trades/quotes to RDB, health to TEL; adds `tpRecvTimeUtcNs` |
| RDB | Stores raw trade/quote events with `rdbApplyTimeUtcNs`; serves queries |
| RTE | Computes VWAP/imbalance from market data; serves memory queries |
| TEL | Subscribes to health via TP; queries RDB and RTE on timer (persistent handles); computes aggregated telemetry |
| Dashboard | Queries TEL for all monitoring data |

Rationale:
- RDB remains focused on market data storage (single responsibility)
- RTE remains focused on analytics (single responsibility)
- TEL is the single source of truth for all monitoring/telemetry
- Health data flows through TP (consistent pub/sub pattern)
- TEL can query multiple sources (RDB + RTE) for comprehensive metrics
- **Persistent handles minimize query overhead**
- Telemetry logic can evolve without affecting storage or analytics
- Raw per-event latencies remain available in RDB for debugging
- Dashboards query only TEL for system monitoring (single endpoint)
- Handler-specific metrics enable performance comparison

### Collection Architecture

```
Trade FH ---> TP :5010 --+--> RDB :5011 (trade_binance, quote_binance)
Quote FH -------^        |
                         +--> RTE :5012 (tradeBuckets, .rte.imb.latest)
                         |
                         +--> TEL :5013 (health_feed_handler subscription)
                                  |
                                  +-- queries RDB via persistent handle
                                  +-- queries RTE via persistent handle
                                  |
                                  +-- telemetry_latency_fh (unified: trade + quote)
                                  +-- telemetry_latency_e2e (trades and quotes)
                                  +-- telemetry_system (RDB, RTE, TEL memory)
                                  +-- health_feed_handler (from TP subscription)
```

### Aggregation Strategy

**Time buckets:**

| Parameter | Value | Rationale |
|-----------|-------|-----------|
| Bucket size | 5 seconds | Stable percentiles with sufficient samples (~50-500 events) |
| Computation frequency | Every 5 seconds (timer) | Matches bucket size |

**Percentiles (per bucket):**

| Percentile | Purpose |
|------------|---------|
| p50 | Typical latency (median) |
| p95 | Majority-case tail |
| max | Spike detection |

### Telemetry Storage Schema

**Feed handler latency aggregates (`telemetry_latency_fh`):**

| Column | Type | Description |
|--------|------|-------------|
| `bucket` | timestamp | Start of 5-second bucket |
| `sym` | symbol | Instrument symbol |
| `handler` | symbol | Handler type (`trade_fh` or `quote_fh`) |
| `parseUs_p50` | float | Median parse latency |
| `parseUs_p95` | float | 95th percentile parse latency |
| `parseUs_max` | long | Maximum parse latency |
| `sendUs_p50` | float | Median send latency |
| `sendUs_p95` | float | 95th percentile send latency |
| `sendUs_max` | long | Maximum send latency |
| `cnt` | long | Number of events in bucket |

**End-to-end latency aggregates (`telemetry_latency_e2e`):**

| Column | Type | Description |
|--------|------|-------------|
| `bucket` | timestamp | Start of 5-second bucket |
| `sym` | symbol | Instrument symbol |
| `handler` | symbol | Handler type (`trade_fh` or `quote_fh`) |
| `fhToTpMs_p50` | float | Median FH to TP latency (ms) |
| `fhToTpMs_p95` | float | 95th percentile FH to TP latency |
| `fhToTpMs_max` | float | Maximum FH to TP latency |
| `tpToRdbMs_p50` | float | Median TP to RDB latency (ms) |
| `tpToRdbMs_p95` | float | 95th percentile TP to RDB latency |
| `tpToRdbMs_max` | float | Maximum TP to RDB latency |
| `e2eMs_p50` | float | Median end-to-end latency (FH to RDB) |
| `e2eMs_p95` | float | 95th percentile E2E latency |
| `e2eMs_max` | float | Maximum E2E latency |
| `cnt` | long | Number of events in bucket |

**System metrics (`telemetry_system`):**

| Column | Type | Description |
|--------|------|-------------|
| `bucket` | timestamp | Start of 5-second bucket |
| `component` | symbol | Process name (`RDB`, `RTE`, `TEL`) |
| `heapMB` | float | Heap size (megabytes) |
| `usedMB` | float | Memory used (megabytes) |
| `peakMB` | float | Peak memory (megabytes) |

### Feed Handler Health (via TP Subscription)

**Health table (`health_feed_handler`):**

| Column | Type | Description |
|--------|------|-------------|
| `time` | timestamp | Time health was published |
| `handler` | symbol | Handler identifier (`trade_fh`, `quote_fh`) |
| `startTimeUtc` | timestamp | When handler process started |
| `uptimeSec` | long | Seconds since process start |
| `msgsReceived` | long | Total messages received from Binance |
| `msgsPublished` | long | Total messages published to TP |
| `lastMsgTimeUtc` | timestamp | Time of last received message |
| `lastPubTimeUtc` | timestamp | Time of last publish to TP |
| `connState` | symbol | Connection state (`connected`, `reconnecting`, `disconnected`) |
| `symbolCount` | int | Number of subscribed symbols |

**Publication frequency:** Every 5 seconds per handler

### Retention Policy

**Telemetry is ephemeral**, consistent with ADR-003 system-wide stance:

| Aspect | Policy |
|--------|--------|
| Persistence | In-memory only |
| On restart | Telemetry tables start empty |
| Historical analysis | Out of scope for current phase |
| Telemetry bucket retention | Keep last 15 minutes |
| Health record retention | Keep last 1 minute per handler |

### Connection Diagnostics

TEL provides handle status for debugging:
```q
.tel.handleStatus[]
/ Returns:
/ port  handle status
/ ---------------------
/ 5011  8      connected
/ 5012  9      connected
```

### Dashboard View Functions

TEL provides pre-formatted view functions for dashboard consumption:

| Function | Description |
|----------|-------------|
| `.tel.vsFhStatus[]` | Feed handler status grid (formatted) |
| `.tel.vsSystemResources[]` | System memory per component |
| `.tel.vsDataVolume[]` | Trade/quote counts per symbol |
| `.tel.fhStatusTable[]` | Raw feed handler health with computed fields |

### Throughput Derivation

Throughput (events per second) is derived from the `cnt` field in latency tables:

```q
/ Events per second = cnt / bucket_size
/ Example: 250 events in 5-second bucket = 50 events/sec
eventsPerSec: cnt % 5
```

No separate `telemetry_throughput` table is needed — throughput is calculated from existing data.

### Analytics Health

Analytics validity is queried directly from RTE rather than stored in TEL:

```q
/ Query RTE for VWAP validity
.rte.getVwap[`BTCUSDT; 5]  / Returns isValid field

/ Query RTE for var-covar validity  
.rte.getVcov[]  / Returns isValid field

/ Query RTE for order book imbalance
.rte.getImbalance[`BTCUSDT]  / Returns current state
```

Dashboards query RTE directly for analytics health indicators.

## Rationale

This approach was selected because it:

- Keeps the RDB focused on market data storage (single responsibility)
- Keeps the RTE focused on analytics (single responsibility)
- Makes TEL the single source of truth for all monitoring
- Uses consistent pub/sub pattern for health (FH → TP → TEL)
- **Persistent handles minimize IPC overhead** (2026-01-07)
- Centralises telemetry logic in TEL where it can evolve easily
- Allows TEL to query multiple sources (RDB + RTE)
- Preserves raw per-event latencies in RDB for debugging
- Provides aggregated views suitable for dashboards
- Aligns with the ephemeral, exploratory nature of the project
- Uses 5-second buckets for stable percentiles with sufficient samples
- Unified handler table enables performance comparison
- Handler column provides flexibility for future handler types
- Derives throughput from existing data (no redundant tables)
- Queries RTE directly for analytics health (no duplication)

## Alternatives Considered

### 1. Open/close IPC connections each query cycle (original design)
Changed (2026-01-07):
- 4 TCP handshakes per 5-second tick
- Unnecessary overhead (~4-12ms per tick)
- File descriptor churn
- Persistent handles are simple and more efficient

### 2. Telemetry computed in RDB
Rejected:
- Mixed storage and computation responsibilities
- Could not easily capture RTE health metrics
- Violated single-responsibility principle

### 3. FH pre-aggregates into buckets
Rejected:
- Adds complexity to C++ feed handlers
- Reduces flexibility (aggregation logic in compiled code)
- Loses raw per-event data for debugging

### 4. Separate throughput table
Rejected:
- Redundant — throughput is trivially derived from `cnt` field
- Adds table maintenance overhead
- No additional value

### 5. Separate analytics health table in TEL
Rejected:
- Would require TEL to query RTE for validity flags
- Duplicates data already available in RTE
- Dashboard already queries RTE for analytics data

## Consequences

### Positive

- Clean separation of concerns (RDB stores market data, RTE computes analytics, TEL handles all monitoring)
- TEL is single source of truth for monitoring (dashboards query one endpoint for telemetry)
- **Persistent handles eliminate connection overhead** (2026-01-07)
- Automatic reconnection on connection loss
- Health uses consistent pub/sub pattern (FH → TP → TEL)
- Flexible aggregation logic (evolves in TEL without affecting RDB/RTE)
- Raw data available in RDB for debugging
- Aggregated data available in TEL for dashboards
- Consistent ephemeral stance
- SLO metrics directly queryable
- 5-second buckets provide stable percentiles
- Unified handler table enables performance comparison
- E2E latency tracked for both trades and quotes
- No redundant tables (throughput derived, analytics health queried)

### Negative / Trade-offs

- Additional process (TEL) to manage
- Telemetry delayed by query interval (5 seconds)
- Telemetry lost on restart
- No historical trend analysis
- 15-minute bucket retention limits lookback
- Handler comparison requires filtering on `handler` column
- Persistent handles require explicit management (close on shutdown)
- Analytics health requires dashboard to query RTE directly

These trade-offs are acceptable for the current phase.

## Links / References

- `../kdbx-real-time-architecture-reference.md`
- `adr-001-timestamps-and-latency-measurement.md` (raw fields, SLO targets)
- `adr-003-tickerplant-logging-and-durability-strategy.md` (ephemeral stance)
- `adr-004-real-time-rolling-analytics-computation.md` (RTE details)
- `adr-007-visualisation-and-consumption-strategy.md` (dashboard consumption)
- `adr-009-Order-Book-Architecture.md` (quote handler details)
- `kdb/tel.q` (implementation)
