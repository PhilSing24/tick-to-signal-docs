# Spec: Binance Quotes Schema (Canonical)

## Purpose

Define the canonical schema for Binance L5 order book quotes as stored in kdb+/KDB-X, including keys, types, timestamp semantics, and instrumentation fields.

## Naming Conventions

| Element | Convention | Example |
|---------|------------|---------|
| Tables | snake_case | `quote_binance` |
| Columns | lowerCamelCase | `bidPrice1`, `askQty5` |
| Timestamps | Unit suffix | `Ms` (milliseconds), `Ns` (nanoseconds), `Us` (microseconds) |
| Time zone | UTC | All timestamps are UTC |

## Source Event (Binance WebSocket)

The Binance `<symbol>@depth@100ms` stream provides incremental order book updates:

| JSON Field | Type | Description |
|------------|------|-------------|
| `e` | string | Event type (`"depthUpdate"`) |
| `E` | long | Event time (ms since Unix epoch) |
| `s` | string | Symbol (e.g., `"BTCUSDT"`) |
| `U` | long | First update ID in event |
| `u` | long | Final update ID in event |
| `b` | array | Bid updates `[[price, qty], ...]` |
| `a` | array | Ask updates `[[price, qty], ...]` |

The feed handler maintains full order book state and publishes L5 snapshots.

## Order Book Synchronisation

The quote feed handler implements the Binance order book sync protocol:

1. **Buffer** WebSocket depth updates
2. **Fetch** REST snapshot (`/api/v3/depth?symbol=X&limit=1000`)
3. **Validate** snapshot `lastUpdateId` against buffered updates
4. **Replay** buffered updates where `U <= lastUpdateId+1 <= u`
5. **Apply** subsequent updates, validating sequence continuity

State machine: `INIT → SYNCING → VALID ↔ INVALID`

## Target Table

### Table Name

`quote_binance`

### Primary Key

No strict uniqueness key (quotes are snapshots, not unique events).

For ordering/correlation: `(sym, time, fhSeqNo)`

### Schema (30 Fields)

| # | Column | Type | Source | Description |
|---|--------|------|--------|-------------|
| 1 | `time` | timestamp | FH | FH receive time, converted to kdb epoch |
| 2 | `sym` | symbol | Binance | Normalised symbol (e.g., `BTCUSDT`) |
| 3 | `bidPrice1` | float | FH | Best bid price (level 1) |
| 4 | `bidPrice2` | float | FH | Bid price level 2 |
| 5 | `bidPrice3` | float | FH | Bid price level 3 |
| 6 | `bidPrice4` | float | FH | Bid price level 4 |
| 7 | `bidPrice5` | float | FH | Bid price level 5 (worst) |
| 8 | `bidQty1` | float | FH | Best bid quantity |
| 9 | `bidQty2` | float | FH | Bid quantity level 2 |
| 10 | `bidQty3` | float | FH | Bid quantity level 3 |
| 11 | `bidQty4` | float | FH | Bid quantity level 4 |
| 12 | `bidQty5` | float | FH | Bid quantity level 5 |
| 13 | `askPrice1` | float | FH | Best ask price (level 1) |
| 14 | `askPrice2` | float | FH | Ask price level 2 |
| 15 | `askPrice3` | float | FH | Ask price level 3 |
| 16 | `askPrice4` | float | FH | Ask price level 4 |
| 17 | `askPrice5` | float | FH | Ask price level 5 (worst) |
| 18 | `askQty1` | float | FH | Best ask quantity |
| 19 | `askQty2` | float | FH | Ask quantity level 2 |
| 20 | `askQty3` | float | FH | Ask quantity level 3 |
| 21 | `askQty4` | float | FH | Ask quantity level 4 |
| 22 | `askQty5` | float | FH | Ask quantity level 5 |
| 23 | `isValid` | boolean | FH | `1b` if order book state is valid |
| 24 | `exchEventTimeMs` | long | Binance `E` | Exchange event time (ms since Unix epoch) |
| 25 | `fhRecvTimeUtcNs` | long | FH | FH wall-clock receive time (ns since Unix epoch) |
| 26 | `fhParseUs` | long | FH | Parse + book update + L5 extract duration (µs, monotonic) |
| 27 | `fhSendUs` | long | FH | L5 build + IPC send duration (µs, monotonic) |
| 28 | `fhSeqNo` | long | FH | FH sequence number (monotonic per FH instance) |
| 29 | `tpRecvTimeUtcNs` | long | TP | TP receive time (ns since Unix epoch) |
| 30 | `rdbApplyTimeUtcNs` | long | RDB | RDB apply time (ns since Unix epoch) |

