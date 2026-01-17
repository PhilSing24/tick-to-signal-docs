# ADR-004: Real-Time Analytics Computation

## Status
Accepted (Updated 2026-01-17)

## Date
Original: 2025-12-17
Updated: 2026-01-11, 2026-01-16, 2026-01-17

## Context

The system ingests real-time market data from Binance via two C++ feed handlers:
- **Trade Feed Handler**: Individual trade executions
- **Quote Feed Handler**: L2 order book depth updates

A core objective of the project is to compute **real-time analytics** on this live data for dashboard display:
- **VWAP (Volume-Weighted Average Price)** from trades
- **Daily Volatility** from trades (annualized)
- **Order Book** from L2 quotes (latest snapshot)
- **Order Book Imbalance** from L2 quotes (with EMA smoothing)

These analytics must:
- Update as batched data arrives (1-second batches)
- Be query-consistent
- Reflect recent data with acceptable latency for dashboard display
- Scale efficiently with message volume

A decision is required on **where and how these analytics are computed**.

**Note:** RTE subscribes to Chained TP (batched 1s) since it serves dashboard display, not latency-sensitive trading decisions.

## Notation

| Acronym | Definition |
|---------|------------|
| CTP | Chained Tickerplant (batched publisher) |
| EMA | Exponential Moving Average |
| IPC | Inter-Process Communication |
| L2 | Level 2 (top 5 bid/ask levels) |
| MLE | Machine Learning Engine |
| OBI | Order Book Imbalance |
| RDB | Real-time Database (query-only) |
| RTE | Real-Time Engine |
| TP | Tickerplant |
| VWAP | Volume-Weighted Average Price |
| WDB | Write-only Database (intraday writedown to HDB) |

## Decision

Real-time analytics will be computed in a **Real-Time Engine (RTE)** process using distinct strategies per metric:

| Metric | Strategy | Description | Update Path |
|--------|----------|-------------|-------------|
| VWAP | Cumulative daily | Sum(price×qty) / Sum(qty) | Batched (1s) |
| Daily Vol | Rolling 30-sec returns | Annualized stddev of log returns | Batched (1s) |
| Order Book | Latest snapshot | L5 bid/ask prices and quantities | Batched (1s) |
| Smooth OBI | EMA of raw OBI | Smoothed order book imbalance | Batched (1s) |

### Architectural Placement

```
TP:5010 ──┬──► WDB:5011 (tick-by-tick → HDB)
          │
          ├──► MLE:5012 (tick-by-tick, trades only)
          │
          └──► CTP:5014 (batched 1s)
                    │
                    ├──► RTE:5015 (VWAP, vol, OBI)
                    ├──► TEL:5016 (FH latency)
                    └──► RDB:5017 (user queries)
```

### Subscription Model

- RTE subscribes to **Chained TP** (port 5014) for batched updates
- Batched delivery is acceptable for dashboard display (1-second delay)
- Reduces load on primary TP hot path

### Processing Mode

The RTE processes updates in **batches**:
- Receives batched data every 1 second from Chained TP
- Updates analytics state for each row in the batch
- No individual tick latency requirements (dashboard use case)

---

## Analytics Implementation

### 1. VWAP (Volume-Weighted Average Price)

**Formula:**
```
VWAP = Σ(price × quantity) / Σ(quantity)
```

**Data Structure:**
```q
/ sym -> (sumPxQty; sumQty; lastPrice; tradeCount)
.rte.st.vwap:()!();
```

**Update Algorithm:**
```q
.rte.updVwap:{[s;price;qty]
  pxqty:price * qty;
  $[s in key .rte.st.vwap;
    .rte.st.vwap[s]+:(pxqty; qty; 0f; 1j);
    .rte.st.vwap[s]:(pxqty; qty; price; 1j)
  ];
  .rte.st.vwap[s;2]:price;  / Update last price
  };
```

**Query Interface:**
```q
.rte.getVwap[]

/ Returns:
sym     vwap     lastPrice volume   trades
------------------------------------------
BTCUSDT 95448.8  95448.21  6.5112   2397
SOLUSDT 144.40   144.39    5602.73  3132
ETHUSDT 3291.44  3291.51   115.55   3738
```

---

### 2. Daily Volatility

