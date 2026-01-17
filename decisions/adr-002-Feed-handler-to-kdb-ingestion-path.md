# ADR-002: Feed Handler to kdb Ingestion Path

## Status
Accepted (Updated 2026-01-17)

## Date
Original: 2025-12-17
Updated: 2026-01-03, 2026-01-06, 2026-01-16, 2026-01-17

## Context

This project ingests real-time Binance market data via WebSocket in C++ feed handlers and publishes it into a kdb+/KDB-X environment for real-time analytics.

Two data types are ingested:
- **Trade data** — Individual trade executions from the trade stream
- **Quote data** — L2 order book depth (5 levels of bid/ask) derived from depth stream

There are multiple technically valid ingestion paths from an external feed handler into kdb, including:

- Direct publication into a Tickerplant via IPC
- Writing to intermediate logs or files for later ingestion
- Embedding q/kdb within the feed handler process
- Publishing directly into a WDB
- Using alternative transports (message queues, streaming platforms, etc.)

Each option represents different trade-offs in terms of latency, complexity, observability, resilience, and alignment with established kdb architectures.

This project is explicitly inspired by the *Building Real Time Event Driven KDB-X Systems* reference architecture, which assumes a tickerplant-centric design.

## Notation

| Acronym | Definition |
|---------|------------|
| CTP | Chained Tickerplant (batched publisher) |
| FH | Feed Handler |
| HDB | Historical Database |
| IPC | Inter-Process Communication |
| L2 |  Level 2 market depth (5 price levels per side) |
| MLE | Machine Learning Engine |
| RDB | Real-time Database (query-only, receives batched data) |
| RTE | Real-Time Engine |
| SIG | Signal Generator (future) |
| TEL | Telemetry |
| TP | Tickerplant |
| WDB | Write-only Database (intraday writedown to HDB) |

## Decision

Both feed handlers publish data **directly into a single Tickerplant via IPC**.

### Architecture

```
Binance Trade Stream ──► Trade FH ──┬──► TP:5010 ──┬──► WDB:5011 (writes to HDB)
                                    │              ├──► MLE:5012 (tick-by-tick)
Binance Depth Stream ──► Quote FH ──┘              │
                                                   └──► CTP:5014 (batched 1s)
                                                             │
                                                             ├──► RTE:5015 (dashboard)
                                                             ├──► TEL:5016 (FH latency)
                                                             └──► RDB:5017 (user queries)
```

### Feed Handler Design

| Handler | Data Type | State | Source |
|---------|-----------|-------|--------|
| Trade FH | `trade_binance` | Stateless | WebSocket trade stream |
| Quote FH | `quote_binance` | Stateful (L2 order book) | WebSocket depth + REST snapshot |

**Separate processes**: Each feed handler runs as an independent process. This follows the "single responsibility" principle and provides:
- Independent failure domains
- Simpler code (no threading)
- Independent restart capability

### Trade Feed Handler

The trade feed handler is **stateless**:
- Receives trade events from Binance WebSocket
- Parses JSON, normalises fields
- **Captures timing measurements** (see ADR-001):
  - `fhParseUs`: Parse latency (JSON → normalized fields)
  - `fhSendUs`: Send latency (build IPC message → send)
- Publishes immediately to tickerplant
- No internal state maintained

**Timing methodology**:
- Uses `std::chrono::steady_clock` for monotonic duration measurement
- Parse timer: starts at message receive, ends after field extraction
- Send timer: starts after parse, ends after IPC send initiation
- Both timings included in published message (see ADR-001)

### Quote Feed Handler

The quote feed handler is **stateful** and maintains **L2 order book state**:
- Uses **OrderBookManager** with flat-array architecture for cache efficiency
- Maintains 5 levels of bid/ask depth per symbol
- Fetches REST snapshot for initial sync
- Applies WebSocket depth deltas incrementally
- Validates sequence numbers for gap detection
- **Captures timing measurements** (see ADR-001):
  - `fhParseUs`: Parse + order book update + L2 extraction latency
  - `fhSendUs`: L2 snapshot construction + IPC send latency
- Publishes L2 quotes (22 price/qty fields) on each update
- Tracks validity state per symbol (INIT → SYNCING → VALID ↔ INVALID)

**Timing methodology**:
- Parse timer includes: JSON parse + book update + L2 extraction
- Send timer includes: L2 snapshot build + IPC serialization
- Significantly higher parse times than trade handler due to stateful processing
- Both timings included in published message (see ADR-001)

**OrderBookManager Architecture**:
- Contiguous memory layout (~16KB for 100 symbols)
- O(1) symbol lookup via index mapping
- Per-symbol state machine
- Optimised for high-frequency updates

### Ingestion Path

- Each C++ feed handler connects to the running kdb tickerplant using the kdb+ C API
- Normalised events are published as update messages to the tickerplant
- The tickerplant remains the single ingress point for:
  - Logging (single combined log file, see ADR-003)
  - Fan-out to subscribers (WDB, MLE, CTP)
  - Time-ordering and sequencing
  - Adding `tpRecvTimeUtcNs` timestamp
  - Gap detection for trade sequence numbers