### q Schema Definition

```q
quote_binance:([]
  time:`timestamp$();
  sym:`symbol$();
  / L5 bid prices (best to worst)
  bidPrice1:`float$();
  bidPrice2:`float$();
  bidPrice3:`float$();
  bidPrice4:`float$();
  bidPrice5:`float$();
  / L5 bid quantities
  bidQty1:`float$();
  bidQty2:`float$();
  bidQty3:`float$();
  bidQty4:`float$();
  bidQty5:`float$();
  / L5 ask prices (best to worst)
  askPrice1:`float$();
  askPrice2:`float$();
  askPrice3:`float$();
  askPrice4:`float$();
  askPrice5:`float$();
  / L5 ask quantities
  askQty1:`float$();
  askQty2:`float$();
  askQty3:`float$();
  askQty4:`float$();
  askQty5:`float$();
  / Metadata and instrumentation
  isValid:`boolean$();
  exchEventTimeMs:`long$();
  fhRecvTimeUtcNs:`long$();
  fhParseUs:`long$();
  fhSendUs:`long$();
  fhSeqNo:`long$();
  tpRecvTimeUtcNs:`long$();
  rdbApplyTimeUtcNs:`long$()
  )
```

## Field Categories

### Market Data (Fields 1-22)

L5 order book snapshot from feed handler.

| Field Group | Count | Description |
|-------------|-------|-------------|
| `bidPrice1-5` | 5 | Bid prices, best (1) to worst (5) |
| `bidQty1-5` | 5 | Bid quantities at each level |
| `askPrice1-5` | 5 | Ask prices, best (1) to worst (5) |
| `askQty1-5` | 5 | Ask quantities at each level |

**Level Ordering:**
- Level 1 = Best bid/ask (top of book)
- Level 5 = 5th best bid/ask
- Bids: `bidPrice1 > bidPrice2 > ... > bidPrice5`
- Asks: `askPrice1 < askPrice2 < ... < askPrice5`

**Empty Levels:**
- If fewer than 5 levels exist, remaining fields are `0n` (null)
- `isValid = 0b` indicates book may be stale or incomplete

### Validity Flag (Field 23)

| Value | Meaning |
|-------|---------|
| `1b` | Order book is synchronised and valid |
| `0b` | Order book is stale, incomplete, or resyncing |

The feed handler sets `isValid = 0b` when:
- Initial sync in progress (INIT, SYNCING states)
- Sequence gap detected (triggers resync)
- REST snapshot fetch in progress

### Instrumentation Data (Fields 24-30)

| Field | Clock Type | Purpose |
|-------|------------|---------|
| `exchEventTimeMs` | Binance server | Event correlation, market-to-FH latency estimate |
| `fhRecvTimeUtcNs` | Wall-clock | Cross-process correlation anchor |
| `fhParseUs` | Monotonic | FH segment latency (JSON parse + book update + L5 extract) |
| `fhSendUs` | Monotonic | FH segment latency (L5 build + IPC send) |
| `fhSeqNo` | N/A | Gap detection, ordering |
| `tpRecvTimeUtcNs` | Wall-clock | FH→TP latency calculation |
| `rdbApplyTimeUtcNs` | Wall-clock | TP→RDB latency, E2E latency |

## Timestamp Semantics

### Time Field (`time`)

- **Source:** FH wall-clock at depth update receipt
- **Conversion:** Unix epoch nanoseconds → kdb epoch timestamp
- **Purpose:** Primary query/filter field

### Exchange Time (`exchEventTimeMs`)

- **Source:** Binance `E` field from depth update
- **Trust:** Indicative only (no clock sync with exchange)
- **Purpose:** Market-to-FH latency estimates

### Monotonic Durations (`fhParseUs`, `fhSendUs`)

- **Source:** FH `std::chrono::steady_clock`
- **Trust:** Always trusted (immune to clock adjustments)
- **Purpose:** FH segment latency measurement

**Quote-specific timing:**
- `fhParseUs` includes: JSON parse + order book state update + L5 extraction
- `fhSendUs` includes: L5 snapshot construction + kdb+ IPC serialization
- Parse timing is significantly higher than trade handler due to stateful processing

**Expected latencies (p95):**
- `fhParseUs`: 100-300µs (vs 20-50µs for trades)
- `fhSendUs`: 5-10µs (similar to trades)