**Formula:**
```
Vol = sqrt(periods_per_year) × stddev(log_returns)
```

**Configuration:**
```q
.rte.cfg.volWindowSec:30;       / Return window (30 sec between prices)
.rte.cfg.volMinReturns:30;      / Minimum returns before displaying vol
```

**Data Structures:**
```q
.rte.st.volPrices:()!();   / sym -> list of (time; price)
.rte.st.volReturns:()!();  / sym -> list of log returns
.rte.st.volLatest:()!();   / sym -> (annualizedVol; returnCount; isValid)
```

**Update Algorithm:**
1. Store price point with timestamp
2. Find price from ~30 seconds ago
3. Calculate log return: `log(price / oldPrice)`
4. Store return
5. After 30+ returns, calculate annualized volatility

**Query Interface:**
```q
.rte.getVol[]

/ Returns:
sym     annualizedVol returnCount isValid
-----------------------------------------
BTCUSDT 45.2          156         1
SOLUSDT 72.1          203         1
ETHUSDT 58.4          189         1
```

**Validity Logic:**
- `isValid = 1` if `returnCount >= 30` (sufficient samples)
- Takes ~15 minutes to become valid (30 returns × 30 sec)

**Edge Case Guards:**
- Skip if old price is null or <= 0
- Skip if log return is null or infinite
- Only update volatility if valid and > 0

---

### 3. Order Book (L5 Snapshot)

**Data Structure:**
```q
/ sym -> (bidPrice1-5; bidQty1-5; askPrice1-5; askQty1-5; time)
.rte.st.book:()!();
```

**Update Algorithm:**
```q
.rte.updBook:{[s;data;time]
  / data indices: 2-6 bidPrice, 7-11 bidQty, 12-16 askPrice, 17-21 askQty
  .rte.st.book[s]:data[2 3 4 5 6 7 8 9 10 11 12 13 14 15 16 17 18 19 20 21],time;
  };
```

**Query Interface:**
```q
.rte.getOrderBook[`BTCUSDT]

/ Returns:
level bidPrice  bidQty    askPrice  askQty
------------------------------------------
1     95448.00  0.523     95448.01  0.412
2     95447.99  1.234     95448.02  0.891
3     95447.98  0.876     95448.03  1.023
4     95447.97  2.145     95448.04  0.567
5     95447.96  1.567     95448.05  0.789