### IPC Mode

Both feed handlers use **asynchronous IPC** to publish updates to the tickerplant.

In kdb+ terms, this is equivalent to using `neg[h](...)` rather than `h(...)`.

Rationale:
- Minimises feed handler blocking
- Reduces end-to-end latency
- Aligns with standard kdb real-time patterns

Trade-off: Asynchronous publishing means the feed handler cannot confirm successful receipt. This is acceptable because:
- TP logging provides durability (ADR-003)
- Upstream replay (Binance reconnect) provides recovery for gaps

### Publishing Mode

Both feed handlers publish **tick-by-tick** (one IPC call per event).

| Handler | Trigger |
|---------|---------|
| Trade FH | Every trade received |
| Quote FH | Every depth delta applied (change-based publishing) |

Rationale:
- Simplifies latency measurement (no batching delays)
- Provides clearest view of per-event latency characteristics
- Appropriate for initial exploratory phase
- Matches the "hot path" pattern from the reference architecture

Trade-off: Tick-by-tick publishing has higher IPC overhead than batching. This is acceptable because:
- Latency measurement clarity is prioritised over throughput
- Current message rates are manageable without batching
- Batching can be introduced later if throughput becomes a concern

### Message Format

Each normalised event is published as a kdb+ IPC message using native serialisation.

On the tickerplant side:
- Messages are received via `.z.ps` (async) handler
- The standard `.u.upd` function is invoked with table name and row data
- TP adds `tpRecvTimeUtcNs` timestamp using `.z.p`

Examples (conceptual kdb+ equivalent):
```q
/ Trade publication
neg[h] (`.u.upd; `trade_binance; tradeData)

/ Quote publication
neg[h] (`.u.upd; `quote_binance; quoteData)
```

### Tables Published

| Table | Source | FH Fields | TP Adds | WDB Adds | Total | Subscribers |
|-------|--------|-----------|---------|----------|-------|-------------|
| `trade_binance` | Trade FH | 12 | 1 | 1 | 14 | WDB, MLE, CTP |
| `quote_binance` | Quote FH | 28 | 1 | 1 | 30 | WDB, CTP |

**Trade Schema (14 fields total)**:
- **FH sends (12)**: `time`, `sym`, `tradeId`, `price`, `qty`, `buyerIsMaker`, `exchEventTimeMs`, `exchTradeTimeMs`, `fhRecvTimeUtcNs`, `fhParseUs`, `fhSendUs`, `fhSeqNo`
- **TP adds (1)**: `tpRecvTimeUtcNs`
- **WDB adds (1)**: `wdbRecvTimeUtcNs`

**Quote L2 Schema (30 fields total)**:
- **FH sends (28)**: 
  - `time`, `sym`
  - `bidPrice1-5`, `bidQty1-5` (10 fields)
  - `askPrice1-5`, `askQty1-5` (10 fields)
  - `isValid`, `exchEventTimeMs`, `fhRecvTimeUtcNs`, `fhParseUs`, `fhSendUs`, `fhSeqNo`
- **TP adds (1)**: `tpRecvTimeUtcNs`
- **WDB adds (1)**: `wdbRecvTimeUtcNs`

**Key differences from original ADR**:
- Added `fhParseUs` and `fhSendUs` to both schemas (2026-01-06)
- Updated field counts accordingly
- Quote handler now tracks timing like trade handler

### Connection Failure Handling

If the connection to the tickerplant is lost:
- The feed handler logs an error
- Incoming messages are dropped (not buffered)
- The feed handler attempts to reconnect with exponential backoff
- Recovery relies on Binance WebSocket reconnection semantics

For the quote handler specifically:
- Order book state is preserved during TP disconnect
- Publications resume when connection restored
- Book validity is maintained independently of TP connection

This fail-fast approach is acceptable given the project's exploratory nature (see ADR-006).

## Rationale

This option was selected because it:

- Aligns with canonical kdb real-time architectures
- Preserves a clear separation of concerns:
  - Feed handlers: external I/O, parsing, normalisation, timestamp capture (ADR-001)
  - Tickerplant: sequencing, publication, logging (ADR-003)
  - WDB: intraday storage and HDB persistence
  - CTP: batched publishing for dashboard/query subscribers
  - RTE: derived analytics (ADR-004)
  - MLE: information-driven bars (trades only)
- Minimises end-to-end latency by avoiding intermediate persistence layers
- Keeps operational complexity low for an exploratory but realistic system
- Allows later evolution (e.g., replication, recovery) without changing the feed handler contract
- Enables detailed latency measurement through timing fields

### Separate Processes Rationale

