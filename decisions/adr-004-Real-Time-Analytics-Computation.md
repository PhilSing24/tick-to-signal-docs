# ADR-004: Real-Time Analytics Computation

## Status
Accepted (Updated 2026-01-16)

## Date
Original: 2025-12-17
Updated: 2026-01-11, 2026-01-16

## Context

The system ingests real-time market data from Binance via two C++ feed handlers:
- **Trade Feed Handler**: Individual trade executions
- **Quote Feed Handler**: L2 order book depth updates

A core objective of the project is to compute **real-time analytics** on this live data:
- **VWAP (Volume-Weighted Average Price)** from trades
- **Order Book Imbalance** from L2 quotes (with EMA smoothing)
- **Variance-Covariance Matrix** from trades (cross-asset correlation)

These analytics must:
- Update incrementally as new data arrives
- Be query-consistent
- Reflect recent data with low latency
- Scale efficiently with message volume
- Avoid full rescans of historical data on each update

A decision is required on **where and how these analytics are computed**.

**Note:** Latency tracking has been moved to the Telemetry process (tel.q) as per ADR-005.

## Notation

| Acronym | Definition |
|---------|------------|
| EMA | Exponential Moving Average |
| IPC | Inter-Process Communication |
| L2 | Level 2 (top 5 bid/ask levels) |
| MLE | Machine Learning Engine |
| OBI | Order Book Imbalance |
| RTE | Real-Time Engine |
| TP | Tickerplant |
| VWAP | Volume-Weighted Average Price |
| WDB | Write-only Database (intraday writedown to HDB) |

## Decision

Real-time analytics will be computed in a **Real-Time Engine (RTE)** process using distinct strategies per metric:

| Metric | Strategy | Window | Update Path |
|--------|----------|--------|-------------|
| VWAP | Time-bucketed aggregation | 1-10 min | Hot (every trade) |
| Order Book Imbalance | Latest + EMA smoothing | Point-in-time | Hot (every quote) |
| OBI History | Rolling table | 7 min | Hot (every quote) |
| L2 Order Book | Latest snapshot | Point-in-time | Hot (every quote) |
| Variance-Covariance | Resampled calculation | 1 hour | Cold (timer, 5s) |

### Architectural Placement

- The **WDB** stores raw trade and quote events and writes to HDB.
- The **RTE** computes and maintains analytics as a dedicated process.
- The RTE is a peer to the WDB, not downstream of it during normal operation.

### Subscription Model

- The RTE subscribes **directly to the tickerplant** for live updates.
- The RTE is a peer subscriber alongside the WDB and MLE.
- The RTE does not query the WDB during normal operation (only for recovery).

```
TP:5010 ──┬──► WDB:5012 (storage: trade_binance, quote_binance → HDB)
          │
          ├──► RTE:5013 (analytics: VWAP, Imbalance, VarCovar)
          │
          └──► MLE:5015 (information-driven bars: DIB, DRB)
```

### Processing Mode

The RTE processes updates **tick-by-tick**:
- Each incoming trade/quote immediately updates state
- Analytics are recomputed on each update (hot path) or timer (cold path)
- No internal batching or buffering

Rationale:
- Matches FH→TP tick-by-tick model (ADR-002)
- Provides lowest latency for analytics updates
- Simplifies latency measurement
- Consistent reasoning across the entire pipeline

---

## Analytics Implementation

### 1. Trade Buckets (Shared State)

Trade buckets are the **shared foundation** for VWAP and Variance-Covariance calculations. Instead of storing individual trades, RTE aggregates trades into **1-second time buckets**:

| Bucket | Symbol | sumPxQty | sumQty | cnt |
|--------|--------|----------|--------|-----|
| 10:00:01 | BTCUSDT | 500000 | 5.4 | 12 |
| 10:00:02 | BTCUSDT | 480000 | 5.2 | 8 |

