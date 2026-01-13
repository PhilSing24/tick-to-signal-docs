# Spec: Binance Trades Schema (Canonical)

## Purpose

Define the canonical schema for Binance trade events as stored in kdb+/KDB-X, including keys, types, timestamp semantics, and instrumentation fields.

## Naming Conventions

| Element | Convention | Example |
|---------|------------|---------|
| Tables | snake_case | `trade_binance` |
| Columns | lowerCamelCase | `exchTradeTimeMs`, `buyerIsMaker` |
| Timestamps | Unit suffix | `Ms` (milliseconds), `Ns` (nanoseconds), `Us` (microseconds) |
| Time zone | UTC | All timestamps are UTC |

## Source Event (Binance WebSocket)

The Binance `<symbol>@trade` stream provides:

| JSON Field | Type | Description |
|------------|------|-------------|
| `e` | string | Event type (always `"trade"`) |
| `E` | long | Event time (ms since Unix epoch) |
| `T` | long | Trade time (ms since Unix epoch) |
| `s` | string | Symbol (e.g., `"BTCUSDT"`) |
| `t` | long | Trade ID |
| `p` | string | Price (numeric string) |
| `q` | string | Quantity (numeric string) |
| `m` | boolean | Buyer is maker |

## Target Table

### Table Name

`trade_binance`

### Primary Key

Uniqueness key: `(sym, tradeId)`

Used for:
- Deduplication during reconnect/replay scenarios
- Correlation across system components

### Schema (14 Fields)

| # | Column | Type | Source | Description |
|---|--------|------|--------|-------------|
| 1 | `time` | timestamp | FH | FH receive time, converted to kdb epoch. Used for windowing and queries. |
| 2 | `sym` | symbol | Binance `s` | Normalised symbol (e.g., `BTCUSDT`). |
| 3 | `tradeId` | long | Binance `t` | Binance trade ID. Unique per symbol. |
| 4 | `price` | float | Binance `p` | Trade price (parsed from string). |
| 5 | `qty` | float | Binance `q` | Trade quantity (parsed from string). |
| 6 | `buyerIsMaker` | boolean | Binance `m` | `1b` if buyer is market maker. |
| 7 | `exchEventTimeMs` | long | Binance `E` | Exchange event time (ms since Unix epoch). |
| 8 | `exchTradeTimeMs` | long | Binance `T` | Exchange trade time (ms since Unix epoch). |
| 9 | `fhRecvTimeUtcNs` | long | FH | FH wall-clock receive time (ns since Unix epoch). |
| 10 | `fhParseUs` | long | FH | Parse/normalise duration (µs, monotonic). |
| 11 | `fhSendUs` | long | FH | IPC send prep duration (µs, monotonic). |
| 12 | `fhSeqNo` | long | FH | FH sequence number (monotonic per FH instance). |
| 13 | `tpRecvTimeUtcNs` | long | TP | TP receive time (ns since Unix epoch). |
| 14 | `rdbApplyTimeUtcNs` | long | RDB | RDB apply time (ns since Unix epoch). |

### q Schema Definition

```q
trade_binance:([]
  time:`timestamp$();
  sym:`symbol$();
  tradeId:`long$();
  price:`float$();
  qty:`float$();
  buyerIsMaker:`boolean$();
  exchEventTimeMs:`long$();
  exchTradeTimeMs:`long$();
  fhRecvTimeUtcNs:`long$();
  fhParseUs:`long$();
  fhSendUs:`long$();
  fhSeqNo:`long$();
  tpRecvTimeUtcNs:`long$();
  rdbApplyTimeUtcNs:`long$()
  )
```

## Field Categories

### Business Data (Fields 1-8)

Core trade information from Binance, normalised for analytics.

| Field | Notes |
|-------|-------|
| `time` | Derived from `fhRecvTimeUtcNs`, converted to kdb epoch (2000.01.01 base). |
| `price`, `qty` | Parsed from Binance JSON strings to 64-bit float. |
| `exchEventTimeMs`, `exchTradeTimeMs` | Often equal for trades. Both preserved for completeness. |

### Instrumentation Data (Fields 9-14)

Latency measurement and correlation fields.

| Field | Clock Type | Purpose |
|-------|------------|---------|
| `fhRecvTimeUtcNs` | Wall-clock | Cross-process correlation anchor |
| `fhParseUs` | Monotonic | FH segment latency (always trusted) |
| `fhSendUs` | Monotonic | FH segment latency (always trusted) |
| `fhSeqNo` | N/A | Gap detection, ordering |
| `tpRecvTimeUtcNs` | Wall-clock | FH→TP latency calculation |
| `rdbApplyTimeUtcNs` | Wall-clock | TP→RDB latency, E2E latency |

## Timestamp Semantics

### Time Field (`time`)

- **Source:** FH wall-clock at message receipt
- **Conversion:** Unix epoch nanoseconds → kdb epoch timestamp
- **Purpose:** Primary query/filter field, rolling window calculations
- **Note:** This is a copy of `fhRecvTimeUtcNs` in kdb timestamp format

### Exchange Times (`exchEventTimeMs`, `exchTradeTimeMs`)

