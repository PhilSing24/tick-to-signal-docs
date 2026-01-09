# Latency Measurement Notes

## Purpose

This document provides practical guidance on how latency measurement is defined and implemented for this project.

It serves as:
- A companion to the ADRs (particularly ADR-001, ADR-005)
- A reference for implementation of timestamp capture and latency calculation
- Documentation of the trust model for cross-process measurements

This document is **project-specific** and reflects decisions made in the ADRs.

*Last updated: 2026-01-07*

## Notation

| Acronym | Definition |
|---------|------------|
| E2E | End-to-End |
| FH | Feed Handler |
| IPC | Inter-Process Communication |
| NTP | Network Time Protocol |
| PTP | Precision Time Protocol (IEEE 1588) |
| RDB | Real-Time Database |
| RTE | Real-Time Engine |
| SLO | Service-Level Objective |
| TEL | Telemetry Process |
| TP | Tickerplant |
| UTC | Coordinated Universal Time |

## Measurement Points

### Overview

The following measurement points are captured across the pipeline:
```
Binance ──► Trade FH ──┬──► TP ──┬──► RDB
                       │         ├──► RTE
Binance ──► Quote FH ──┘         └──► TEL (aggregation)
```

**Current status:** Full pipeline instrumented for both trade and quote handlers.

### Upstream (Binance)

| Point | Field Name | Source | Unit | Description |
|-------|------------|--------|------|-------------|
| Exchange event time | `exchEventTimeMs` | Binance `E` field | ms since epoch | Time Binance generated the event |
| Exchange trade time | `exchTradeTimeMs` | Binance `T` field | ms since epoch | Time the trade occurred (trades only) |

Notes:
- For trade events, `E` and `T` may be equal
- Quote events only have `E` (no trade time)
- Both are server-side timestamps from Binance
- Clock skew between Binance and local system is expected

### Feed Handlers

Both trade and quote feed handlers capture identical instrumentation fields:

| Point | Field Name | Clock Type | Unit | Description |
|-------|------------|------------|------|-------------|
| FH receive time | `fhRecvTimeUtcNs` | Wall-clock | ns since epoch | When WebSocket message is available to process |
| FH parse duration | `fhParseUs` | Monotonic | microseconds | Duration from receive to parse/normalise complete |
| FH send duration | `fhSendUs` | Monotonic | microseconds | Duration from parse complete to IPC send initiation |
| FH sequence number | `fhSeqNo` | N/A | integer | Monotonically increasing per FH instance |

**Handler-specific timing characteristics:**

| Handler | `fhParseUs` includes | Expected p95 |
|---------|---------------------|--------------|
| Trade FH | JSON parse + field extraction | 20-50µs |
| Quote FH | JSON parse + book update + L5 extraction | 100-300µs |

| Handler | `fhSendUs` includes | Expected p95 |
|---------|---------------------|--------------|
| Trade FH | IPC message build + serialization | 5-10µs |
| Quote FH | L5 snapshot build + IPC serialization | 5-10µs |

Notes:
- `fhRecvTimeUtcNs` is wall-clock for cross-process correlation
- `fhParseUs` and `fhSendUs` are monotonic-derived durations (always trusted)
- `fhSeqNo` supports gap detection (see ADR-001, ADR-006)
- Quote handler parse time is higher due to stateful order book management

### Tickerplant

| Point | Field Name | Clock Type | Unit | Description |
|-------|------------|------------|------|-------------|
| TP receive time | `tpRecvTimeUtcNs` | Wall-clock | ns since epoch | When TP receives message (`.z.p` converted) |

Notes:
- Captured in `.u.upd` handler
- Converted from `.z.p` to nanoseconds since Unix epoch
- Used for FH-to-TP latency calculation
- TP logs to separate files per data type (ADR-003)

### Real-Time Database

| Point | Field Name | Clock Type | Unit | Description |
|-------|------------|------------|------|-------------|
| RDB apply time | `rdbApplyTimeUtcNs` | Wall-clock | ns since epoch | When update becomes query-consistent |

Notes:
- Captured at start of `.u.upd` handler before insert
- Added during both live updates AND log replay
- Represents end of core pipeline (data queryable)
- Used for TP-to-RDB and E2E latency calculation

