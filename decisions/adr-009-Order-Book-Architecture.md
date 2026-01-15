# ADR-009: Order Book Architecture

## Status
Accepted (Updated 2026-01-14)

## Date
Original: 2025-12-29
Updated: 2026-01-03, 2026-01-06, 2026-01-11, 2026-01-14

## Context

The system ingests real-time trade data from Binance. To support trading strategies and market analysis, order book quote data is also required.

Binance provides order book data via two mechanisms:
1. REST API snapshot (`/api/v3/depth`)
2. WebSocket delta stream (`@depth`)

Neither alone is sufficient:
- Snapshots become stale immediately
- Deltas require a baseline to apply against

Binance specifies a reconciliation protocol:
1. Buffer deltas from WebSocket
2. Fetch REST snapshot
3. Apply buffered deltas with sequence validation
4. Continue applying live deltas

A decision is required on how to implement order book ingestion with correct reconciliation.

**Update (2026-01-03):** Extended from Level 1 (best bid/ask only) to Level 2 with 5 levels of depth to enable richer analytics including multi-level order book imbalance.

**Update (2026-01-06):** Added timing measurements (`fhParseUs`, `fhSendUs`) for latency analysis consistent with trade feed handler (see ADR-001).

**Update (2026-01-11):** Corrected logging section to align with ADR-003 (single log file per day).

**Update (2026-01-14):** Standardized terminology to use "Level 2 (5 levels)"

## Notation

| Acronym | Definition |
|---------|------------|
| FH | Feed Handler |
| L1 | Level 1 — best bid/ask only (top of book) |
| L2 | Level 2 — multiple price levels of depth |
| TP | Tickerplant |

**Depth terminology used in this document:**
- "Level 2 (5 levels)" or "L2 depth" refers to order book data with 5 price levels per side
- The system captures the top 5 bid and top 5 ask levels

## Decision

Level 2 quotes (5 levels of depth) are ingested via a dedicated Quote Feed Handler process that implements snapshot + delta reconciliation with a state machine and publishes 5 levels of bid/ask depth with timing measurements.

### Process Architecture

**Separate process (not integrated into Trade FH):**

| Aspect | Trade FH | Quote FH |
|--------|----------|----------|
| Data source | `@trade` stream | `@depth` stream + REST |
| Complexity | Simple (stateless) | Complex (book state, reconciliation) |
| State | None | L2 order book per symbol (5 levels) |
| Parse timing | JSON parse only (~20-50μs p95) | Parse + book update + depth extract (~100-300μs p95) |
| Failure mode | Independent | Independent |
| Restart | No impact on quotes | No impact on trades |

Rationale:
- Single responsibility principle
- Independent failure and restart
- Simpler debugging (separate logs)
- No threading required
- Different performance characteristics measurable independently

### OrderBookManager Architecture

The quote feed handler uses `OrderBookManager` with a flat-array design optimised for multiple symbols:

| Feature | Implementation |
|---------|----------------|
| Memory layout | Contiguous flat arrays (~16KB for 100 symbols) |
| Symbol lookup | O(1) via `symbolToIndex_` map |
| Per-symbol state | State enum + lastUpdateId + 5 bid levels + 5 ask levels |
| Cache efficiency | Sequential memory access during updates |

**Data structure per symbol:**
```cpp
struct SymbolBook {
    State state;           // INIT, SYNCING, VALID, INVALID
    uint64_t lastUpdateId;
    std::array<Level, 5> bids;  // Level = {price, qty}
    std::array<Level, 5> asks;
};
```

**Memory layout:**
- All `SymbolBook` structs in contiguous array
- ~160 bytes per symbol
- 100 symbols ≈ 16KB (fits in L1 cache)

### Timing Measurement

Following ADR-001, the quote feed handler captures timing measurements:

**Parse timing (`fhParseUs`):**
- Starts: WebSocket message buffer available
- Includes: JSON parse + order book state update + 5-level extraction
- Ends: After quote snapshot is extracted
- Measurement: `std::chrono::steady_clock` (monotonic)

