# ADR-005: Telemetry and Metrics Aggregation Strategy

## Status
Accepted (Updated 2026-01-18)

## Date
Original: 2025-12-17
Updated: 2026-01-05, 2026-01-06, 2026-01-07, 2026-01-14, 2026-01-16, 2026-01-17, 2026-01-18

## Context

The system ingests real-time market data from Binance via two C++ feed handlers and publishes them into a kdb+/KDB-X pipeline:
- **Trade Feed Handler**: Processes individual trade executions
- **Quote Feed Handler**: Maintains L5 order book snapshots from depth updates

To understand system behaviour, diagnose issues, and validate latency targets, we need:
- Visibility into feed handler latency (per handler type)
- Health indicators to detect problems (per handler)
- Aggregated views suitable for dashboards

ADR-001 defines raw timestamp fields captured per event:
- `fhParseUs`, `fhSendUs` - feed handler segment latencies (both handlers)
- `fhRecvTimeUtcNs` - wall-clock timestamp for correlation

A decision is required on **how these raw measurements are aggregated, stored, and consumed**, with support for multiple handler types.

**Note:** TEL focuses on **feed handler latency only**. E2E pipeline latency has been removed as it's not critical for the current use case.

## Notation

| Acronym | Definition |
|---------|------------|
| CTP | Chained Tickerplant (batched publisher) |
| FH | Feed Handler |
| HDB | Historical Database |
| IPC | Inter-Process Communication |
| MLE | Machine Learning Engine |
| RDB | Real-time Database (query-only) |
| RTE | Real-Time Engine |
| SLO | Service-Level Objective |
| TEL | Telemetry Process |
| TP | Tickerplant |
| WDB | Write-only Database (intraday writedown to HDB) |

## Decision

Telemetry will be collected from **per-event measurements** received via subscription to Chained TP and **aggregated in a dedicated TEL process**.

### Telemetry Categories

| Category | Metrics | Source |
|----------|---------|--------|
| FH segment latencies | `fhParseUs`, `fhSendUs` per handler | Per-event fields from market data |
| FH health | `uptimeSec`, `msgsReceived`, `connState` per handler | Published by FH → TP → CTP → TEL |

### Collection Architecture

```
Trade FH ───► TP:5010 ───► CTP:5014 ───┬───► RTE:5015
Quote FH ──────────────^               ├───► TEL:5016
                                       └───► RDB:5017
```

TEL subscribes to Chained TP for:
- `trade_binance` (extracts `fhParseUs`, `fhSendUs`)
- `quote_binance` (extracts `fhParseUs`, `fhSendUs`)
- `health_feed_handler` (FH health metrics)

### Schema Validation

TEL validates incoming message schemas on first receipt to detect schema drift:

**Named field indices (not hardcoded):**
```q
/ Trade schema indices
.tel.idx.trade.sym:1;
.tel.idx.trade.fhParseUs:9;
.tel.idx.trade.fhSendUs:10;
.tel.idx.trade.expectedFields:13;

/ Quote schema indices
.tel.idx.quote.sym:1;
.tel.idx.quote.fhParseUs:25;
.tel.idx.quote.fhSendUs:26;
.tel.idx.quote.expectedFields:29;
```

**Validation on first message:**
- Checks field count matches expected
- Logs warning if mismatch detected
- Tracks validation state per table

**Query interface:**
```q
.tel.schemaStatus[]
/ Returns:
/ table         validated valid expectedFields
/ --------------------------------------------
/ trade_binance 1         1     13
/ quote_binance 1         1     29
```

### Aggregation Strategy

**Time buckets:**

| Parameter | Value | Rationale |
|-----------|-------|-----------|
| Bucket size | 5 seconds | Balances granularity with overhead |
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

**Feed handler health (`health_feed_handler`):**

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

### Retention Policy

**Telemetry is ephemeral**, consistent with ADR-003 system-wide stance:

| Aspect | Policy |
|--------|--------|
| Persistence | In-memory only |
| On restart | Telemetry tables start empty |
| Telemetry bucket retention | Keep last 15 minutes |
| Health record retention | Keep last 1 minute per handler |

### Connection Resilience

TEL uses resilient connection handling to CTP (see ADR-008):
- Exponential backoff on connection failure
- Timer-based reconnection attempts
- `.z.pc` handler detects disconnection
- Process continues in degraded mode if CTP unavailable

### Query Interface

**FH Latency:**
```q
.tel.getFhLatency[]     / Latest FH latency stats per handler/sym
.tel.vsFhLatency[]      / Dashboard-formatted latency grid
```

**FH Health:**
```q
.tel.fhStatus[]         / Raw health stats per handler
.tel.vsFhStatus[]       / Dashboard-formatted status grid
```

**Schema:**
```q
.tel.schemaStatus[]     / Schema validation status
```

**Health:**
```q
.health[]               / Standardized health check
```

**Raw Tables:**
```q
telemetry_latency_fh    / FH latency aggregates
health_feed_handler     / FH health records
```

### Throughput Derivation

Throughput (events per second) is derived from the `cnt` field:

```q
/ Events per second = cnt / bucket_size
/ Example: 250 events in 5-second bucket = 50 events/sec
eventsPerSec: cnt % 5
```

No separate throughput table is needed.

## Rationale

This approach was selected because it:

- **Focuses on FH latency**: The most reliable metrics (monotonic-derived)
- **Subscribes to Chained TP**: Batched data is sufficient for telemetry
- **Validates schemas**: Catches schema drift early via named indices
- **Removes E2E latency**: Cross-process wall-clock latency is less reliable
- **Removes system memory queries**: Simplifies TEL, reduces IPC
- **Single source for monitoring**: TEL handles all telemetry
- **Ephemeral stance**: Matches project's exploratory nature

## Alternatives Considered

### 1. Query WDB/RTE for latency data (previous design)
Changed (2026-01-17):
- Adds IPC overhead
- Subscription-based is simpler
- TEL has all data it needs from market data events

### 2. E2E latency tracking (previous design)
Removed (2026-01-17):
- Wall-clock derived, less reliable
- FH latency (monotonic) is sufficient
- Simplifies implementation

### 3. System memory tracking (previous design)
Removed (2026-01-17):
- Requires IPC to WDB/RTE
- Not critical for current use case
- Can be added later if needed

### 4. Direct TP subscription
Rejected:
- Adds load to primary TP
- Batched data via CTP is sufficient
- TEL doesn't need tick-by-tick

### 5. Hardcoded field indices
Rejected (2026-01-18):
- Brittle, breaks silently on schema change
- Named indices with validation are safer
- Minimal overhead for significant reliability gain

## Consequences

### Positive

- Simple, focused telemetry (FH latency only)
- Subscription-based (no IPC queries)
- Schema validation catches drift early
- Named indices document field positions
- Matches Chained TP batch interval
- Low overhead
- Reliable metrics (monotonic-derived)
- Dashboard-ready query functions
- Resilient connection handling

### Negative / Trade-offs

- No E2E pipeline latency
- No system memory tracking
- 1-second delay from CTP batching (plus 5s aggregation)
- Telemetry lost on restart

These trade-offs are acceptable for the current phase.

## Links / References

- `reference/kdbx-real-time-architecture-reference.md`
- `adr-001-timestamps-and-latency-measurement.md` (raw fields, SLO targets)
- `adr-003-tickerplant-logging-and-durability-strategy.md` (ephemeral stance)
- `adr-004-real-time-rolling-analytics-computation.md` (RTE details)
- `adr-007-visualisation-and-consumption-strategy.md` (dashboard consumption)
- `adr-008-error-handling-strategy.md` (connection resilience)
