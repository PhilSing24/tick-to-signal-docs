# ADR-011: Financial Machine Learning Engine

## Status
Accepted

## Date
2026-01-15

## Context

The system ingests real-time trade data from Binance which, while suitable for traditional analytics (VWAP, OBI, Var-Covar in RTE), is not optimal for machine learning applications. Traditional time-based sampling creates issues:

- **Oversampling** during quiet periods (noise)
- **Undersampling** during volatile periods (missing information)
- **Non-IID returns**: Time-sampled returns exhibit serial correlation

A core objective is to implement **information-driven bars** as described in Marcos López de Prado's *Advances in Financial Machine Learning*, specifically:

- **Dollar Imbalance Bars (DIB)**: Sample when net buy/sell dollar pressure exceeds threshold
- **Dollar Runs Bars (DRB)**: Sample when persistent one-sided dollar flow exceeds threshold
- **Triple Barrier Labeling**: Label bars for supervised ML based on future price movement

These features must:
- Process trades tick-by-tick in real-time
- Weight trades by economic significance (dollar value, not tick count)
- Adapt thresholds dynamically to changing market conditions
- Support per-symbol configuration for different liquidity profiles
- Generate labels suitable for ML classification tasks

A decision is required on **where and how these ML features are computed**.

## Notation

| Acronym | Definition |
|---------|------------|
| AFML | Advances in Financial Machine Learning (López de Prado) |
| DIB | Dollar Imbalance Bars |
| DRB | Dollar Runs Bars |
| EWMA | Exponential Weighted Moving Average |
| HDB | Historical Database |
| MLE | Machine Learning Engine |
| RDB | Real-Time Database |
| TIB | Tick Imbalance Bars (not implemented) |
| TP | Tickerplant |
| VIB | Volume Imbalance Bars (not implemented) |

## Decision

ML feature computation will be performed in a dedicated **Machine Learning Engine (MLE)** process using dollar-based information bars:

| Feature | Strategy | Update Path | Output |
|---------|----------|-------------|--------|
| Dollar Imbalance Bars | Cumulative signed dollar flow | Hot (every trade) | OHLCV bars |
| Dollar Runs Bars | Consecutive same-direction flow | Hot (every trade) | OHLCV bars |
| Triple Barrier Labels | Forward-looking price targets | Cold (timer, 30s) | Classification labels |
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
TP ──┬──► RDB : 5011 (storage: trade_binance, quote_binance)
     │
     ├──► RTE : 5012 (analytics: VWAP, Imbalance, VarCovar)
     │
     └──► MLE : 5015 (ML features: DIB, DRB, Labels)
```

- The **MLE** subscribes directly to the tickerplant for live updates
- The **MLE** is a peer subscriber alongside RDB and RTE
- The **MLE** maintains its own state for bar computation and labeling

### Processing Mode

The MLE processes updates **tick-by-tick**:
- Each incoming trade updates the dollar imbalance accumulator
- Bar emission occurs when threshold is exceeded (not on timer)
- Labeling runs on timer (30s) for matured bars

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

### 3. Adaptive Thresholds

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

## Triple Barrier Labeling

### Concept

For supervised ML, we need labels. The triple barrier method labels each bar based on which price barrier is hit first:

```
         Upper Barrier (+profit target)
              ─────────────────────
                    ╱
Entry Price ───────●
                    ╲
              ─────────────────────
         Lower Barrier (-stop loss)
         
         |←── Vertical Barrier (time limit) ──→|