### Real-Time Engine

RTE does not add timestamps to market data. It computes analytics (VWAP, imbalance) from incoming data.

| Metric | Description | Implementation |
|--------|-------------|----------------|
| VWAP | Volume-weighted average price | Time-bucketed aggregation (1s buckets) |
| Imbalance | Order book imbalance | Latest L5 snapshot per symbol |

Notes:
- RTE subscribes directly to TP (peer to RDB)
- No per-event timing added (analytics are derived)
- Auto-replays from logs on startup (ADR-006)

### Telemetry Process

TEL aggregates latency metrics from RDB queries:

| Table | Contents | Source |
|-------|----------|--------|
| `telemetry_latency_fh` | FH segment percentiles (unified trade + quote) | RDB query |
| `telemetry_latency_e2e` | Cross-process percentiles (trades only) | RDB query |
| `telemetry_system` | Memory usage per process | RDB, RTE, TEL queries |
| `health_feed_handler` | FH health metrics | TP subscription |

Notes:
- TEL uses persistent IPC handles to RDB/RTE (ADR-005)
- 5-second aggregation buckets
- 15-minute retention

## Field Mapping Summary

### Trade Table (`trade_binance` - 14 fields)

| Measurement Point | Field Name | Status |
|-------------------|------------|--------|
| Exchange event time | `exchEventTimeMs` | ✓ Implemented |
| Exchange trade time | `exchTradeTimeMs` | ✓ Implemented |
| FH receive | `fhRecvTimeUtcNs` | ✓ Implemented |
| FH parse duration | `fhParseUs` | ✓ Implemented |
| FH send duration | `fhSendUs` | ✓ Implemented |
| FH sequence | `fhSeqNo` | ✓ Implemented |
| TP receive | `tpRecvTimeUtcNs` | ✓ Implemented |
| RDB apply | `rdbApplyTimeUtcNs` | ✓ Implemented |

### Quote Table (`quote_binance` - 30 fields)

| Measurement Point | Field Name | Status |
|-------------------|------------|--------|
| Exchange event time | `exchEventTimeMs` | ✓ Implemented |
| FH receive | `fhRecvTimeUtcNs` | ✓ Implemented |
| FH parse duration | `fhParseUs` | ✓ Implemented |
| FH send duration | `fhSendUs` | ✓ Implemented |
| FH sequence | `fhSeqNo` | ✓ Implemented |
| TP receive | `tpRecvTimeUtcNs` | ✓ Implemented |
| RDB apply | `rdbApplyTimeUtcNs` | ✓ Implemented |

Note: Quote table also includes 22 L5 price/qty fields and `isValid` flag.

## Segment Latency Definitions

### Feed Handler Segments (Monotonic — Always Trusted)

| Segment | Definition | Formula |
|---------|------------|---------|
| `fh_parse_us` | Parse/normalise duration | `fhParseUs` (captured directly) |
| `fh_send_us` | Post-parse to IPC initiation | `fhSendUs` (captured directly) |
| `fh_total_us` | Total FH processing | `fhParseUs + fhSendUs` |

These are derived from monotonic clock and are **always trusted** regardless of clock sync.

### Cross-Process Segments (Wall-Clock — Trust Depends on Sync)

All timestamps use nanoseconds since epoch. Unit conversions are explicit.

| Segment | Definition | Formula | Status |
|---------|------------|---------|--------|
| `market_to_fh_ms` | Binance to FH receive | `(fhRecvTimeUtcNs / 1e6) - exchEventTimeMs` | ✓ Measurable (indicative) |
| `fh_to_tp_ms` | FH send to TP receive | `(tpRecvTimeUtcNs - fhRecvTimeUtcNs) / 1e6` | ✓ Measurable |
| `tp_to_rdb_ms` | TP receive to RDB apply | `(rdbApplyTimeUtcNs - tpRecvTimeUtcNs) / 1e6` | ✓ Measurable |
| `e2e_to_rdb_ms` | FH receive to RDB apply | `(rdbApplyTimeUtcNs - fhRecvTimeUtcNs) / 1e6` | ✓ Measurable |

