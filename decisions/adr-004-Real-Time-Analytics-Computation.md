# ADR-004: Real-Time Analytics Computation (VWAP and Order Book Imbalance)

## Status
Accepted (Updated 2026-01-06)

## Date
Original: 2025-12-17
Updated: 2026-01-06

## Context

The system ingests real-time market data from Binance via two C++ feed handlers:
- **Trade Feed Handler**: Individual trade executions
- **Quote Feed Handler**: L5 order book depth updates

A core objective of the project is to compute **real-time analytics** on this live data:
- **VWAP (Volume-Weighted Average Price)** from trades
- **Order Book Imbalance** from L5 quotes

These analytics must:
- Update incrementally as new data arrives
- Be query-consistent
- Reflect recent data with low latency
- Scale efficiently with message volume
- Avoid full rescans of historical data on each update

A decision is required on **where and how these analytics are computed**.

## Notation

| Acronym | Definition |
|---------|------------|
| IPC | Inter-Process Communication |
| L5 | Level 5 (top 5 bid/ask levels) |
| RDB | Real-Time Database |
| RTE | Real-Time Engine |
| TP | Tickerplant |
| VWAP | Volume-Weighted Average Price |

## Decision

Real-time analytics will be computed in a **Real-Time Engine (RTE)** process using two distinct strategies:
1. **VWAP**: Time-bucketed aggregation (production-grade, memory-efficient)
2. **Order Book Imbalance**: Latest snapshot (stateless, O(1) lookup)

### Architectural Placement

- The **RDB** stores raw trade and quote events and provides query-consistent access.
- The **RTE** computes and maintains analytics as a dedicated process.
- The RTE is a peer to the RDB, not downstream of it during normal operation.

### Subscription Model

- The RTE subscribes **directly to the tickerplant** for live updates.
- The RTE is a peer subscriber alongside the RDB.
- The RTE does not query the RDB during normal operation (only for recovery).

```
TP ──┬──► RDB (storage: trade_binance, quote_binance)
     │
     └──► RTE (analytics: VWAP, Imbalance)
```

### Processing Mode

The RTE processes updates **tick-by-tick**:
- Each incoming trade/quote immediately updates state
- Analytics are recomputed on each update
- No internal batching or buffering

Rationale:
- Matches FH→TP tick-by-tick model (ADR-002)
- Provides lowest latency for analytics updates
- Simplifies latency measurement
- Consistent reasoning across the entire pipeline

---

## Analytics Implementation

### 1. VWAP (Volume-Weighted Average Price)

**Formula:**
```
VWAP = Σ(price × quantity) / Σ(quantity)
```

**Implementation Strategy: Time-Bucketed Aggregation**

Instead of storing individual trades (memory-intensive), RTE aggregates trades into **1-second time buckets**:

| Bucket | Symbol | sumPxQty | sumQty | cnt |
|--------|--------|----------|--------|-----|
| 10:00:01 | BTCUSDT | 500000 | 5.4 | 12 |
| 10:00:02 | BTCUSDT | 480000 | 5.2 | 8 |

**Data Structure:**
```q
vwapBuckets:([]
  sym:`symbol$();
  bucket:`timestamp$();     / 1-second granularity
  sumPxQty:`float$();       / Σ(price × quantity)
  sumQty:`float$();         / Σ(quantity)
  cnt:`long$()              / Trade count
);
`sym`bucket xkey `vwapBuckets;  / Keyed for O(1) upsert
```

**Update Algorithm:**
1. Incoming trade arrives with (time, sym, price, qty)
2. Floor time to 1-second bucket: `bucket = time - (time mod 1000000000)`
3. Upsert into keyed table:
   - If bucket exists: increment sumPxQty, sumQty, cnt
   - If new bucket: insert new row
4. This is **O(1)** regardless of number of trades

**Query Algorithm:**
For 5-minute VWAP query:
1. Calculate cutoff: `cutoff = now - 5 minutes`
2. Select all buckets: `select from vwapBuckets where sym=s, bucket >= cutoff`
3. Sum across buckets:
   ```q
   totalPxQty: sum sumPxQty
   totalQty: sum sumQty
   vwap: totalPxQty % totalQty
   ```
4. This is **O(buckets)** = O(300) for 5-minute window, not O(trades) = O(10,000+)

**Performance Characteristics:**

| Metric | Per-Trade Storage | Bucketed Storage |
|--------|-------------------|------------------|
| Insert time | O(n) growing | O(1) constant |
| Memory/symbol | ~5MB (10k trades) | ~50KB (300 buckets) |
| Query time (5min) | O(n) scan | O(buckets) ≈ O(1) |
| Memory efficiency | 1x | **100x better** |

**Configuration:**
- **Bucket size**: 1 second (configurable via `.rte.cfg.bucketSec`)
- **Default window**: 5 minutes (configurable via `.rte.cfg.defaultWindowMin`)
- **Retention**: 10 minutes (configurable via `.rte.cfg.retentionMin`)

