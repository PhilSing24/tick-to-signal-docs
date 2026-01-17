# ADR-011: Financial Machine Learning Engine

## Status
Accepted (Updated 2026-01-18)

## Date
2026-01-15 (Updated 2026-01-16, 2026-01-17, 2026-01-18)

## Context

The system ingests real-time trade data from Binance which, while suitable for traditional analytics (VWAP, OBI, Var-Covar in RTE), is not optimal for machine learning applications. Traditional time-based sampling creates issues:

- **Oversampling** during quiet periods (noise)
- **Undersampling** during volatile periods (missing information)
- **Non-IID returns**: Time-sampled returns exhibit serial correlation

A core objective is to implement **information-driven bars** as described in Marcos López de Prado's *Advances in Financial Machine Learning*, specifically:

- **Dollar Imbalance Bars (DIB)**: Sample when net buy/sell dollar pressure exceeds threshold
- **Dollar Runs Bars (DRB)**: Sample when persistent one-sided dollar flow exceeds threshold
- **Position Signals**: Generate trading positions based on bar characteristics

These features must:
- Process trades tick-by-tick in real-time
- Weight trades by economic significance (dollar value, not tick count)
- Adapt thresholds dynamically to changing market conditions
- Support per-symbol configuration for different liquidity profiles
- Generate position signals suitable for trading decisions

A decision is required on **where and how these ML features are computed**.

## Notation

| Acronym | Definition |
|---------|------------|
| AFML | Advances in Financial Machine Learning (López de Prado) |
| CTP | Chained Tickerplant (batched publisher) |
| DIB | Dollar Imbalance Bars |
| DRB | Dollar Runs Bars |
| EWMA | Exponential Weighted Moving Average |
| HDB | Historical Database |
| MLE | Machine Learning Engine |
| RTE | Real-Time Engine |
| SIG | Signal Generator (future) |
| TIB | Tick Imbalance Bars (not implemented) |
| TP | Tickerplant |
| VIB | Volume Imbalance Bars (not implemented) |
| WDB | Write-only Database (intraday writedown to HDB) |

## Decision

ML feature computation will be performed in a dedicated **Machine Learning Engine (MLE)** process using dollar-based information bars:

| Feature | Strategy | Update Path | Output |
|---------|----------|-------------|--------|
| Dollar Imbalance Bars | Cumulative signed dollar flow | Hot (every trade) | OHLCV bars |
| Dollar Runs Bars | Consecutive same-direction flow | Hot (every trade) | OHLCV bars |
| Position Signals | Buy/sell dollar percentage | On bar completion | Position (-1, 0, +1) |
| Adaptive Thresholds | EWMA of bar characteristics | On bar completion | Per-symbol thresholds |

### Why Dollar-Based (Not Tick or Volume)

| Bar Type | Unit | Cross-Asset Comparable? | Robust to Splits? | Crypto Suitable? |
|----------|------|-------------------------|-------------------|------------------|
| Tick Bars | # Trades | ❌ | ❌ | ❌ (tiny trades dominate) |
| Volume Bars | Shares/Coins | ❌ | ❌ | ⚠️ (ignores price level) |
| **Dollar Bars** | $ Exchanged | ✅ | ✅ | ✅ |

Dollar-based bars ensure:
- A $5 trade and a $500,000 trade are weighted appropriately
- Cross-asset comparison is meaningful (same $ threshold = same economic significance)
- Robust to stock splits, token redenominations, etc.

### Architectural Placement

```
TP:5010 ──┬──► WDB:5011 (storage → HDB)
          │
          ├──► MLE:5012 (tick-by-tick: DIB, DRB, Positions)
          │
          └──► CTP:5014 (batched 1s)
                    │
                    └──► RTE:5015 (dashboard: VWAP, vol, OBI)
```

- The **MLE** subscribes directly to the tickerplant for tick-by-tick trade updates
- The **MLE** is a peer subscriber alongside WDB (on the hot path)
- The **MLE** maintains its own state for bar computation and position signals
- **RTE** subscribes to Chained TP for batched dashboard analytics (different purpose)

### Connection Resilience

MLE implements resilient connection handling (see ADR-008):
- Exponential backoff on TP connection failure
- Timer-based reconnection attempts
- Process starts in degraded mode if TP unavailable
- Resumes normal operation when connection restored
- Reports connection state via `.health[]`

### Processing Mode

The MLE processes updates **tick-by-tick**:
- Each incoming trade updates the dollar imbalance accumulator
- Bar emission occurs when threshold is exceeded (not on timer)
- Position signals generated on bar completion
- Cleanup runs on timer (30s) for stale bars

---