Note: E2E latency is computed for trades only in TEL (quote E2E is less meaningful due to stateful processing).

### Unit Conversions

| From | To | Conversion |
|------|-----|------------|
| Nanoseconds | Milliseconds | Divide by 1,000,000 (1e6) |
| Microseconds | Milliseconds | Divide by 1,000 (1e3) |
| kdb timestamp | Nanoseconds since epoch | `946684800000000000 + "j"$ts - 2000.01.01D0` |

## Clock Types

### Monotonic Time (Duration Clock)

**Purpose:** Measure durations within a single process.

**Characteristics:**
- Never moves backwards
- Not affected by NTP/PTP adjustments
- Cannot be compared across processes
- Suitable for segment timing within FH

**Implementation:**
- C++: `std::chrono::steady_clock`
- kdb: Use timestamp differences within same process

**Trust:** Always trusted for duration measurements.

### Wall-Clock Time (UTC/Calendar)

**Purpose:** Correlate events across processes/hosts.

**Characteristics:**
- Can be adjusted by NTP/PTP
- Can move backwards (rare, but possible)
- Comparable across processes when clock sync is acceptable
- Subject to clock skew between hosts

**Implementation:**
- C++: `std::chrono::system_clock` (recorded as UTC nanoseconds since epoch)
- kdb: `.z.p` converted to nanoseconds since epoch

**Trust:** Depends on clock synchronisation quality.

## Clock Synchronisation Trust Model

### Trust Rules

| Deployment | Clock Sync | Monotonic Latencies | Cross-Process Latencies |
|------------|------------|---------------------|------------------------|
| Single host (development) | N/A | ✓ Trusted | ✓ Acceptable (indicative) |
| Multi-host, NTP | NTP | ✓ Trusted | ✓ Trusted for millisecond-class |
| Multi-host, PTP | PTP (IEEE 1588) | ✓ Trusted | ✓ Trusted for sub-millisecond |
| Multi-host, no sync | None | ✓ Trusted | ✗ Unreliable |

### Trust Decision Matrix

| Segment | Clock Type | Single Host | Multi-Host NTP | Multi-Host PTP |
|---------|------------|-------------|----------------|----------------|
| `fh_parse_us` | Monotonic | ✓ | ✓ | ✓ |
| `fh_send_us` | Monotonic | ✓ | ✓ | ✓ |
| `market_to_fh_ms` | Wall-clock | Indicative | Indicative | Indicative |
| `fh_to_tp_ms` | Wall-clock | ✓ | ✓ | ✓ |
| `tp_to_rdb_ms` | Wall-clock | ✓ | ✓ | ✓ |
| `e2e_to_rdb_ms` | Wall-clock | ✓ | ✓ | ✓ |

Notes:
- `market_to_fh_ms` is always "indicative" because Binance clock is outside our control
- Negative values for `market_to_fh_ms` indicate clock misalignment (expected)

### Clock Quality Monitoring

When running multi-host deployments:

| Metric | Source | Threshold |
|--------|--------|-----------|
| NTP offset | `chronyc tracking` or `ntpq -p` | < 5ms for millisecond trust |
| NTP sync state | `chronyc tracking` | Must be synchronised |
| PTP offset | `ptp4l` logs | < 100µs for sub-millisecond trust |

If clock offset exceeds threshold:
- Cross-host E2E latency measurements should be flagged as unreliable
- Rely on per-segment monotonic metrics instead
- Log warning for operational visibility

## SLO Expression

### Target Percentiles

| Percentile | Purpose | Description |
|------------|---------|-------------|
| p50 | Typical latency | Median; what most requests experience |
| p95 | Majority-case tail | 95th percentile; captures most tail latency |
| max | Spike detection | Maximum observed; identifies outliers |

Note: p99 was removed because 5-second buckets don't have enough samples for meaningful p99 (see ADR-005).

### Rolling Windows

| Window | Purpose | Use Case |
|--------|---------|----------|
| 5-second | TEL bucket size | Real-time aggregation |
| 1-minute | Incident detection | Real-time alerting, anomaly detection |
| 15-minute | Trend analysis | Stable trend, capacity planning |

### SLO Scope

SLOs should explicitly state:

| Dimension | Current Scope |
|-----------|---------------|
| Message types | Trades and quotes |
| Handler types | `trade_fh` and `quote_fh` |
| Symbol universe | BTCUSDT, ETHUSDT, SOLUSDT |
| Recovery bursts | Excluded from live SLOs |

### Example SLO Statements
```
Trade FH parse latency p95 < 50µs over 1-minute rolling window
Quote FH parse latency p95 < 300µs over 1-minute rolling window
FH send latency p95 < 10µs over 1-minute rolling window (both handlers)
FH to TP latency p95 < 5ms over 1-minute rolling window (single host)
TP to RDB latency p95 < 1ms over 1-minute rolling window (single host)
E2E to RDB p95 < 10ms over 1-minute rolling window (single host, trades)
```

## Telemetry Aggregation

### Overview

TEL computes telemetry aggregations every 5 seconds via timer, querying RDB for raw latency data. Aggregations are stored in dedicated tables.

### Telemetry Tables

| Table | Contents | Handler |
|-------|----------|---------|
| `telemetry_latency_fh` | FH segment latencies (parseUs, sendUs) — p50/p95/max per bucket | Both (unified with `handler` column) |
| `telemetry_latency_e2e` | Cross-process latencies (fhToTp, tpToRdb, e2e) — p50/p95/max per bucket | Trades only |
| `telemetry_system` | Memory usage (heapMB, usedMB) | RDB, RTE, TEL |
| `health_feed_handler` | FH health metrics (uptime, counts, state) | Both |

### Aggregation Parameters

| Parameter | Value |
|-----------|-------|
| Bucket size | 5 seconds |
| Percentiles computed | p50, p95, max |
| Aggregation location | TEL (port 5013) |
| Retention | 15 minutes (ephemeral) |
| Timer interval | 5000ms |
| IPC handles | Persistent (no reconnect per tick) |

### Example Queries (via TEL port 5013)

```q
/ Latest FH latency stats (both handlers)
select from telemetry_latency_fh where bucket = max bucket

/ Compare trade vs quote handler parse latency
select handler, sym, avg parseUs_p95 as avgParse_p95
  from telemetry_latency_fh
  where bucket > .z.p - 00:05:00
  by handler, sym

/ E2E latency trend (last 5 minutes, trades only)
select from telemetry_latency_e2e where bucket > .z.p - 0D00:05:00

/ 1-minute rolling p95 for trade FH parse latency
select p95_1min:avg parseUs_p95 by sym 
  from telemetry_latency_fh 
  where handler=`trade_fh, bucket > .z.p - 0D00:01:00

/ Feed handler health
.tel.fhStatusTable[]