Running trade and quote handlers as separate processes was selected because:
- Simpler than multi-threaded single process
- Independent failure domains (quote crash doesn't affect trades)
- Different complexity profiles (stateless vs stateful)
- Easier debugging and development
- Matches "single responsibility" principle from reference architecture
- Allows independent timing analysis (trade vs quote latency comparison)

### L2 vs L1 Rationale

L2 with 5 levels depth was selected over L1 (best bid/ask only) because:
- Enables order book imbalance calculations across multiple levels
- Provides richer market microstructure data
- Flat-array architecture keeps memory overhead minimal
- Wide schema (fixed 5 levels) is more efficient than nested lists for analytics

### Timing Measurement Rationale

Including `fhParseUs` and `fhSendUs` in every message was selected because:
- Enables per-event latency analysis (see ADR-005)
- No separate telemetry channel needed
- Minimal overhead (2 long fields)
- Simplifies implementation (single message path)
- Raw data available in WDB for debugging
- Aligns with exploratory project goals

## Alternatives Considered

### 1. Writing to Files or Logs
Rejected because:
- Adds latency and operational overhead
- Duplicates functionality already handled by the tickerplant
- Moves the system away from event-driven design

### 2. Embedding q inside the Feed Handler
Rejected because:
- Couples C++ lifecycle and kdb runtime tightly
- Complicates deployment and debugging
- Reduces architectural clarity

### 3. Publishing Directly to a WDB
Rejected because:
- Bypasses the tickerplant's role in sequencing and fan-out
- Makes scaling and recovery more difficult
- Deviates from standard kdb design patterns

### 4. Using External Messaging Systems
Out of scope for this project:
- Introduces unnecessary infrastructure
- Distracts from core kdb architecture exploration

### 5. Synchronous IPC
Rejected because:
- Blocks the feed handler until the tickerplant acknowledges
- Increases end-to-end latency
- Not necessary given TP logging (ADR-003)

### 6. Batched Publishing
Rejected because:
- Introduces batching delay into latency measurements
- Complicates per-event latency analysis
- Not required at current message rates

May be reconsidered if throughput requirements increase.

### 7. Single Multi-Threaded Feed Handler
Rejected because:
- Adds threading complexity
- Single failure domain for both data types
- Harder to debug and reason about
- No compelling benefit at current scale
- Cannot independently restart trade vs quote handlers

### 8. L1 Only (Best Bid/Ask)
Rejected because:
- Limits analytics to top-of-book only
- Cannot calculate multi-level imbalance
- L2 overhead is minimal with flat-array design

### 9. Separate Telemetry Channel for Timing
Rejected because:
- Adds complexity (two publication paths)
- Harder to correlate timing with specific events
- Per-event timing fields are simple and effective
- Minimal overhead (2 fields)

## Consequences

### Positive

- Clean, canonical ingestion pipeline
- Clear ownership boundaries between components
- Easy to reason about latency and ordering
- Trade feed handler remains simple and stateless
- Quote feed handler encapsulates L2 book complexity efficiently
- Tick-by-tick publishing provides clear latency visibility
- Asynchronous IPC minimises feed handler blocking
- Independent processes allow independent restart and timing analysis
- Single TP simplifies subscriber management
- L2 depth enables richer analytics (imbalance across levels)
- Per-event timing enables detailed latency analysis
- Raw timing data available in WDB for debugging
- Both handlers instrumented consistently
- TP validates trade `fhSeqNo` for gap detection (quote gaps deferred)

### Negative / Trade-offs

- Requires a running tickerplant for ingestion
- Two processes to manage instead of one
- Quote handler maintains state (more complex than trade handler)
- Asynchronous publishing means no delivery confirmation
- Tick-by-tick has higher IPC overhead than batching
- Messages are lost if connection drops (no local buffering)
- L2 schema is wider than L1 (30 vs 14 fields)
- Per-event timing fields increase storage volume (acceptable for exploratory project)
- Quote handler timing includes stateful processing (higher than trade handler)

These trade-offs are acceptable and consistent with the project's goals.

## Performance Observations

Based on production telemetry (see ADR-005):

**Trade Feed Handler:**
- Parse latency (p95): ~20-50μs
- Send latency (p95): ~5-10μs
- Simple JSON parse + field extraction

**Quote Feed Handler:**
- Parse latency (p95): ~100-300μs (includes book update)
- Send latency (p95): ~5-10μs (similar to trade)
- Stateful processing accounts for higher parse time

These measurements validate the architectural decision to instrument both handlers and confirm the performance characteristics expected from the design.

## Links / References

- `reference/kdbx-real-time-architecture-reference.md`
- `notes/kdbx-real-time-architecture-measurement-notes.md`
- `adr-001-timestamps-and-latency-measurement.md` (timestamp capture and timing measurement in FH)
- `adr-003-tickerplant-logging-and-durability-strategy.md` (logging and durability)
- `adr-004-real-time-analytics-computation.md` (RTE analytics)
- `adr-005-telemetry-and-metrics-aggregation-strategy.md` (timing aggregation in TEL)
- `adr-006-recovery-and-replay-strategy.md` (recovery and replay)
- `adr-009-Order-Book-Architecture.md` (quote handler order book details)
- `specs/trades-schema.md` (trade table schema)
- `specs/quotes-schema.md` (quote table schema)