```

| Barrier Hit | Label | Meaning |
|-------------|-------|---------|
| Upper | +1 | Price went up → profitable long |
| Lower | -1 | Price went down → profitable short |
| Vertical | 0 | No clear direction → no trade |

### Implementation

**Configuration:**
```q
.mle.cfg.label.upperPct:0.5;       / +0.5% take profit (fixed)
.mle.cfg.label.lowerPct:0.5;       / -0.5% stop loss (fixed)
.mle.cfg.label.horizonBars:10;     / Max 10 bars forward
.mle.cfg.label.horizonMs:300000;   / Or 5 minutes max
.mle.cfg.label.useVolAdjust:1b;    / Use volatility-adjusted barriers
.mle.cfg.label.volLookback:20;     / Bars for volatility calculation
.mle.cfg.label.volMultiplier:2.0;  / Barrier = 2 × rolling volatility
```

**Volatility-Adjusted Barriers:**
Instead of fixed percentages, barriers scale with recent volatility:
```q
volatility = stddev(log returns of last 20 bars)
upperPrice = entryPrice × (1 + volMultiplier × volatility)
lowerPrice = entryPrice × (1 - volMultiplier × volatility)
```

**Data Structure:**
```q
labeled_bars:([]
    time:`timestamp$();           / Entry time
    sym:`symbol$();
    close:`float$();              / Entry price
    dollarVolume:`float$();       / Bar features...
    ticks:`long$();
    theta:`float$();
    buyDollarPct:`float$();
    duration:`long$();
    / Label columns
    label:`short$();              / -1, 0, +1
    ret:`float$();                / Actual return at exit
    exitTime:`timestamp$();
    exitPrice:`float$();
    barrierHit:`symbol$();        / `upper`lower`vertical
    upperBarrier:`float$();
    lowerBarrier:`float$();
    volatility:`float$()
);
```

**Labeling Algorithm:**
1. For each DIB bar older than horizon time
2. Look forward through subsequent DIB bars
3. Check if high >= upperBarrier (upper hit)
4. Check if low <= lowerBarrier (lower hit)
5. If neither within horizon → vertical hit
6. Assign label based on which barrier hit first

**Timer Execution:**
Labels are computed on a 30-second timer (cold path) since they require future data.

---

## State Schema Summary

| State | Type | Memory | Update | Cleanup |
|-------|------|--------|--------|---------|
| dib_bars | Table | ~1MB/day | On threshold breach | Timer (30s) |
| drb_bars | Table | ~500KB/day | On threshold breach | Timer (30s) |
| labeled_bars | Table | ~1MB/day | Timer (30s) | Timer (30s) |
| .mle.theta | Dictionary | ~100 bytes | Every trade | On bar emit |
| .mle.ewmaThreshold | Dictionary | ~100 bytes | On bar emit | Never (persists) |
| .mle.buyRun/.sellRun | Dictionary | ~100 bytes | Every trade | On bar emit |

**Retention:**
```q
.mle.cfg.barRetentionMin:1440;     / Keep 24 hours of bars
```

---

## Query Interface Summary

**Status & Monitoring:**
```q
.mle.status[]                     / Current accumulator state
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

**Labeling:**
```q
.mle.labelStats[]                 / Label distribution and win rates
.mle.getLabeled[`BTCUSDT; 10]     / Last 10 labeled bars
.mle.getLabeled[`; 20]            / Last 20 labeled bars (all symbols)
```

**Example Output - .mle.labelStats[]:**
```
sym    | bars longWins shortWins noMove winRate  avgRet
-------| ------------------------------------------------
BTCUSDT| 134  67       43        10     55.8%    0.009%
ETHUSDT| 89   21       12        48     25.9%   -0.015%
SOLUSDT| 22   8        14        0      36.4%   -0.008%
```

---

## Timer Functions

The MLE timer runs every 30 seconds:

```q
.z.ts:{[]
    .mle.cleanup[];       / Remove bars older than 24 hours
    .mle.labelBars[];     / Label matured DIB bars
};
```

---

## Recovery Strategy

On MLE restart:

| Step | Action |
|------|--------|
| 1 | Connect and subscribe to tickerplant |
| 2 | Begin accumulating new bars |
| 3 | Adaptive thresholds retained (learned values persist EOD) |
| 4 | Labels become valid after horizon time (5 min) |

**Note:** Unlike RTE, MLE does not replay TP logs. Bars are regenerated from live data. For historical analysis, bars should be computed from HDB tick data.

**Warm-up Times:**
- DIB/DRB Bars: Immediate (first threshold breach)
- Adaptive Thresholds: ~10 bars for stabilization
- Labels: horizon time + vol lookback (5 min + 20 bars)

---

## Performance Characteristics

### Hot Path (Per-Trade)

| Operation | Latency | Notes |
|-----------|---------|-------|
| Tick rule | <100ns | Price comparison |
| Theta accumulation | <100ns | Addition |
| Run tracking | <100ns | Conditional add/reset |
| Threshold check | <100ns | Comparison |
| Bar emission | <10μs | Table insert |

### Cold Path (Timer)

| Operation | Latency | Notes |
|-----------|---------|-------|
| Label single bar | <1ms | Forward scan through bars |
| Label all matured | <100ms | Batch processing |
| Cleanup | <10ms | Delete from tables |

### Memory (3 symbols, 24 hours)

| Component | Memory | Notes |
|-----------|--------|-------|
| dib_bars | ~1MB | ~300 bars/symbol × 13 columns |
| drb_bars | ~500KB | ~150 bars/symbol × 13 columns |
| labeled_bars | ~1MB | ~300 bars/symbol × 16 columns |
| State dictionaries | ~1KB | Accumulators, thresholds |
| **Total** | **~3MB** | Scales linearly with symbols |

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

### Volatility-Adjusted Labels:
- **Regime-aware**: Barriers scale with market conditions
- **Comparable labels**: Same "difficulty" across periods
- **Better ML signal**: Labels reflect meaningful moves, not noise

### Separate Process (Not in RTE):
- **Different consumers**: RTE serves dashboards, MLE serves ML pipelines
- **Different lifecycles**: MLE config changes more frequently
- **Clean separation**: Analytics vs feature engineering

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
- Different update frequencies (RTE has 5s timer for vcov)
- Different consumers (dashboards vs ML)
- Cleaner to separate concerns

### 4. Fixed thresholds only
Rejected:
- Doesn't adapt to market regime changes
- Requires manual tuning per symbol
- EWMA provides automatic adaptation

### 5. Time-based labeling horizon only
Rejected:
- Bar-based horizon captures "market time"
- Combined approach (bars AND time) is more robust
- Aligns with López de Prado methodology

### 6. Store DIB/DRB in HDB
Rejected:
- Parameters change during research
- Labels are path-dependent (future data)
- Better to compute from raw ticks on demand
- Raw tick data is source of truth

---

## Consequences

### Positive

- Information-driven sampling (not time-based)
- Dollar-weighted for economic significance
- Cross-asset comparable thresholds
- Adaptive to market regime
- Per-symbol configuration
- Triple barrier labels for supervised ML
- Volatility-adjusted barriers
- Low latency hot path (<1μs)
- Production-grade implementation

### Negative / Trade-offs

- Labels require warm-up (horizon time + vol lookback)
- Adaptive thresholds need ~10 bars to stabilize
- No replay from TP logs (bars regenerated live)
- Volatility lookback requires sufficient bar history

---

## Future Evolution

| Enhancement | Trigger |
|-------------|---------|
| Meta-labeling | Position sizing based on model confidence |
| Fractional differentiation | Stationarity while preserving memory |
| Sample weights | Uniqueness-based weighting for overlapping labels |
| Cross-asset events | Detect correlated liquidation cascades |
| HDB integration | Batch computation for historical backtest |
| Feature engineering | Structural breaks, entropy features |

---

## Links / References

- López de Prado, M. (2018). *Advances in Financial Machine Learning*. Wiley.
  - Chapter 2: Information-Driven Bars
  - Chapter 3: Labeling (Triple Barrier Method)
  - Chapter 4: Sample Weights
- `adr-002-feed-handler-to-kdb-ingestion-path.md` (tick-by-tick precedent)
- `adr-004-Real-Time-Analytics-Computation.md` (RTE architecture, peer pattern)
- `rte.q` (peer process for analytics)
- `mle.q` (implementation)