- **Source:** Binance server
- **Trust:** Indicative only (no clock sync with exchange)
- **Purpose:** Market-to-FH latency estimates, event correlation

### Pipeline Times (`fhRecvTimeUtcNs`, `tpRecvTimeUtcNs`, `rdbApplyTimeUtcNs`)

- **Source:** Respective component wall-clock
- **Trust:** Depends on clock sync quality (see ADR-001)
- **Purpose:** Cross-process latency measurement

### Monotonic Durations (`fhParseUs`, `fhSendUs`)

- **Source:** FH `std::chrono::steady_clock`
- **Trust:** Always trusted (immune to clock adjustments)
- **Purpose:** FH segment latency measurement

**Expected latencies (p95):**
- `fhParseUs`: 20-50µs (simple JSON parse + field extraction)
- `fhSendUs`: 5-10µs (IPC serialization)

## Latency Calculations

| Metric | Formula | Trust |
|--------|---------|-------|
| FH parse latency | `fhParseUs` | Always |
| FH send latency | `fhSendUs` | Always |
| FH → TP | `(tpRecvTimeUtcNs - fhRecvTimeUtcNs) / 1e6` ms | Clock sync dependent |
| TP → RDB | `(rdbApplyTimeUtcNs - tpRecvTimeUtcNs) / 1e6` ms | Clock sync dependent |
| End-to-end | `(rdbApplyTimeUtcNs - fhRecvTimeUtcNs) / 1e6` ms | Clock sync dependent |
| Market → FH | `(fhRecvTimeUtcNs / 1e6) - exchEventTimeMs` ms | Indicative only |

## Sequence Numbers

`fhSeqNo` is a monotonically increasing counter per FH instance.

| Use Case | Method |
|----------|--------|
| Gap detection | Check for non-contiguous values |
| Ordering | Secondary sort after `time` |
| Replay validation | Verify completeness after recovery |

**Note:** `fhSeqNo` resets to 1 on FH restart. Gaps indicate dropped messages.

## Type Rationale

| Field | Type | Rationale |
|-------|------|-----------|
| `price`, `qty` | float | 64-bit precision sufficient; matches kdb conventions |
| `*Ms` fields | long | Milliseconds since epoch fit in 64-bit signed |
| `*Ns` fields | long | Nanoseconds since epoch fit in 64-bit signed |
| `*Us` fields | long | Microseconds; consistent integer type |
| `fhSeqNo` | long | Allows >2B messages per session |

## Design Decisions

### Per-Event Instrumentation

Unlike some architectures that emit timing data separately, this schema stores all instrumentation per-event. This enables:
- Per-event latency debugging
- Correlation without joins
- Simplified telemetry aggregation

Trade-off: Larger row size (~150 bytes vs ~80 bytes without instrumentation).

### Telemetry Aggregation

Per-event data is aggregated in the TEL process (see ADR-005):
- `telemetry_latency_fh` — FH segment percentiles (unified trade + quote)
- `telemetry_latency_e2e` — Cross-process latency percentiles (trades only)
- `telemetry_system` — Memory usage per process

This provides both granular debugging and efficient dashboarding.

## Derived Analytics

The RTE computes VWAP (Volume-Weighted Average Price) from trade data using time-bucketed aggregation:

```q
/ Time-bucketed trade data (1-second buckets)
/ Shared foundation for VWAP and Variance-Covariance
tradeBuckets:([]
  sym:`symbol$();
  bucket:`timestamp$();
  sumPxQty:`float$();   / Σ(price × qty)
  sumQty:`float$();     / Σ(qty)
  cnt:`long$()
);

/ Query: 5-minute VWAP
vwap: sum[sumPxQty] % sum[sumQty]
```

See ADR-004 for RTE analytics details.

## Log Replay

Trade data is logged by TP to `logs/YYYY.MM.DD.log` (single file containing all market data) in `-11!` compatible format. On RDB restart, trades are automatically replayed:

```q
/ Automatic replay on RDB startup
.rdb.replay[.z.D]
```

See ADR-006 for recovery and replay details.

## Version History

| Version | Date | Changes |
|---------|------|---------|
| 1.0 | 2025-12-17 | Initial 9-field schema |
| 2.0 | 2025-12-18 | Extended to 14 fields with full instrumentation |
| 2.1 | 2026-01-03 | Added RTE VWAP reference |
| 2.2 | 2026-01-07 | Added log replay reference, TEL process reference |
| 2.3 | 2026-01-11 | Renamed vwapBuckets to tradeBuckets, fixed log file reference |

## References

- `adr-001-timestamps-and-latency-measurement.md` — Clock types, trust model
- `adr-002-feed-handler-to-kdb-ingestion-path.md` — FH→TP data flow
- `adr-004-real-time-rolling-analytics-computation.md` — RTE analytics (VWAP)
- `adr-005-telemetry-and-metrics-aggregation-strategy.md` — TEL aggregation
- `adr-006-recovery-and-replay-strategy.md` — Log replay
- `quotes-schema.md` — L5 quote table schema
- `kdbx-real-time-architecture-measurement-notes.md` — Measurement implementation