**Data Structure:**
```q
tradeBuckets:([]
  sym:`symbol$();
  bucket:`timestamp$();     / 1-second granularity
  sumPxQty:`float$();       / Σ(price × quantity)
  sumQty:`float$();         / Σ(quantity)
  cnt:`long$()              / Trade count
);
`sym`bucket xkey `tradeBuckets;  / Keyed for O(1) upsert
```

**Update Algorithm (Hot Path):**
1. Incoming trade arrives with (time, sym, price, qty)
2. Floor time to 1-second bucket: `bucket = time - (time mod 1000000000)`
3. Upsert into keyed table:
   - If bucket exists: increment sumPxQty, sumQty, cnt
   - If new bucket: insert new row
4. This is **O(1)** regardless of number of trades

**Configuration:**
```q
.rte.cfg.bucketSec:1;              / 1-second buckets (for VWAP precision)
.rte.cfg.bucketRetentionMin:65;    / Keep 65 minutes (for 1-hour vcov window)
```

**Why 1-second buckets with 65-minute retention:**
- VWAP needs fine granularity (1-sec) for precision
- Var-covar needs long window (60-min) for meaningful correlations
- 65 minutes provides buffer for 1-hour calculations

---

### 2. VWAP (Volume-Weighted Average Price)

**Formula:**
```
VWAP = Σ(price × quantity) / Σ(quantity)
```

**Query Algorithm:**
For 5-minute VWAP query:
1. Calculate cutoff: `cutoff = now - 5 minutes`
2. Select all buckets: `select from tradeBuckets where sym=s, bucket >= cutoff`
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

### 3. Variance-Covariance Matrix

**Purpose:** Measure cross-asset correlation and volatility for portfolio analysis.

**Implementation Strategy: Resampled Calculation (Cold Path)**

Var-covar uses a different time scale than VWAP:
- **VWAP**: 1-second buckets, short windows (1-10 min)
- **Var-covar**: 30-second resampling, long window (1 hour)

**Why different time scales:**
- 1-second returns are noisy (price barely moves)
- 30-second returns capture meaningful price changes
- 1-hour window provides ~120 data points for stable statistics

**Data Flow:**
```
tradeBuckets (1-sec) → Resample to 30-sec → Log returns → Covariance matrix
```

**Algorithm:**
1. Extract prices from 1-sec buckets within window
2. Resample to 30-sec buckets (last price per bucket)
3. Forward-fill missing prices (assume unchanged)
4. Compute log returns: `log(price[t] / price[t-1])`
5. Find valid indices (all symbols have data)
6. Build var-covar matrix using `cov` function

**Data Structures:**
```q
/ Latest matrix (updated every 5 seconds by timer)
.rte.vcov.latest:`syms`matrix`window`buckets`isValid!(...)

/ History for charting (flattened)
.rte.vcov.history:([] time:`timestamp$(); sym1:`symbol$(); sym2:`symbol$(); covar:`float$())
```

**Configuration:**
```q
.rte.cfg.vcovWindowMin:60;      / 1-hour window
.rte.cfg.vcovBucketSec:30;      / Resample to 30-sec
.rte.cfg.vcovMinBuckets:100;    / Need ~100 of 120 for valid matrix
.rte.cfg.vcovRetentionMin:15;   / Keep 15 min of history
```

**Query Interface:**
```q
/ Calculate fresh matrix (any window)
.rte.getVarCovar[60]

/ Returns:
syms   | `BTCUSDT`ETHUSDT`SOLUSDT
matrix | (3x3 covariance matrix)
window | 60
buckets| 120
isValid| 1b

/ Get latest matrix (from timer)
.rte.getVcov[]

/ Get correlation matrix (derived)
.rte.getCorrelation[]

/ Returns:
       | BTCUSDT   ETHUSDT   SOLUSDT
-------| -----------------------------
BTCUSDT| 1         0.576     0.422
ETHUSDT| 0.576     1         0.532
SOLUSDT| 0.422     0.532     1

/ Get correlation as table (for dashboard grid)
.rte.getCorrelationTable[]

/ Get annualized volatility
.rte.getAnnualizedVol[]

/ Returns:
BTCUSDT| 0.084
ETHUSDT| 0.117
SOLUSDT| 0.317

/ Get covariance history for charting
.rte.getVcovHistory[`BTCUSDT; `ETHUSDT; 15]
```

