# ADR-001: Timestamps and Latency Measurement

## Status
Accepted (Updated 2026-01-06)

## Date
Original: 2025-12-17
Updated: 2026-01-06

## Context
This project ingests real-time Binance market data via WebSocket (JSON), normalises it in high-performance C++ feed handlers (Boost.Beast + RapidJSON), and publishes it into a kdb+/KDB-X environment (tickerplant/RDB) via IPC (kdb+ C API).

**Feed Handlers:**
- **Trade Feed Handler**: Ingests individual trade executions (simple 1:1 message flow)
- **Quote Feed Handler**: Maintains L5 order book state from depth updates (stateful, complex processing)

We need consistent timestamping to:
- Measure latency within feed handlers reliably (parsing, normalisation, IPC send).
- Estimate end-to-end latency through the kdb pipeline.
- Correlate events across components (FH, TP, RDB, downstream).
- Support debugging during reconnect/recovery scenarios.

Binance events include upstream timestamps:
- `E`: event time (ms since epoch, exchange/server-side)
- `T`: trade time (ms since epoch, exchange-side) - **trades only**

For trade events, `T` and `E` may be equal but are treated as distinct fields. Quote events (depth updates) only have `E`.

## Notation

| Acronym | Definition |
|---------|------------|
| FH | Feed Handler |
| IPC | Inter-Process Communication |
| NTP | Network Time Protocol |
| PTP | Precision Time Protocol (IEEE 1588) |
| RDB | Real-Time Database |
| RTE | Real-Time Engine |
| SLO | Service-Level Objective |
| TP | Tickerplant |
| UTC | Coordinated Universal Time |

## Decision

### 1) Clock Types
We use two clock concepts:

- **Monotonic time** (duration clock): for measuring intra-process durations in the C++ feed handlers.
  - C++: `std::chrono::steady_clock`
  - Rationale: monotonic time never moves backwards and is not affected by NTP/PTP adjustments.

- **Wall-clock time** (UTC/calendar): for cross-process correlation between FH and kdb components.
  - C++: `std::chrono::system_clock` (recorded as UTC)
  - kdb: `.z.p` for process-side wall-clock timestamps
  - Rationale: wall-clock time is comparable across processes/hosts when clock sync quality is acceptable.

### 2) Timestamp Fields Captured

#### Trade Events
For each inbound trade event, the feed handler captures:

**Upstream (from Binance message):**

| Field | Source | Unit |
|-------|--------|------|
| `exchEventTimeMs` | `E` | milliseconds since epoch |
| `exchTradeTimeMs` | `T` | milliseconds since epoch |
| `tradeId` | `t` | integer |

**Feed handler wall-clock (UTC):**

| Field | Description | Unit |
|-------|-------------|------|
| `fhRecvTimeUtcNs` | Wall-clock timestamp when WebSocket message is available to process | nanoseconds since epoch |

**Feed handler monotonic timing (durations):**

| Field | Description | Unit |
|-------|-------------|------|
| `fhParseUs` | Duration from receive to completion of JSON parse + normalisation | microseconds |
| `fhSendUs` | Duration from completion of parse/normalisation to IPC send initiation | microseconds |

**Feed handler sequence number:**

| Field | Description | Unit |
|-------|-------------|------|
| `fhSeqNo` | Monotonically increasing sequence number per feed handler instance | integer |

The sequence number is recommended (not optional) because it:
- Supports gap detection during normal operation
- Aids replay validation if recovery is later introduced (see ADR-006)
- Provides ordering diagnostics independent of exchange IDs
- Has negligible implementation cost

#### Quote Events (L5 Order Book)
For each L5 quote snapshot published, the feed handler captures:

**Upstream (from Binance depth update message):**

| Field | Source | Unit |
|-------|--------|------|
| `exchEventTimeMs` | `E` from depth update | milliseconds since epoch |

**Note:** Quote messages do NOT have `exchTradeTimeMs` (trades only). There is no exchange-provided trade ID equivalent for quotes.

**Feed handler wall-clock (UTC):**

| Field | Description | Unit |
|-------|-------------|------|
| `fhRecvTimeUtcNs` | Wall-clock timestamp when depth update WebSocket message received | nanoseconds since epoch |

**Feed handler monotonic timing (durations):**

| Field | Description | Unit |
|-------|-------------|------|
| `fhParseUs` | Duration for JSON parse + order book state update + L5 extraction | microseconds |
| `fhSendUs` | Duration to build L5 snapshot and send via IPC | microseconds |

**Feed handler sequence number:**

| Field | Description | Unit |
|-------|-------------|------|
| `fhSeqNo` | Monotonically increasing sequence number per feed handler instance | integer |

**Key differences between trade and quote handlers:**
- Quote `fhParseUs` includes stateful order book management (significantly higher than trades)
- Quote `fhSendUs` includes L5 snapshot construction (similar magnitude to trades)
- Quote handler may publish multiple L5 snapshots per depth update due to change-based publishing logic
- Quote handler includes snapshot synchronization overhead (REST API fetch + buffered delta replay)