### Pipeline Times

Same semantics as trade schema (see `trades-schema.md`).

## Latency Calculations

| Metric | Formula | Trust |
|--------|---------|-------|
| FH parse latency | `fhParseUs` | Always |
| FH send latency | `fhSendUs` | Always |
| FH → TP | `(tpRecvTimeUtcNs - fhRecvTimeUtcNs) / 1e6` ms | Clock sync dependent |
| TP → RDB | `(rdbApplyTimeUtcNs - tpRecvTimeUtcNs) / 1e6` ms | Clock sync dependent |
| End-to-end | `(rdbApplyTimeUtcNs - fhRecvTimeUtcNs) / 1e6` ms | Clock sync dependent |
| Market → FH | `(fhRecvTimeUtcNs / 1e6) - exchEventTimeMs` ms | Indicative only |

## Derived Analytics

The RTE computes order book imbalance from L5 quotes:

```q
/ Total depth on each side
bidDepth: bidQty1 + bidQty2 + bidQty3 + bidQty4 + bidQty5
askDepth: askQty1 + askQty2 + askQty3 + askQty4 + askQty5

/ Imbalance: +1 = all bids, -1 = all asks, 0 = balanced
imbalance: (bidDepth - askDepth) % (bidDepth + askDepth)
```

See ADR-004 for RTE analytics details.

## Sequence Numbers

`fhSeqNo` is a monotonically increasing counter per quote FH instance.

| Use Case | Method |
|----------|--------|
| Gap detection | Check for non-contiguous values |
| Ordering | Secondary sort after `time` |

**Note:** `fhSeqNo` resets to 1 on FH restart.

## Type Rationale

| Field | Type | Rationale |
|-------|------|-----------|
| `*Price*`, `*Qty*` | float | 64-bit precision sufficient; matches kdb conventions |
| `isValid` | boolean | Simple flag for book validity |
| `*Ms` fields | long | Milliseconds since epoch |
| `*Ns` fields | long | Nanoseconds since epoch |
| `*Us` fields | long | Microseconds; consistent integer type |
| `fhSeqNo` | long | Allows >2B messages per session |

## Design Decisions

### L5 vs L1

L5 depth was chosen over L1 (best bid/ask only) because:
- Enables multi-level imbalance calculations
- Provides richer market microstructure data
- Flat schema is more efficient than nested lists
- Memory overhead is minimal (~240 bytes/row)

### Wide Schema vs Nested

Wide schema (22 separate price/qty columns) was chosen over nested lists because:
- Fixed depth (5 levels) is known at design time
- Avoids nested list overhead in kdb+
- Enables efficient columnar operations
- Simpler queries (`select bidPrice1` vs `select first each bidPrices`)

### Parse/Send Durations

The quote feed handler captures `fhParseUs` and `fhSendUs` (added 2026-01-06):
- Enables performance comparison with trade handler
- `fhParseUs` includes stateful book update (higher than trade)
- `fhSendUs` is similar to trade handler (IPC serialization)
- Per-event timing enables detailed latency analysis

## OrderBookManager Architecture

The quote feed handler uses `OrderBookManager` for efficient L5 state:

| Feature | Implementation |
|---------|----------------|
| Memory layout | Flat array, contiguous (~16KB for 100 symbols) |
| Symbol lookup | O(1) via index map |
| State machine | Per-symbol: INIT → SYNCING → VALID ↔ INVALID |
| Update | Apply delta to sorted price map, extract top 5 |

See `cpp/include/order_book_manager.hpp` for implementation.

## Version History

| Version | Date | Changes |
|---------|------|---------|
| 1.0 | 2025-12-29 | Initial L1 schema (11 fields) |
| 2.0 | 2026-01-03 | Extended to L5 schema (28 fields) |
| 2.1 | 2026-01-06 | Added `fhParseUs`, `fhSendUs` (30 fields) |

## References

- `adr-001-timestamps-and-latency-measurement.md` — Clock types, trust model
- `adr-002-feed-handler-to-kdb-ingestion-path.md` — FH→TP data flow
- `adr-004-real-time-rolling-analytics-computation.md` — RTE imbalance calculation
- `adr-005-telemetry-and-metrics-aggregation-strategy.md` — Telemetry aggregation
- `adr-009-l5-order-book-architecture.md` — Quote handler architecture
- `trades-schema.md` — Trade table schema (comparison)
- `cpp/include/order_book_manager.hpp` — OrderBookManager implementation