/ Connection status (persistent handles)
.tel.handleStatus[]
```

## Correlation and Gap Detection

### Correlation Fields

| Field | Purpose | Scope |
|-------|---------|-------|
| `fhSeqNo` | FH-level ordering and gap detection | Per FH instance |
| `tradeId` | Binance trade identification | Per symbol (trades only) |
| `sym` | Symbol grouping | BTCUSDT, ETHUSDT, SOLUSDT |

### Gap Detection

| Gap Type | Detection Method | Recovery |
|----------|------------------|----------|
| FH sequence gap | `fhSeqNo` not contiguous | None (logged, not recovered) |
| Trade ID gap | `tradeId` not contiguous | None; Binance doesn't guarantee contiguous |
| Time gap | Large jump in `exchEventTimeMs` | None |

Gaps are detected and observable via telemetry but not recoverable from upstream (ADR-006).

### Uniqueness Key

**Trades:** `(sym, tradeId)`
**Quotes:** `(sym, time, fhSeqNo)` (no strict uniqueness)

Used for:
- Deduplication (if reconnect/replay occurs)
- Correlation across system components

## Log Replay Considerations

### Replay Timing

During log replay (RDB/RTE startup):
- `rdbApplyTimeUtcNs` is set to current time (replay time), not original time
- This is intentional: represents when data became queryable in current session
- Original `tpRecvTimeUtcNs` is preserved from log

### Replay vs Live Telemetry

| Metric | During Replay | After Replay |
|--------|---------------|--------------|
| `fhParseUs`, `fhSendUs` | From log (original FH timing) | Live |
| `tpRecvTimeUtcNs` | From log (original TP timing) | Live |
| `rdbApplyTimeUtcNs` | Current time (replay time) | Live |
| E2E latency | Not meaningful (mixed times) | Meaningful |

Recommendation: Exclude replay data from E2E SLO calculations.

## Implementation Checklist

### Trade Feed Handler (C++) — ✓ Complete

- [x] Capture `fhRecvTimeUtcNs` using `std::chrono::system_clock`
- [x] Capture monotonic start time using `std::chrono::steady_clock`
- [x] Compute `fhParseUs` after parse/normalise
- [x] Compute `fhSendUs` after row build (before IPC send)
- [x] Increment `fhSeqNo` per published message
- [x] Extract `E`, `T`, `t` from Binance JSON
- [x] Send all fields to TP in correct order
- [x] Publish health metrics every 5 seconds

### Quote Feed Handler (C++) — ✓ Complete

- [x] Capture `fhRecvTimeUtcNs` using `std::chrono::system_clock`
- [x] Capture monotonic start time using `std::chrono::steady_clock`
- [x] Compute `fhParseUs` after parse + book update + L5 extraction
- [x] Compute `fhSendUs` after L5 build (before IPC send)
- [x] Increment `fhSeqNo` per published message
- [x] Extract `E` from Binance depth update JSON
- [x] Maintain order book state (snapshot + delta reconciliation)
- [x] Send all fields to TP in correct order
- [x] Publish health metrics every 5 seconds

### Tickerplant (q) — ✓ Complete

- [x] Implement pub/sub infrastructure via u.q
- [x] Capture `tpRecvTimeUtcNs` in `.u.upd` handler using `.z.p`
- [x] Convert `.z.p` to nanoseconds since epoch
- [x] Add `tpRecvTimeUtcNs` to both trade and quote schemas
- [x] Log to separate files per data type (`-11!` compatible format)
- [x] Handle subscriber disconnect (`.z.pc`)

### RDB (q) — ✓ Complete

- [x] Subscribe to TP for trades and quotes
- [x] Capture `rdbApplyTimeUtcNs` in `.u.upd` handler
- [x] Add `rdbApplyTimeUtcNs` to both schemas
- [x] Store all timestamp fields in `trade_binance` and `quote_binance`
- [x] Auto-replay from logs on startup

### RTE (q) — ✓ Complete

- [x] Subscribe to TP for trades and quotes
- [x] Compute VWAP using time-bucketed aggregation
- [x] Compute order book imbalance from L5 quotes
- [x] Maintain configurable retention (10 minutes)
- [x] Auto-replay from logs on startup

### TEL (q) — ✓ Complete

- [x] Subscribe to TP for health metrics
- [x] Query RDB for latency data (persistent handles)
- [x] Query RTE for memory stats (persistent handles)
- [x] Compute FH latency aggregations (unified trade + quote)
- [x] Compute E2E latency aggregations (trades only)
- [x] Compute system memory metrics
- [x] Implement 15-minute retention cleanup

### LOG (q) — ✓ Complete

- [x] List log files with sizes and chunk counts
- [x] Verify log file integrity
- [x] Cleanup logs older than retention period
- [x] Provide summary by date

## Links / References

- `kdbx-real-time-architecture-reference.md`
- `adr-001-timestamps-and-latency-measurement.md` (authoritative field definitions)
- `adr-002-feed-handler-to-kdb-ingestion-path.md` (FH to TP path)
- `adr-003-tickerplant-logging-and-durability-strategy.md` (TP logging)
- `adr-004-real-time-rolling-analytics-computation.md` (RTE analytics)
- `adr-005-telemetry-and-metrics-aggregation-strategy.md` (TEL aggregation)
- `adr-006-recovery-and-replay-strategy.md` (log replay)
- `adr-007-visualisation-and-consumption-strategy.md` (latency dashboard display)
- `adr-009-l5-order-book-architecture.md` (quote handler timing)
- `adr-010-log-management-and-lifecycle.md` (LOG process)
- `trades-schema.md` (trade table schema)
- `quotes-schema.md` (quote table schema)