**Send timing (`fhSendUs`):**
- Starts: After depth extraction completes
- Includes: Quote snapshot construction + kdb+ IPC serialization
- Ends: After IPC send initiation
- Measurement: `std::chrono::steady_clock` (monotonic)

**Expected latencies (p95):**
- `fhParseUs`: 100-300μs (higher than trade due to stateful processing)
- `fhSendUs`: 5-10μs (similar to trade)

**Comparison to trade handler:**
- Quote parse is 5-10x slower (stateful book updates vs simple parse)
- Quote send is similar (both use kdb+ IPC serialization)

See ADR-001 for full timing measurement specification.

### State Machine
```
INIT --> SYNCING --> VALID <--> INVALID
  ^                    |           |
  |                    v           |
  +--------------------+-----------+
         (rebuild on gap)
```

| State | Description | Transitions |
|-------|-------------|-------------|
| INIT | No data, buffering deltas | Start snapshot fetch → SYNCING |
| SYNCING | Snapshot received, applying buffered deltas | Valid delta → VALID |
| VALID | Book consistent, publishing quotes | Sequence gap → INVALID |
| INVALID | Sequence gap detected | Reset → INIT |

**Timing during state transitions:**
- Snapshot fetch is NOT timed (one-time initialization)
- Buffered delta replay is included in parse timing
- Only WebSocket depth updates are timed

### Sequence Validation

Per Binance specification:

**First delta after snapshot:**
- Must satisfy: `U <= lastUpdateId + 1 <= u`
- Where `U` = first update ID, `u` = final update ID, `lastUpdateId` = snapshot ID

**Subsequent deltas:**
- Must satisfy: `U == lastUpdateId + 1`
- Any gap triggers immediate invalidation

### Depth Configuration

| Aspect | Value | Rationale |
|--------|-------|-----------|
| Internal book depth | Full (sorted map) | Required for correct delta application |
| Published depth | 5 levels per side | Rich enough for imbalance, manageable schema |
| REST snapshot request | 50 levels | Ensure coverage during reconciliation |

### Publication Discipline

**Publish triggers:**
- Every depth delta that successfully applies (when VALID)
- Validity state changes (VALID → INVALID or vice versa)

**Publication rules:**
- Publish full 5-level snapshot on each update
- Include `isValid` flag for downstream trust
- Include timing measurements (`fhParseUs`, `fhSendUs`)
- Downstream can filter on `isValid = 1b` for clean data

Rationale:
- Simplifies downstream logic (always full depth state)
- Consistent with tick-by-tick trade publishing
- `isValid` flag protects downstream from stale data
- Timing enables performance analysis per handler type

### Invalid State Handling

When sequence gap detected:
1. Mark book INVALID immediately
2. Publish one invalid quote (`isValid = 0b`, prices may be stale, timing still captured)
3. Continue publishing with `isValid = 0b` until rebuilt
4. Reset to INIT state
5. Re-request snapshot

Downstream consumers see explicit invalid state and can react appropriately.

### Schema

**quote_binance table (30 fields):**

| # | Column | Type | Source | Description |
|---|--------|------|--------|-------------|
| 1 | `time` | timestamp | FH | FH receive time (kdb epoch) |
| 2 | `sym` | symbol | Binance | Trading symbol |
| 3-7 | `bidPrice1-5` | float | FH | Bid prices (best to worst) |
| 8-12 | `bidQty1-5` | float | FH | Bid quantities |
| 13-17 | `askPrice1-5` | float | FH | Ask prices (best to worst) |
| 18-22 | `askQty1-5` | float | FH | Ask quantities |
| 23 | `isValid` | boolean | FH | Book validity flag |
| 24 | `exchEventTimeMs` | long | Binance | Exchange event time |
| 25 | `fhRecvTimeUtcNs` | long | FH | Wall-clock receive (ns) |
| 26 | `fhParseUs` | long | FH | Parse + book update latency (μs) |
| 27 | `fhSendUs` | long | FH | Snapshot build + send latency (μs) |
| 28 | `fhSeqNo` | long | FH | FH sequence number |
| 29 | `tpRecvTimeUtcNs` | long | TP | TP receive time (ns) |
| 30 | `rdbApplyTimeUtcNs` | long | RDB | RDB apply time (ns) |