**Validity Logic:**
- `isValid = 1b` when `buckets >= .rte.cfg.vcovMinBuckets` (100)
- Requires ~50 minutes warm-up for valid 1-hour calculation
- Matrix is computed but flagged invalid during warm-up

---

### 4. Order Book Imbalance (OBI)

**Purpose:** Measure buying vs selling pressure from L2 depth.

**Formula:**
```
OBI = (bidDepth - askDepth) / (bidDepth + askDepth)
```
Where depth = sum of quantities across 5 levels.

**Range:** -1 (all asks) to +1 (all bids), 0 = balanced

**EMA Smoothing:**
Raw OBI is noisy; we apply Exponential Moving Average:
```
smOBI[t] = α × OBI[t] + (1 - α) × smOBI[t-1]
```

**Configuration:**
```q
.rte.cfg.obiAlpha:0.05;        / Lower = smoother (5% weight to new value)
.rte.cfg.obiThreshold:0.3;     / Threshold for buyer/seller pressure
.rte.cfg.obiRetentionMin:7;    / Keep 7 minutes of history
```

**Data Structures:**
```q
.rte.imb.latest    / Latest OBI per symbol (dict)
.rte.imb.ema       / EMA state per symbol (dict)
.rte.imb.history   / Rolling history table
```

**Pressure Classification:**
| smOBI | Pressure |
|-------|----------|
| > 0.3 | `buyer` |
| < -0.3 | `seller` |
| else | `neutral` |

**Query Interface:**
```q
/ Single symbol
.rte.getImbalance[`BTCUSDT]

/ All symbols with pressure
.rte.getImbalanceAll[]

/ Returns:
sym     OBI    smOBI  pressure
------------------------------
BTCUSDT 0.15   0.22   neutral
ETHUSDT 0.45   0.38   buyer
SOLUSDT -0.32  -0.28  neutral

/ History for charting
.rte.getOBIHistory[`BTCUSDT; 5]  / Last 5 minutes
```

---

### 5. L2 Order Book State

**Purpose:** Maintain latest L2 snapshot for dashboard display.

**Storage Strategy: Raw List (Hot Path Optimized)**

The order book is stored as a **raw list** for minimal update overhead:
```q
.rte.book.latest[`BTCUSDT]
/ Indices 0-4:   bidPrice1-5
/ Indices 5-9:   bidQty1-5
/ Indices 10-14: askPrice1-5
/ Indices 15-19: askQty1-5
/ Index 20:      time
```

**Update (Hot Path):**
```q
.rte.book.update:{[s;data;time]
  .rte.book.latest[s]:data[2 3 4 5 6 7 8 9 10 11 12 13 14 15 16 17 18 19 20 21],time;
  };
```

**Query (Cold Path - Formatted for Display):**
```q
.rte.getOrderBook[`BTCUSDT]

/ Returns table:
bidQty   bidPrice  askPrice  askQty
-----------------------------------
0.5      90700     90701     0.3
1.2      90699     90702     0.8
...

.rte.getSpread[`BTCUSDT]

/ Returns:
spread | 1
mid    | 90700.5
bestBid| 90700
bestAsk| 90701
time   | ...
```

---

## Dashboard Query Functions

The RTE provides specialized query functions optimized for KX Dashboard consumption:

### Symbol Summary Table
```q
/ Get summary for all symbols with VWAP trend
.rte.getSummary[1;5]  / 1-min vs 5-min VWAP

/ Returns:
sym     lastPrice  vwapShort  vwapLong   trend
----------------------------------------------
BTCUSDT 94521.00   94518.50   94502.30   up
ETHUSDT 3321.50    3320.80    3318.40    up
SOLUSDT 185.20     185.10     185.50     down
```

### Volatility Comparison Table
```q
/ Compare annualized vol to implied vol
.rte.getVolComparison[]