**kdb-side timestamps (both trades and quotes):**

| Field | Description | Unit |
|-------|-------------|------|
| `tpRecvTimeUtcNs` | TP receive time captured at ingest (`.z.p` converted to nanoseconds since epoch) | nanoseconds since epoch |
| `rdbApplyTimeUtcNs` | Time the update becomes query-consistent in the RDB (nanoseconds since epoch) | nanoseconds since epoch |

**Notes:**
- Monotonic instants are not stored as absolute values; only derived durations are recorded.
- Durations are stored in microseconds for practical precision without excessive storage overhead.

### 3) Schema and Storage

For this exploratory project, all timestamp and latency fields are stored **per-event** in the market data tables. This simplifies implementation and enables direct correlation between specific events and their latency characteristics during debugging.

**Trade table (`trade_binance`) stores:**
- `exchEventTimeMs`
- `exchTradeTimeMs`
- `fhRecvTimeUtcNs`
- `fhParseUs`
- `fhSendUs`
- `fhSeqNo`
- `tradeId`
- Normalised business fields (sym, price, qty, buyerIsMaker, etc.)
- `tpRecvTimeUtcNs`
- `rdbApplyTimeUtcNs`

**Quote table (`quote_binance`) stores:**
- `exchEventTimeMs` (from depth update)
- `fhRecvTimeUtcNs`
- `fhParseUs`
- `fhSendUs`
- `fhSeqNo`
- L5 order book snapshot (bidPrice1-5, bidQty1-5, askPrice1-5, askQty1-5)
- `isValid` (order book synchronization status flag)
- `tpRecvTimeUtcNs`
- `rdbApplyTimeUtcNs`

**Telemetry tables store aggregated metrics (per handler type):**
- Time bucket (e.g., 5-second buckets)
- Handler identifier (`trade_fh` or `quote_fh`)
- Symbol
- Event counts
- Latency percentiles (p50, p95) computed from per-event fields
- Max values for spike detection

Rationale for per-event storage:
- Exploratory project with single user; schema "pollution" is not a concern
- Enables debugging specific events with their exact latency values
- Simplifies implementation (single message path, no separate telemetry routing)
- Aggregation is still performed for dashboard consumption (see ADR-005)

### 4) Latency Metrics Definitions

**Feed handler segment latencies (monotonic-derived, always trusted):**

| Metric | Trade Handler | Quote Handler |
|--------|---------------|---------------|
| `fh_parse_us` | JSON parse + field extraction | JSON parse + order book update + L5 extraction |
| `fh_send_us` | IPC message serialization | L5 snapshot construction + IPC serialization |

**Expected latency ranges (p95, typical conditions):**
- **Trade Handler:** parse ~20-50μs, send ~5-10μs
- **Quote Handler:** parse ~100-300μs, send ~5-10μs

Quote handler parse time is significantly higher due to:
- Stateful order book management (apply bids/asks updates)
- L5 extraction from full order book
- Change detection logic for publish throttling

**Cross-process latencies (wall-clock-derived, trust depends on clock sync):**

All timestamps use nanoseconds since epoch. Unit conversions are explicit in formulas.

| Metric | Definition | Formula | Notes |
|--------|------------|---------|-------|
| `market_to_fh_ms` | Binance to FH receive | `(fhRecvTimeUtcNs / 1e6) - exchEventTimeMs` | Indicative only; includes network + exchange semantics; may be negative if clocks are misaligned |
| `fh_to_tp_ms` | FH send to TP receive | `(tpRecvTimeUtcNs - fhRecvTimeUtcNs) / 1e6` | Same-unit subtraction, then convert to ms |
| `tp_to_rdb_ms` | TP receive to RDB apply | `(rdbApplyTimeUtcNs - tpRecvTimeUtcNs) / 1e6` | |
| `e2e_system_ms` | FH receive to RDB apply | `(rdbApplyTimeUtcNs - fhRecvTimeUtcNs) / 1e6` | Full system latency |

**Handler-specific considerations:**
- Trade handler: 1 message in → 1 event out (simple flow)
- Quote handler: 1 depth update in → 0-N L5 snapshots out (change-based publishing)

### 5) SLO Expression and Aggregation Windows

Latency SLOs are expressed using percentiles over rolling windows.

**Target percentiles:**

| Percentile | Purpose |
|------------|---------|
| p50 | Typical latency (median) |
| p95 | Majority-case tail |

**Rolling windows:**

| Window | Purpose |
|--------|---------|
| 5-second | Real-time telemetry bucketing (see ADR-005) |
| 1-minute | Incident detection, real-time alerting |
| 15-minute | Stable trend analysis, capacity planning |