**Cleanup Strategy:**
- Timer runs every 5 seconds (`.rte.cfg.cleanupIntervalMs`)
- Deletes buckets older than retention period
- Prevents unbounded memory growth

**Query Interface:**
```q
/ Get 5-minute VWAP for BTCUSDT
.rte.getVwap[`BTCUSDT; 5]

/ Returns single-row table:
sym     vwap     totalQty tradeCount isValid
--------------------------------------------
BTCUSDT 92521.72 0.185    25         1
```

**Validity Logic:**
- `isValid = 1` if `tradeCount > 10` (sufficient samples)
- `isValid = 0` otherwise (insufficient data)

---

### 2. Order Book Imbalance

**Formula:**
```
Imbalance = (bidDepth - askDepth) / (bidDepth + askDepth)
```

Where:
- `bidDepth = sum(bidQty1 through bidQty5)` (L5 total)
- `askDepth = sum(askQty1 through askQty5)` (L5 total)

**Result Range:** -1.0 (all asks) to +1.0 (all bids)

**Implementation Strategy: Latest Snapshot (Stateless)**

Only the **most recent** imbalance value is stored per symbol:

**Data Structure:**
```q
.rte.imb.latest:()!();  / Dictionary: sym → (bidDepth, askDepth, imbalance, time)
```

**Update Algorithm:**
1. Incoming quote arrives with L5 data
2. Calculate total bid depth: `bidDepth = sum of bidQty1-5`
3. Calculate total ask depth: `askDepth = sum of askQty1-5`
4. Calculate imbalance: `imb = (bidDepth - askDepth) / (bidDepth + askDepth)`
5. Store in dictionary: `.rte.imb.latest[sym] = (bidDepth, askDepth, imb, time)`
6. This is **O(1)** - just a dictionary update

**Query Algorithm:**
```q
/ Get latest imbalance for BTCUSDT
.rte.getImbalance[`BTCUSDT]

/ Returns single-row table:
sym     bidDepth askDepth imbalance time
-----------------------------------------
BTCUSDT 15.2     12.8     0.086     ...
```

**Performance Characteristics:**
- **Memory**: O(symbols) - one value per symbol (~1KB total)
- **Update**: O(1) - dictionary lookup + arithmetic
- **Query**: O(1) - dictionary lookup

**No Historical Data:**
- Only latest snapshot is retained
- No time series or rolling window
- Rationale: Imbalance is point-in-time metric, history not required for current use case

---

## State Schema

### VWAP State (vwapBuckets table)

| Column | Type | Description |
|--------|------|-------------|
| `sym` | symbol | Instrument symbol (key) |
| `bucket` | timestamp | Bucket start time, 1-second granularity (key) |
| `sumPxQty` | float | Σ(price × quantity) for all trades in bucket |
| `sumQty` | float | Σ(quantity) for all trades in bucket |
| `cnt` | long | Number of trades in bucket |

**Key:** (`sym`, `bucket`) - ensures O(1) upsert

### Imbalance State (.rte.imb.latest dictionary)

Per-symbol dictionary entry:
```q
`bidDepth`askDepth`imbalance`time ! (15.2; 12.8; 0.086; 2026.01.06D10:30:45.123)
```

No table structure - just a dictionary for O(1) access.

---

## Query Interface

**VWAP Queries:**
```q
/ Get VWAP for symbol over N minutes
.rte.getVwap[`BTCUSDT; 5]    / 5-minute VWAP
.rte.getVwap[`ETHUSDT; 1]    / 1-minute VWAP

/ Custom window (advanced)
.rte.vwap.calc[`SOLUSDT; 3]  / 3-minute VWAP
```

**Imbalance Queries:**
```q
/ Get latest imbalance for symbol
.rte.getImbalance[`BTCUSDT]
.rte.getImbalance[`ETHUSDT]
```

**Diagnostic Queries:**
```q
/ Check bucket count per symbol
select count i by sym from vwapBuckets

/ Check oldest bucket (data retention)
select min bucket by sym from vwapBuckets

