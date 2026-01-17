# ADR-007: Visualisation and Consumption Strategy

## Status
Accepted (Updated 2026-01-18)

## Date
2025-12-17 (Updated 2025-12-19, 2026-01-14, 2026-01-16, 2026-01-17, 2026-01-18)

## Context

The system produces:
- Real-time market data (Binance trades and quotes)
- Derived analytics (VWAP, daily vol, order book imbalance)
- ML features (DIB, DRB bars, labels)
- Telemetry metrics (FH latency)

These outputs must be:
- Observable by humans during development and operation
- Suitable for validating system behaviour
- Easy to iterate on during exploration

A decision is required on how system outputs are visualised and consumed.

## Notation

| Acronym | Definition |
|---------|------------|
| CTP | Chained Tickerplant (batched publisher) |
| FH | Feed Handler |
| IPC | Inter-Process Communication |
| MLE | Machine Learning Engine |
| RDB | Real-time Database (query-only) |
| RTE | Real-Time Engine |
| TEL | Telemetry Process |
| TP | Tickerplant |
| WDB | Write-only Database (intraday writedown to HDB) |

## Decision

The project will use **KX Dashboards** as the primary visualisation mechanism, with **polling-based queries** at **1-second intervals**.

### Visualisation Technology

**KX Dashboards** is selected because:
- Native to the KDB-X ecosystem
- Requires minimal integration effort
- Supports time-series and real-time analytics well
- Well-suited for exploratory and diagnostic workflows
- No additional infrastructure required

### Data Delivery Model

**Polling (pull-based)** is selected over streaming (push-based):

| Factor | Polling | Streaming |
|--------|---------|-----------|
| Implementation complexity | Simple | Complex |
| Server-side requirements | None | Subscription management |
| Recovery handling | Automatic (next poll) | Must handle reconnect |
| Update granularity | Poll interval | Per-event possible |

### Query Targets

Dashboards query **RDB, RTE, MLE, and TEL**:

```
              +──► RDB:5017 (raw trades, quotes)
              │
Dashboard ────+──► RTE:5015 (VWAP, vol, OBI, order book)
              │
              +──► MLE:5012 (DIB, DRB bars, labels, positions)
              │
              +──► TEL:5016 (FH latency, FH health)
```

| Data | Source | Port | Query |
|------|--------|------|-------|
| Last trade price | RDB | 5017 | `trade_binance` |
| Raw trade history | RDB | 5017 | `trade_binance` |
| VWAP | RTE | 5015 | `.rte.getVwap[]` |
| Daily volatility | RTE | 5015 | `.rte.getVol[]` |
| L5 order book | RTE | 5015 | `.rte.getOrderBook[]` |
| Order book imbalance | RTE | 5015 | `.rte.getOBIAll[]` |
| DIB bars | MLE | 5012 | `.mle.getDIB[]` |
| DRB bars | MLE | 5012 | `.mle.getDRB[]` |
| ML label stats | MLE | 5012 | `.mle.labelStats[]` |
| Current positions | MLE | 5012 | `.mle.getPositions[]` |
| FH latency metrics | TEL | 5016 | `.tel.vsFhLatency[]` |
| FH health status | TEL | 5016 | `.tel.vsFhStatus[]` |
| Process health | Any | Various | `.health[]` |

### Dashboard Specification

**Dashboard 1: Market Overview**

| Panel | Data Source | Content |
|-------|-------------|---------|
| Last Price | RDB | Most recent trade price per symbol |
| VWAP | RTE | `.rte.getVwap[]` |
| Daily Vol | RTE | `.rte.getVol[]` |
| Order Book | RTE | `.rte.getOrderBook[]` |
| OBI Indicator | RTE | `.rte.getOBIAll[]` |

**Dashboard 2: System Health**

| Panel | Data Source | Content |
|-------|-------------|---------|
| FH Health Grid | TEL | `.tel.vsFhStatus[]` |
| FH Latency | TEL | `.tel.vsFhLatency[]` |
| Trade Volume | RDB | Count per symbol |
| Quote Volume | RDB | Count per symbol |
| Process Health | All | `.health[]` from each process |

**Dashboard 3: ML Features**

| Panel | Data Source | Content |
|-------|-------------|---------|
| DIB Bar Summary | MLE | `.mle.barStats[]` |
| DRB Bar Summary | MLE | `.mle.getDRB[]` count |
| Label Distribution | MLE | `.mle.labelStats[]` |
| Adaptive Thresholds | MLE | `.mle.thresholds[]` |
| Current Positions | MLE | `.mle.getPositions[]` |

### Update Latency Expectations

| Metric | Source | Target Latency (data age) |
|--------|--------|---------------------------|
| Last price | RDB | < 3 seconds (1s batch + 2s query) |
| VWAP | RTE | < 3 seconds |
| Telemetry | TEL | < 7 seconds (1s batch + 5s bucket + 1s query) |

These targets account for:
- Chained TP batch interval (1 second)
- Poll interval (up to 1 second)
- Query execution time (< 100ms)
- TEL aggregation bucket (5 seconds)

### Symbol Scope

Initial dashboards display data for:
- BTCUSDT
- ETHUSDT
- SOLUSDT

## Rationale

KX Dashboards with polling was selected because:

- **Native integration** - No additional tooling required
- **Rapid development** - Pre-built components
- **Sufficient for purpose** - Human-scale observation doesn't require streaming
- **Alignment with project goals** - Exploratory, not production-grade

## Consequences

### Positive

- Rapid feedback during development
- Clear visibility into analytics (RTE), ML features (MLE), and telemetry (TEL)
- Tight integration with kdb ecosystem
- Minimal infrastructure requirements
- Simple polling model
- Standardized `.health[]` across all processes for consistent monitoring

### Negative / Trade-offs

- 1-second delay from Chained TP batching
- Polling introduces redundant queries
- No alerting capability

These trade-offs are acceptable for the current phase.

## Links / References

- `adr-001-timestamps-and-latency-measurement.md` (latency metrics)
- `adr-004-real-time-rolling-analytics-computation.md` (RTE analytics)
- `adr-005-telemetry-and-metrics-aggregation-strategy.md` (TEL metrics)
- `adr-011-financial-machine-learning.md` (MLE bar and position queries)