.rte.getSpread[`BTCUSDT]

/ Returns:
sym     bid      ask      spread mid
---------------------------------------
BTCUSDT 95448.00 95448.01 0.01   95448.005
```

---

### 4. Order Book Imbalance (OBI)

**Formula:**
```
Raw OBI = (bidDepth - askDepth) / (bidDepth + askDepth)
Smooth OBI = alpha × Raw OBI + (1 - alpha) × Previous Smooth OBI
```

**Configuration:**
```q
.rte.cfg.obiAlpha:0.05;   / EMA smoothing (lower = smoother)
```

**Data Structure:**
```q
/ sym -> (rawOBI; smoothOBI; time)
.rte.st.obi:()!();
```

**Update Algorithm:**
```q
.rte.updOBI:{[s;bidDepth;askDepth;time]
  total:bidDepth + askDepth;
  rawOBI:$[total > 0f; (bidDepth - askDepth) % total; 0n];
  prevSmooth:$[s in key .rte.st.obi; .rte.st.obi[s;1]; rawOBI];
  smoothOBI:(.rte.cfg.obiAlpha * rawOBI) + ((1 - .rte.cfg.obiAlpha) * prevSmooth);
  .rte.st.obi[s]:(rawOBI; smoothOBI; time);
  };
```

**Query Interface:**
```q
.rte.getOBI[`smooth]    / Smooth OBI only
.rte.getOBI[`raw]       / Raw OBI only
.rte.getOBIAll[]        / Both

/ Returns:
sym     rawOBI  smoothOBI
-------------------------
BTCUSDT 0.123   0.087
SOLUSDT -0.045  -0.032
ETHUSDT 0.234   0.156
```

**Interpretation:**
- OBI > 0: More bid depth (buying pressure)
- OBI < 0: More ask depth (selling pressure)
- Smooth OBI filters noise for trading signals

---

## State Schema Summary

| State | Type | Memory | Update |
|-------|------|--------|--------|
| .rte.st.vwap | Dictionary | ~300 bytes | O(1) |
| .rte.st.volPrices | Dictionary | ~50KB (10min) | O(1) |
| .rte.st.volReturns | Dictionary | ~10KB | O(1) |
| .rte.st.volLatest | Dictionary | ~100 bytes | O(1) |
| .rte.st.book | Dictionary | ~500 bytes | O(1) |
| .rte.st.obi | Dictionary | ~100 bytes | O(1) |

**Total:** ~60KB (scales linearly with symbols)

---

## Query Interface Summary

```q
.health[]                / Standardized health check (process, port, uptime, status, memory)
.rte.getVwap[]           / VWAP all symbols
.rte.getVol[]            / Daily vol (annualized %)
.rte.getOrderBook[`SYM]  / L5 order book
.rte.getSpread[`SYM]     / Bid/ask/spread/mid
.rte.getOBI[`smooth]     / Smooth OBI
.rte.getOBI[`raw]        / Raw OBI
.rte.getOBIAll[]         / Both raw and smooth OBI
.rte.replay[.z.D]        / Replay today's log
```

---

## Recovery Strategy

On RTE restart:

| Step | Action |
|------|--------|
| 1 | Replay today's TP log file (if exists) |
| 2 | Connect and subscribe to Chained TP for batched updates |
| 3 | Analytics become valid as data accumulates |

**Warm-up Times:**
- VWAP: Instant (first trade makes it valid)
- Daily Vol: ~15 minutes (need 30 returns)
- Order Book: Instant (first quote makes it valid)
- OBI: Instant (first quote makes it valid)

---

## Rationale

### Simplified Analytics:
- Dashboard use case doesn't need complex var-covar
- VWAP, Vol, Order Book, OBI cover core needs
- Simpler implementation, easier to maintain

### Batched Subscription (via Chained TP):
- 5-second delay acceptable for dashboard display
- Reduces load on primary TP hot path
- Hot path reserved for WDB (persistence) and MLE (signal generation)

### Daily VWAP:
- Cumulative is simpler than windowed
- Resets at EOD
- Standard market metric

### 30-Second Return Window for Vol:
- Short enough to capture intraday volatility
- Long enough to avoid noise
- Standard approach for HF volatility estimation

### EMA Smoothing for OBI:
- Raw OBI too noisy for trading signals
- Alpha=0.05 provides good smoothing
- Both raw and smooth available

---

## Alternatives Considered

### 1. Tick-by-tick subscription (from primary TP)
Rejected:
- Adds load to primary TP
- Not needed for dashboard use case
- Batched is sufficient

### 2. Var-covar matrix
Removed (2026-01-17):
- Complex implementation
- Not needed for current dashboard
- Can be added later if needed

### 3. Windowed VWAP (rolling 5-min, etc.)
Simplified:
- Daily cumulative is simpler
- Meets current requirements
- Can add windowed later if needed

### 4. Real-time volatility (per-trade)
Rejected:
- Too noisy
- 30-second returns are more stable
- Batched updates are sufficient

---

## Consequences

### Positive

- Simple, focused analytics
- Low memory footprint (~60KB)
- Fast updates (O(1) per row)
- Clear query interface
- Batched subscription reduces primary TP load
- Dashboard-ready output

### Negative / Trade-offs

- 1-second delay from batching
- Daily VWAP resets at EOD (no rolling window)
- Vol needs warm-up (~15 minutes)
- No var-covar matrix (simplified out)

---

## Future Evolution

| Enhancement | Trigger |
|-------------|---------|
| Rolling VWAP windows | Trading strategy needs |
| Var-covar matrix | Portfolio risk dashboard |
| More symbols | Scale to 100+ instruments |
| Tick-by-tick mode | Latency-sensitive use case |

---

## Links / References

- `adr-001-timestamps-and-latency-measurement.md` (latency targets)
- `adr-002-feed-handler-to-kdb-ingestion-path.md` (data flow)
- `adr-005-telemetry-and-metrics-aggregation-strategy.md` (TEL handles latency)
- `adr-007-visualisation-and-consumption-strategy.md` (dashboard consumption)
- `adr-009-Order-Book-Architecture.md` (quote feed handler details)