SLOs should explicitly state:
- Which message types are included (trades, quotes, or both)
- Which handler is measured (trade_fh, quote_fh)
- Which symbol universe is included (BTCUSDT, ETHUSDT, SOLUSDT initially)
- Whether reconnect/recovery bursts are included or excluded
- Whether invalid order book states (quote_fh) are included or excluded

**Example SLO statement:**
> "Trade feed handler (trade_fh) p95 end-to-end latency (FH receive to RDB apply) shall be < 10ms for BTCUSDT, measured over 1-minute windows, excluding reconnection bursts."

### 6) Clock Synchronisation Trust Model

- Segment latencies derived from monotonic time are **always trusted**.
- Cross-host end-to-end latency derived from wall-clock is **only trusted if clock sync quality is acceptable**.

**Trust rules:**

| Deployment | Clock Sync | Cross-host E2E Trust |
|------------|------------|----------------------|
| Single host (development) | N/A | Acceptable; indicative |
| Multi-host, NTP | NTP | Trusted for millisecond-class reporting |
| Multi-host, PTP | PTP | Required for sub-millisecond precision |

**Clock quality monitoring:**
- Clock quality telemetry SHOULD be captured when running multi-host (e.g., NTP/chrony tracking offset).
- If clock offset exceeds a defined threshold (e.g., >5ms), cross-host end-to-end latency measurements should be flagged as unreliable.
- When cross-host latency is unreliable, rely on segment metrics (`fhParseUs`, `fhSendUs`) instead.

### 7) Correlation and Deduplication

#### Trade Events

**Primary uniqueness key:**
- (`sym`, `tradeId`)

**Deduplication behaviour:**
- If reconnect/replay occurs, duplicates are detected by the primary key.
- `fhSeqNo` provides additional ordering context for diagnostics.

**Gap detection:**
- `fhSeqNo` gaps indicate missed events at the feed handler level.
- `tradeId` gaps (per symbol) may indicate missed events at the exchange level, though Binance does not guarantee contiguous trade IDs.

#### Quote Events

**Primary uniqueness key:**
- (`sym`, `time`) - quotes are point-in-time snapshots, not distinct exchange events

**Deduplication behaviour:**
- Quote handler uses change-based publishing; duplicate L5 snapshots are suppressed by the feed handler.
- If reconnect occurs, order book is reset and resynchronized via REST snapshot.

**Gap detection:**
- `fhSeqNo` gaps indicate missed publications at the feed handler level.
- No exchange-provided sequence number for L5 snapshots (depth updates use `U`/`u` updateId ranges internally).
- Order book synchronization state tracked via `isValid` flag.

**Synchronization diagnostics:**
- `isValid=0` indicates order book is not synchronized (INIT/SYNCING/INVALID states).
- `isValid=1` indicates order book is VALID and L5 data is trustworthy.
- Latency metrics for `isValid=0` quotes may be excluded from SLO calculations.

## Consequences

### Positive
- Reliable feed handler latency measurement independent of wall-clock adjustments.
- Clear separation between correlation timestamps (wall-clock) and performance timings (monotonic durations).
- Per-event storage enables precise debugging and correlation.
- Explicit unit naming (`*Ns`, `*Ms`, `*Us`) prevents conversion errors.
- Explicit SLO percentiles and windows provide clear targets for measurement.
- Sequence numbers support future recovery/replay without schema changes.
- Handler-specific characteristics (trade vs quote) are explicitly documented and measurable.
- Telemetry system can compare trade vs quote performance side-by-side.

### Negative / Trade-offs
- End-to-end latency across hosts can be misleading without good clock sync; requires explicit trust rules and monitoring.
- Per-event latency fields increase storage volume (acceptable for exploratory project).
- Additional instrumentation effort in both FH and TP/RDB for both handler types.
- Sequence number adds a small amount of state to each feed handler.
- Quote handler latency is inherently more variable due to stateful processing; requires careful interpretation.
- Quote `isValid` flag adds complexity to SLO calculations (exclusion logic).

## Implementation Notes

### Quote Handler Timing Specifics

**Parse timing (`fhParseUs`):**
- Starts when WebSocket message buffer is available
- Ends when L5 extraction completes
- Includes: JSON parse, order book state update, L5 extraction
- Does NOT include: snapshot fetch (initial sync only), publish decision logic

**Send timing (`fhSendUs`):**
- Starts after L5 extraction completes
- Ends when IPC send is initiated
- Includes: L5 snapshot construction, kdb+ C API serialization

**Snapshot initialization:**
- Initial REST snapshot fetch is NOT timed (one-time operation)
- Buffered delta replay during sync is included in `fhParseUs` for those events

## Links / References
- `../kdbx-real-time-architecture-reference.md`
- `../kdbx-real-time-architecture-measurement-notes.md`
- `../api-binance.md`
- `adr-005-telemetry-and-metrics-aggregation-strategy.md` (for telemetry aggregation details)
- `adr-006-recovery-and-replay-strategy.md` (for recovery deferral context)
- `adr-009-L1-Order-Book-Architecture.md` (for quote handler state machine details)