## Bar Implementation

### 1. Dollar Imbalance Bars (DIB)

**Concept:** Sample a new bar when the cumulative *signed* dollar flow becomes sufficiently imbalanced, indicating informed trading activity.

**Tick Rule (Trade Direction):**
```
bt = +1  if price > prev_price (uptick → buy aggressor)
bt = -1  if price < prev_price (downtick → sell aggressor)
bt = bt-1 if price = prev_price (unchanged → use previous)
```

**Imbalance Accumulator:**
```
θ = Σ(bt × price × quantity)
```

Where `bt × price × quantity` is the signed dollar value of each trade.

**Bar Emission:**
When `|θ| ≥ threshold`, emit a bar and reset.

**Data Structure:**
```q
dib_bars:([]
    time:`timestamp$();           / Bar close time
    sym:`symbol$();               / Symbol
    open:`float$();               / Open price
    high:`float$();               / High price
    low:`float$();                / Low price
    close:`float$();              / Close price
    volume:`float$();             / Total volume (base currency)
    dollarVolume:`float$();       / Total dollar volume
    ticks:`long$();               / Number of trades in bar
    theta:`float$();              / Final imbalance (signed dollars)
    threshold:`float$();          / Threshold that triggered bar
    buyDollarPct:`float$();       / % of dollar volume from buys
    duration:`long$()             / Bar duration in milliseconds
);
```

**Example Output:**
| time | sym | ticks | dollarVolume | theta | buyDollarPct | duration |
|------|-----|-------|--------------|-------|--------------|----------|
| 23:57:24.567 | BTCUSDT | 421 | $105,484 | -$105,473 | 0.005% | 3010ms |
| 23:57:24.568 | BTCUSDT | 41 | $105,264 | -$105,264 | 0% | 1ms |
| 23:57:43.055 | BTCUSDT | 4 | $103,912 | +$103,912 | 100% | 0ms |

**Key Insight:** The 421-tick bar and 4-tick bar have similar dollar volumes but capture very different market events:
- 421 ticks over 3 seconds: Sustained sell pressure
- 4 ticks in <1ms: Whale buy (single large order)

This is exactly what information-driven sampling is designed to capture.

---

### 2. Dollar Runs Bars (DRB)

**Concept:** Sample when *consecutive* same-direction dollar flow exceeds threshold, capturing persistent accumulation or distribution.

**Run Tracking:**
```
if bt = +1:  buyRun += dollarValue;  sellRun = 0
if bt = -1:  sellRun += dollarValue; buyRun = 0
```

**Bar Emission:**
When `max(buyRun, sellRun) ≥ threshold`, emit a bar and reset.

**Difference from DIB:**

| Scenario | DIB θ | DRB maxRun | Interpretation |
|----------|-------|------------|----------------|
| Buy, Sell, Buy, Sell (each $25K) | 0 | $25K (resets each time) | Choppy, no signal |
| Buy, Buy, Buy, Buy (each $25K) | +$100K | $100K (accumulating) | Strong buy signal |

DRB specifically detects **persistent** one-sided flow, while DIB detects **net** imbalance.

**Data Structure:**
```q
drb_bars:([]
    time:`timestamp$();
    sym:`symbol$();
    open:`float$(); high:`float$(); low:`float$(); close:`float$();
    volume:`float$();
    dollarVolume:`float$();
    ticks:`long$();
    maxRun:`float$();             / Max run value that triggered
    runDirection:`short$();       / 1 = buy run, -1 = sell run
    threshold:`float$();
    duration:`long$()
);
```

---

### 3. Position Signals

**Concept:** Generate trading positions based on the buy/sell dollar percentage in completed DIB bars.

**Signal Logic:**
```q
.mle.cfg.signal.buyThreshold:65.0;    / Enter long if buyDollarPct > 65%
.mle.cfg.signal.sellThreshold:35.0;   / Enter short if buyDollarPct < 35%

/ On bar completion:
newPos:$[buyPct > buyThreshold; 1h;        / Long
         buyPct < sellThreshold; -1h;       / Short
         currentPos];                       / Hold
```

**Position State (per symbol):**
```q
.mle.position:`short$();          / Current position: -1, 0, or 1
.mle.positionTime:`timestamp$();  / Timestamp of last position change
.mle.positionPrice:`float$();     / Price at last position change
```

**Position History Table:**
```q
positions:([]
    time:`timestamp$();           / Position change time
    sym:`symbol$();               / Symbol
    position:`short$();           / New position: -1, 0, or 1
    price:`float$();              / Price at position change
    trigger:`float$();            / buyDollarPct that triggered change
    dollarVolume:`float$();       / Bar dollar volume
    ticks:`long$()                / Bar tick count
);
```

---

### 4. Adaptive Thresholds

Static thresholds don't work across:
- Different symbols (BTC vs SOL have different volumes)
- Different market regimes (quiet vs volatile periods)

**Solution:** EWMA-based adaptive thresholds that learn from actual bar characteristics.

**Algorithm:**
On each bar completion:
```q
actualImbalRatio = |θ| / dollarVolume
ewmaThreshold = α × (dollarVolume × actualImbalRatio) + (1-α) × ewmaThreshold
ewmaImbalRatio = α × actualImbalRatio + (1-α) × ewmaImbalRatio
```

**Configuration:**
```q
.mle.cfg.alpha.threshold:0.02;    / Slow adaptation (stable)
.mle.cfg.alpha.imbalance:0.05;    / Medium adaptation

/ Per-symbol initial thresholds
.mle.cfg.init.threshold:`BTCUSDT`ETHUSDT`SOLUSDT!100000 75000 40000f;
.mle.cfg.init.thresholdDefault:50000.0;
```

**Rationale for Per-Symbol Thresholds:**
| Symbol | Daily $ Volume | Initial Threshold | Expected Bars/Day |
|--------|----------------|-------------------|-------------------|
| BTCUSDT | ~$10B | $100K | ~100 |
| ETHUSDT | ~$5B | $75K | ~100 |
| SOLUSDT | ~$500M | $40K | ~100 |

Rule of thumb: threshold ≈ 0.1% of daily dollar volume for ~100 bars/day.

---

## State Schema Summary

| State | Type | Memory | Update | Cleanup |
|-------|------|--------|--------|---------|
| dib_bars | Table | ~1MB/day | On threshold breach | Timer (30s) |
| drb_bars | Table | ~500KB/day | On threshold breach | Timer (30s) |
| positions | Table | ~100KB/day | On position change | Timer (30s) |
| .mle.theta | Vector | ~100 bytes | Every trade | On bar emit |
| .mle.ewmaThreshold | Vector | ~100 bytes | On bar emit | Never (persists) |
| .mle.buyRun/.sellRun | Vector | ~100 bytes | Every trade | On bar emit |
| .mle.position | Vector | ~100 bytes | On bar emit | Never |

**Retention:**
```q
.mle.cfg.barRetentionMin:1440;     / Keep 24 hours of bars
```

---

## Query Interface Summary

**Status & Monitoring:**
```q
.health[]                         / Standardized health check (incl. connection state)
.mle.status[]                     / Current accumulator state + positions
.mle.thresholds[]                 / Current adaptive thresholds
.mle.resetThreshold[`SOLUSDT]     / Reset symbol to initial threshold
.mle.resetAllThresholds[]         / Reset all to initial thresholds
```

**Bar Retrieval:**
```q
.mle.getDIB[`BTCUSDT; 10]         / Last 10 DIB bars for symbol
.mle.getDIB[`; 20]                / Last 20 DIB bars (all symbols)
.mle.getDRB[`BTCUSDT; 10]         / Last 10 DRB bars for symbol
.mle.barStats[]                   / Bar statistics by symbol
```

**Position Interface:**
```q
.mle.getPosition[`BTCUSDT]        / Current position (-1/0/1)
.mle.getPositions[]               / All current positions
.mle.getPositionHistory[`BTCUSDT; 10]  / Last 10 position changes
.mle.getPositionHistory[`; 20]    / All symbols
.mle.positionStats[]              / Position change statistics
```

**Example Output - .mle.getPositions[]:**
```
sym     position price    time
---------------------------------------------
BTCUSDT 1        95448.21 2026.01.18D10:23:45
ETHUSDT -1       3291.44  2026.01.18D10:22:31
SOLUSDT 0        144.39   2026.01.18D09:15:22
```

---

## Timer Functions

The MLE timer runs every 30 seconds:

```q
.z.ts:{[]
    .mle.cleanup[];       / Remove bars older than 24 hours
};
```

---

## Recovery Strategy

On MLE restart:

| Step | Action |
|------|--------|
| 1 | Attempt connection to TP (with exponential backoff if unavailable) |
| 2 | Replay today's TP log file (if exists) |
| 3 | Run cleanup to remove stale bars |
| 4 | Subscribe to tickerplant for live updates |
| 5 | Begin accumulating new bars |
| 6 | Adaptive thresholds restored from replay |

**Warm-up Times:**
- DIB/DRB Bars: Immediate (first threshold breach)
- Adaptive Thresholds: ~10 bars for stabilization
- Positions: Immediate (computed on bar completion)

---

## Performance Characteristics

### Hot Path (Per-Trade)

| Operation | Latency | Notes |
|-----------|---------|-------|
| Symbol lookup | <100ns | O(1) hash lookup |
| Tick rule | <100ns | Price comparison |
| Theta accumulation | <100ns | Array addition |
| Run tracking | <100ns | Conditional add/reset |
| Threshold check | <100ns | Array comparison |
| Bar emission | <10μs | Table insert |
| Position check | <1μs | Threshold comparison |

### Cold Path (Timer)

| Operation | Latency | Notes |
|-----------|---------|-------|
| Cleanup | <10ms | Delete from tables |

### Memory (3 symbols, 24 hours)

| Component | Memory | Notes |
|-----------|--------|-------|
| dib_bars | ~1MB | ~300 bars/symbol × 13 columns |
| drb_bars | ~500KB | ~150 bars/symbol × 13 columns |
| positions | ~100KB | ~50 changes/symbol × 7 columns |
| State vectors | ~1KB | Accumulators, thresholds |
| **Total** | **~2MB** | Scales linearly with symbols |

---

## Rationale

### Dollar-Based Over Tick/Volume:
- **Cross-asset comparable**: Same $ threshold works across price levels
- **Crypto-suitable**: Tiny bot trades don't dominate
- **Economically meaningful**: Weights by actual capital flow

### DIB + DRB (Not Just DIB):
- **Different signals**: DIB = net pressure, DRB = persistence
- **Complementary**: Choppy market triggers DRB differently than DIB
- **Low marginal cost**: Same hot path, minimal extra state

### Adaptive Thresholds:
- **Market regime adaptation**: Quiet vs volatile periods
- **Per-symbol tuning**: Automatic, not manual
- **EWMA stability**: Slow adaptation prevents oscillation

### Position Signals:
- **Simple rules**: Buy/sell percentage thresholds
- **Immediate**: Computed on bar completion
- **Per-symbol**: Independent positions per instrument

### Separate Process (Not in RTE):
- **Different consumers**: RTE serves dashboards, MLE serves trading
- **Different lifecycles**: MLE config changes more frequently
- **Clean separation**: Analytics vs signal generation

---

## Alternatives Considered

### 1. Tick Imbalance Bars (TIB)
Rejected:
- Tiny trades dominate in crypto
- Not cross-asset comparable
- López de Prado recommends dollar-based for most applications

### 2. Volume Imbalance Bars (VIB)
Rejected:
- Ignores price level (1 BTC at $100K ≠ 1 BTC at $10K)
- Dollar bars capture economic significance better

### 3. Compute bars in RTE
Rejected:
- Different update frequencies (RTE has batched subscription)
- Different consumers (dashboards vs trading)
- Cleaner to separate concerns

### 4. Fixed thresholds only
Rejected:
- Doesn't adapt to market regime changes
- Requires manual tuning per symbol
- EWMA provides automatic adaptation

### 5. Triple Barrier Labeling
Deferred:
- Labels require future data (not suitable for real-time signals)
- Can be added for historical backtest
- Position signals provide immediate actionable output

---

## Consequences

### Positive

- Information-driven sampling (not time-based)
- Dollar-weighted for economic significance
- Cross-asset comparable thresholds
- Adaptive to market regime
- Per-symbol configuration
- Position signals for trading decisions
- Low latency hot path (<1μs)
- Connection resilience with exponential backoff
- Standardized `.health[]` for monitoring

### Negative / Trade-offs

- Adaptive thresholds need ~10 bars to stabilize
- Replay from TP logs required for state recovery
- Position signals are simple (no ML model yet)

---

## Future Evolution

| Enhancement | Trigger |
|-------------|---------|
| Position persistence | Save/load positions across restart |
| Signal latency metrics | Track time from trade to position change |
| Triple Barrier Labeling | Historical backtest requirement |
| Meta-labeling | Position sizing based on model confidence |
| Fractional differentiation | Stationarity while preserving memory |
| Cross-asset events | Detect correlated liquidation cascades |

---

## Links / References

- López de Prado, M. (2018). *Advances in Financial Machine Learning*. Wiley.
  - Chapter 2: Information-Driven Bars
  - Chapter 3: Labeling (Triple Barrier Method)
  - Chapter 4: Sample Weights
- `adr-002-feed-handler-to-kdb-ingestion-path.md` (tick-by-tick precedent)
- `adr-004-Real-Time-Analytics-Computation.md` (RTE architecture, peer pattern)
- `adr-008-error-handling-strategy.md` (connection resilience)
- `rte.q` (peer process for analytics)
- `mle.q` (implementation)