**Field count progression:**
- FH sends: 28 fields (was 26 before timing added)
- TP adds: 1 field (`tpRecvTimeUtcNs`)
- RDB adds: 1 field (`rdbApplyTimeUtcNs`)
- Total: 30 fields (was 28 before timing added)

See `docs/specs/quotes-schema.md` for full schema specification.

### Logging

Single log file per day per ADR-003:
- Combined log: `logs/YYYY.MM.DD.log` (contains both trades and quotes)

Rationale:
- Consistent with ADR-003 ephemeral logging stance
- Single log simplifies tickerplant operation
- Replay processes both tables together

### REST Client

Synchronous HTTPS client for snapshot fetch:
- Uses Boost.Beast (already a dependency)
- Blocking call during SYNCING state
- Fetches 50 levels (configurable)
- Parses JSON response

Trade-off: Blocking snapshot fetch is acceptable because:
- Only occurs on startup and after gaps
- Simpler than async implementation
- Gaps are rare in normal operation

## Rationale

This approach was selected because:

- **Correctness**: Snapshot + delta reconciliation is the only way to build accurate books
- **Explicit validity**: Downstream never sees silently stale data
- **Separation**: Quote FH failure doesn't affect trade ingestion
- **Simplicity**: State machine is clear and testable
- **Performance**: Flat-array design is cache-efficient for multi-symbol workloads
- **Analytics**: 5-level depth enables order book imbalance calculations
- **Observability**: Timing measurements enable performance comparison vs trade handler

### Level 2 (5 levels) vs Level 1 Rationale

Level 2 with 5 levels of depth was selected over Level 1 (best bid/ask only) because:
- Enables multi-level order book imbalance: `(sum bidQty) vs (sum askQty)`
- Provides richer market microstructure data
- Flat schema (22 price/qty columns) is efficient for kdb+ analytics
- Minimal overhead: ~240 bytes/row vs ~100 bytes for L1

### Timing Measurement Rationale

Including timing measurements was selected because:
- Enables performance comparison between trade and quote handlers
- Quote handler has unique characteristics (stateful, complex)
- Per-event timing simpler than separate telemetry
- Consistent with trade handler instrumentation (ADR-001)
- Minimal overhead (2 long fields, <1μs to capture)

## Alternatives Considered

### 1. Integrate into Trade FH (single process)
Rejected:
- Requires threading or async complexity
- Single failure domain
- Mixed concerns
- Cannot independently measure performance

### 2. Use `@depth@100ms` throttled stream
Rejected:
- Still requires snapshot reconciliation
- Higher latency (100ms batches)
- No simplification of reconciliation logic

### 3. Poll REST snapshots only
Rejected:
- High latency (minimum practical poll ~1s)
- Rate limit concerns
- Not real-time

### 4. Trust deltas without validation
Rejected:
- Silent corruption on gaps
- Violates "downstream must trust book" principle
- Debugging nightmare

### 5. Level 1 only (original design)
Superseded:
- Insufficient for multi-level imbalance analytics
- 5-level overhead is minimal with flat-array design
- Wide schema is more efficient than nested lists

### 6. Nested list schema for variable depth
Rejected:
- Higher memory overhead in kdb+
- More complex queries
- Fixed 5-level depth is sufficient and more efficient

### 7. Per-symbol OrderBook objects
Rejected:
- Pointer indirection reduces cache efficiency
- More memory fragmentation
- Flat-array design is faster for multi-symbol updates

### 8. No timing measurements (original design)
Updated:
- Quote handler now instrumented like trade handler
- Enables performance analysis and comparison
- Supports telemetry aggregation (ADR-005)

## Consequences

### Positive