/ Returns:
sym     annualizedVol impliedVol vsIVol
---------------------------------------
BTCUSDT 8.41          45.00      below
ETHUSDT 11.68         65.00      below
SOLUSDT 31.69         67.00      below
```

**Configuration for implied volatility:**
```q
.rte.cfg.iVol:`BTCUSDT`ETHUSDT`SOLUSDT!45.00 65.00 67.00;  / 30-day IV (%)
```

---

## State Schema Summary

| State | Type | Memory | Update | Cleanup |
|-------|------|--------|--------|---------|
| tradeBuckets | Keyed table | ~400KB (65min × 3sym) | O(1) upsert | Timer (5s) |
| .rte.imb.latest | Dictionary | ~300 bytes | O(1) | None |
| .rte.imb.ema | Dictionary | ~100 bytes | O(1) | None |
| .rte.imb.history | Table | ~2MB (7min) | O(1) append | Timer (5s) |
| .rte.book.latest | Dictionary | ~500 bytes | O(1) | None |
| .rte.vcov.latest | Dictionary | ~1KB | Timer (5s) | None |
| .rte.vcov.history | Table | ~50KB (15min) | Timer (5s) | Timer (5s) |

---

## Query Interface Summary

**VWAP:**
```q
.rte.getVwap[`BTCUSDT; 5]              / 5-minute VWAP
.rte.getSummary[1; 5]                  / All symbols with VWAP trend
```

**Order Book:**
```q
.rte.getOrderBook[`BTCUSDT]            / L2 order book table
.rte.getSpread[`BTCUSDT]               / Spread and mid-price
.rte.getImbalance[`BTCUSDT]            / Order book imbalance (single symbol)
.rte.getImbalanceAll[]                 / All symbols with pressure
.rte.getOBIHistory[`BTCUSDT; 5]        / OBI history for charting
```

**Variance-Covariance:**
```q
.rte.getVarCovar[60]                   / Calculate fresh matrix
.rte.getVcov[]                         / Latest matrix (from timer)
.rte.getCorrelation[]                  / Correlation matrix (dict)
.rte.getCorrelationTable[]             / Correlation matrix (table for dashboard)
.rte.getAnnualizedVol[]                / Annualized volatility
.rte.getVolComparison[]                / Vol vs implied vol
.rte.getVcovHistory[`BTCUSDT;`ETHUSDT;15]  / Covariance history
```

---

## Timer Functions

The RTE timer runs every 5 seconds:

```q
.z.ts:{[]
  .rte.bucket.cleanup[];      / Remove old trade buckets (> 65 min)
  .rte.vcov.update[];         / Recalculate var-covar matrix
  .rte.vcov.cleanup[];        / Remove old vcov history (> 15 min)
  .rte.imb.cleanup[];         / Remove old OBI history (> 7 min)
  };