/ Check imbalance dictionary
.rte.imb.latest
```

---

## Recovery Strategy

On RTE restart:

| Step | Action |
|------|--------|
| 1 | Connect and subscribe to tickerplant for live updates |
| 2 | **VWAP**: Start with empty buckets, fill naturally as trades arrive |
| 3 | **Imbalance**: Start with empty dictionary, update on first quote per symbol |
| 4 | Analytics become valid as data accumulates |

**No RDB replay** (differs from original ADR-004):
- VWAP buckets fill naturally within retention window (~10 minutes)
- Imbalance is instant (first quote makes it valid)
- Simpler implementation, faster startup
- Aligns with ADR-003 (ephemeral stance)

This differs from the original design which queried RDB for recovery. Current approach is simpler and sufficient for the project's exploratory nature.

---

## Performance Characteristics

### VWAP (Bucketed)

| Metric | Value | Notes |
|--------|-------|-------|
| Memory/symbol | ~50KB | 600 buckets × ~100 bytes (10-min retention) |
| Insert latency | <1μs p99 | O(1) keyed upsert |
| Query latency (5min) | <100μs p99 | Sum ~300 buckets |
| Cleanup interval | 5 seconds | Timer-based old bucket deletion |

### Order Book Imbalance (Snapshot)

| Metric | Value | Notes |
|--------|-------|-------|
| Memory/symbol | ~100 bytes | Single dictionary entry |
| Update latency | <1μs p99 | Dictionary update + arithmetic |
| Query latency | <1μs p99 | Dictionary lookup |

### System-wide (3 symbols)

| Metric | Value |
|--------|-------|
| Total memory | ~200KB (VWAP + imbalance) |
| CPU usage | Negligible (<1% steady state) |
| Scalability | Can handle 100+ symbols with same approach |

---

## Rationale

This approach was selected because:

### VWAP Bucketing:
- **Production-grade**: Industry-standard approach for time-series aggregation
- **Memory efficient**: 100x reduction vs. per-trade storage
- **Fast queries**: O(buckets) not O(trades)
- **Scalable**: Memory and query time predictable, independent of trade volume
- **Flexible**: Window size configurable without schema changes

### Imbalance Snapshot:
- **Minimal overhead**: O(1) memory and compute
- **Sufficient for use case**: Imbalance is point-in-time metric
- **Simple**: No windowing, no cleanup needed
- **Fast**: Instant updates and queries

### Architectural:
- Clean separation (RDB=storage, RTE=analytics)
- RTE can be restarted independently
- Different strategies for different metrics (fit for purpose)
- Aligns with kdb+/KDB-X best practices

---

## Alternatives Considered

### 1. Per-trade VWAP storage (original design)
Rejected:
- Memory grows unbounded with trade volume
- O(n) insert time as buffer grows
- O(n) query time to scan all trades
- Not production-grade

### 2. Compute VWAP on-demand from RDB
Rejected:
- Repeated full scans on each query
- Poor scalability
- Adds load to RDB
- Higher query latency

### 3. Store full order book history for imbalance trends
Rejected:
- Not required for current use case
- Adds memory overhead
- Adds complexity
- Can be added later if needed

### 4. Single strategy for both metrics
Rejected:
- VWAP needs windowing, imbalance doesn't
- Different data access patterns
- Different performance requirements
- Fit-for-purpose is better than one-size-fits-all

### 5. No cleanup timer (manual cleanup)
Rejected:
- Risk of unbounded memory growth
- Requires external monitoring
- Timer is simple and reliable

---

## Consequences

### Positive

- Production-grade VWAP implementation (bucketed, efficient)
- Minimal overhead imbalance tracking (O(1) everything)
- Clean architectural separation
- Low-latency analytics updates
- Scalable and memory-efficient
- Predictable performance characteristics
- Different strategies optimized for different metrics
- Clear validity indicators for consumers
- Simple recovery (no RDB dependency)

### Negative / Trade-offs

- VWAP state must rebuild after restart (~10 minutes to full window)
- No historical imbalance trends (only latest value)
- Bucketing introduces slight precision loss (acceptable)
- Timer adds minimal complexity
- Cannot query VWAP windows smaller than bucket size (1 second)

These trade-offs are acceptable for the current phase.

---

## Future Evolution

| Enhancement | Trigger |
|-------------|---------|
| Configurable bucket size | Sub-second VWAP requirements |
| Imbalance time series | Historical imbalance analysis |
| Additional metrics | OHLC, spreads, volatility |
| Circular buffers | Memory optimization for 100+ symbols |
| RDB recovery | Requirement for immediate valid state on restart |
| Multiple window sizes | Different consumers need different windows |

---

## Implementation Notes

### VWAP Bucket Calculation
```q
/ Floor timestamp to 1-second bucket
bucketTime:`timestamp$.rte.cfg.bucketNs * `long$time div .rte.cfg.bucketNs;
```

Where `.rte.cfg.bucketNs = 1000000000` (1 second in nanoseconds).

### Imbalance Calculation
```q
total:bidDepth + askDepth;
imb:$[total > 0f; (bidDepth - askDepth) % total; 0n];
```

Returns `0n` (null) if both bid and ask depth are zero (edge case).

---

## Links / References

- `../kdbx-real-time-architecture-reference.md`
- `../kdbx-real-time-architecture-measurement-notes.md`
- `adr-001-timestamps-and-latency-measurement.md` (latency targets)
- `adr-002-feed-handler-to-kdb-ingestion-path.md` (tick-by-tick precedent)
- `adr-003-tickerplant-logging-and-durability-strategy.md` (ephemeral stance)
- `adr-005-telemetry-and-metrics-aggregation-strategy.md` (monitoring RTE)
- `adr-007-visualisation-and-consumption-strategy.md` (dashboard consumption)
- `adr-009-L1-Order-Book-Architecture.md` (quote feed handler details)