- Correct book state guaranteed by reconciliation
- Explicit validity for downstream trust
- Independent process lifecycle
- Clear state machine for debugging
- Cache-efficient flat-array design
- 5-level depth enables imbalance analytics
- Timing measurements enable performance analysis
- Consistent instrumentation with trade handler
- Per-event timing simpler than separate telemetry

### Negative / Trade-offs

- Two processes to manage instead of one
- REST snapshot introduces brief blocking
- More complex than trade-only system
- Sequence gaps require full rebuild (no partial recovery)
- Wider schema than L1 (30 vs 14 fields)
- Parse timing higher than trade handler (expected due to stateful processing)
- Timing fields increase storage volume (acceptable for exploratory project)

These trade-offs are acceptable for correct Level 2 data with observability.

## Implementation

| Component | File | Purpose |
|-----------|------|---------|
| OrderBookManager | `cpp/include/order_book_manager.hpp` | Flat-array book storage, state machine |
| RestClient | `cpp/include/rest_client.hpp` | Snapshot fetching |
| QuoteFeedHandler | `cpp/include/quote_feed_handler.hpp` | WebSocket, reconciliation, timing |
| Main | `cpp/src/quote_feed_handler.cpp` | Entry point |

**Timing capture locations:**
```cpp
// Parse timing
auto parseStart = std::chrono::steady_clock::now();
processMessage(msg, fhRecvTimeUtcNs);  // includes JSON parse + book update + depth extract
auto parseEnd = std::chrono::steady_clock::now();
lastParseUs_ = duration_cast<microseconds>(parseEnd - parseStart).count();

// Send timing
auto sendStart = std::chrono::steady_clock::now();
// ... build 5-level snapshot + IPC serialization ...
auto sendEnd = std::chrono::steady_clock::now();
long long fhSendUs = duration_cast<microseconds>(sendEnd - sendStart).count();
```

## Derived Analytics

The RTE computes order book imbalance from Level 2 quotes:
```q
/ Sum depth across all 5 levels
bidDepth: sum (bidQty1; bidQty2; bidQty3; bidQty4; bidQty5)
askDepth: sum (askQty1; askQty2; askQty3; askQty4; askQty5)

/ Imbalance: +1 = all bids, -1 = all asks, 0 = balanced
imbalance: (bidDepth - askDepth) % (bidDepth + askDepth)
```

See ADR-004 for RTE analytics details.

## Performance Observations

Based on production telemetry (see ADR-005):

**Quote Feed Handler (p95 latencies):**
- `fhParseUs`: 100-300μs (JSON parse + book update + depth extraction)
- `fhSendUs`: 5-10μs (snapshot build + IPC send)

**Comparison to Trade Feed Handler:**
- Quote parse is 5-10x slower (stateful processing)
- Quote send is similar (both use kdb+ IPC)

These measurements validate:
- Order book management overhead is measurable and acceptable
- IPC serialization cost is similar for both handlers
- Stateful processing accounts for higher parse latency

## Future Evolution

| Enhancement | Trigger |
|-------------|---------|
| 10 or 20 level publication | Deeper analytics requirement |
| Async snapshot fetch | Latency sensitivity |
| Weighted imbalance | Price-weighted analytics |
| Book pressure metrics | Trading strategy requirement |
| E2E latency tracking | Quote-specific E2E SLOs |

## Links / References

- `reference/kdbx-real-time-architecture-reference.md`
- `adr-001-timestamps-and-latency-measurement.md` (timing measurement specification)
- `adr-002-feed-handler-to-kdb-ingestion-path.md` (ingestion architecture)
- `adr-003-tickerplant-logging-and-durability-strategy.md` (single log file per day)
- `adr-004-real-time-analytics-computation.md` (RTE imbalance)
- `adr-005-telemetry-and-metrics-aggregation-strategy.md` (timing aggregation)
- `adr-008-error-handling-strategy.md` (reconnect, error handling)
- `specs/api-binance.md` (depth stream specification)
- `specs/quotes-schema.md` (full schema specification)