```

---

## Recovery Strategy

On RTE restart:

| Step | Action |
|------|--------|
| 1 | Replay today's TP log file (if exists) |
| 2 | Connect and subscribe to tickerplant for live updates |
| 3 | Apply cleanup to remove data outside retention windows |
| 4 | Analytics become valid as data accumulates |

**Warm-up Times:**
- VWAP: ~5 minutes for valid 5-min calculation
- Imbalance: Instant (first quote makes it valid)
- Order Book: Instant (first quote makes it valid)
- Var-Covar: ~50 minutes for valid 1-hour calculation

---

## Performance Characteristics

### Hot Path (Per-Trade/Quote)

| Operation | Latency | Notes |
|-----------|---------|-------|
| Trade bucket upsert | <1μs | O(1) keyed upsert |
| Order book update | <1μs | Raw list assignment |
| Imbalance + EMA update | <1μs | Dictionary + arithmetic |
| OBI history append | <1μs | Table append |

### Cold Path (Query/Timer)

| Operation | Latency | Notes |
|-----------|---------|-------|
| VWAP query (5min) | <100μs | Sum ~300 buckets |
| Order book query | <10μs | Format list to table |
| Var-covar calculation | <10ms | 120 buckets × 3 symbols |
| Correlation derivation | <1ms | Matrix operations |
| OBI history query | <1ms | Filter ~3000 rows |

### Memory (3 symbols)

| Component | Memory | Notes |
|-----------|--------|-------|
| Trade buckets | ~400KB | 65min × 60sec × 3sym × ~100 bytes |
| OBI history | ~2MB | 7min of quote imbalances |
| Vcov history | ~50KB | 15min of matrix snapshots |
| Snapshots | ~2KB | Order book, imbalance latest |
| **Total** | **~3MB** | Scales linearly with symbols |

---

## Rationale

### Trade Buckets (Shared State):
- **Reuse**: One storage serves both VWAP and var-covar
- **Flexibility**: Different window sizes for different analytics
- **Efficiency**: O(1) insert, O(buckets) query

### VWAP Bucketing:
- **Production-grade**: Industry-standard approach
- **Memory efficient**: 100x reduction vs. per-trade storage
- **Fast queries**: O(buckets) not O(trades)

### OBI EMA Smoothing:
- **Noise reduction**: Raw OBI too volatile for trading signals
- **Configurable**: Alpha adjusts responsiveness vs stability
- **Pressure indicator**: Clear buyer/seller classification

### Var-Covar Resampling:
- **Meaningful returns**: 30-sec vs noisy 1-sec
- **Stable statistics**: 120 data points per calculation
- **Cold path**: Expensive calculation runs on timer, not per-trade

### Order Book Raw List:
- **Hot path optimized**: Single vector slice on update
- **Cold path formatting**: Table created only when queried

### Latency Tracking Moved to Telemetry:
- **Separation of concerns**: Analytics vs monitoring
- **RTE focused**: Core analytics only
- **See ADR-005**: Telemetry handles latency

---

## Alternatives Considered

### 1. Separate VWAP and var-covar buckets
Rejected:
- Redundant storage
- More complex cleanup
- Trade buckets serve both well

### 2. Real-time var-covar (per-trade)
Rejected:
- Too expensive for hot path
- 1-sec returns are noisy anyway
- Timer-based is sufficient

### 3. Store formatted order book tables
Rejected:
- Wastes allocation on hot path
- Raw list is faster to update
- Formatting can be deferred to query

### 4. Single bucket size for all analytics
Rejected:
- VWAP needs precision (1-sec)
- Var-covar needs stability (30-sec)
- Resample at query time is better

### 5. Store raw OBI without smoothing
Rejected:
- Too noisy for trading signals
- EMA provides cleaner indicator
- History still stores both raw and smoothed

---

## Consequences

### Positive

- Production-grade VWAP implementation
- Cross-asset correlation analysis (var-covar)
- Smoothed OBI with pressure indicator
- OBI history for trend analysis
- L2 order book for dashboard
- Dashboard-optimized query functions
- Efficient memory usage (~3MB total)
- Hot path optimized (<1μs updates)
- Clear validity indicators

### Negative / Trade-offs

- VWAP state must rebuild after restart (~5 minutes)
- Var-covar needs warm-up (~50 minutes for valid)
- OBI history limited to 7 minutes
- Bucketing introduces slight precision loss

---

## Future Evolution

| Enhancement | Trigger |
|-------------|---------|
| Annualized var-covar storage | Portfolio risk dashboard |
| Configurable OBI alpha per symbol | Different volatility profiles |
| Configurable vcov window | Different consumer needs |
| More symbols | Scale to 100+ instruments |
| OHLC bars | Charting requirements |

---

## Links / References

- `adr-001-timestamps-and-latency-measurement.md` (latency targets)
- `adr-002-feed-handler-to-kdb-ingestion-path.md` (tick-by-tick precedent)
- `adr-003-tickerplant-logging-and-durability-strategy.md` (ephemeral stance)
- `adr-005-telemetry-and-metrics-aggregation-strategy.md` (latency tracking moved here)
- `adr-007-visualisation-and-consumption-strategy.md` (dashboard consumption)
- `adr-009-Order-Book-Architecture.md` (quote feed handler details)
