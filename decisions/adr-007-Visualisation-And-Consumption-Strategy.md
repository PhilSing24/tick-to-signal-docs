# ADR-007: Visualisation and Consumption Strategy

## Status
Accepted (Updated 2026-01-14)

## Date
2025-12-17 (Updated 2025-12-19, 2026-01-14)

## Context

The system produces:
- Real-time market data (Binance trades and quotes)
- Derived analytics (VWAP, order book imbalance, variance-covariance)
- Telemetry and latency metrics

These outputs must be:
- Observable by humans during development and operation
- Suitable for validating system behaviour
- Easy to iterate on during exploration

Multiple visualisation approaches are possible:

| Approach | Description |
|----------|-------------|
| Custom UI builds | Bespoke web or desktop application |
| External BI tools | Grafana, Tableau, etc. |
| Streaming dashboards | Push-based real-time updates |
| KX Dashboards | Native KDB-X visualisation tooling |

A decision is required on how system outputs are visualised and consumed.

## Notation

| Acronym | Definition |
|---------|------------|
| FH | Feed Handler |
| IPC | Inter-Process Communication |
| RDB | Real-Time Database |
| RTE | Real-Time Engine |
| TEL | Telemetry Process |
| TP | Tickerplant |

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

Polling is appropriate because:
- Simpler to implement
- Sufficient for human-scale observation
- Avoids premature optimisation
- Aligns with exploratory project goals

### Polling Frequency

**1-second polling interval** is selected:

| Frequency | Trade-off |
|-----------|-----------|
| 200ms | Minimum for human perception; highest load |
| 500ms | Good responsiveness; moderate load |
| **1 second** | **Balanced; acceptable responsiveness** |
| 5 seconds | Low load; sluggish feel |

Rationale:
- Acceptable responsiveness for development use
- Minimal query load on RDB/RTE/TEL
- Note: TEL buckets are 5 seconds, so telemetry data updates every 5s regardless

### Query Targets

Dashboards query **RDB, RTE, and TEL**:

```
              +---> RDB :5011 (raw trades, quotes)
              |
Dashboard ----+---> RTE :5012 (VWAP, imbalance, var-covar, analytics validity)
              |
              +---> TEL :5013 (latency telemetry, FH health, system metrics)
```

| Data | Source | Port | Table/Query |
|------|--------|------|-------------|
| Last trade price | RDB | 5011 | `trade_binance` |
| Raw trade history | RDB | 5011 | `trade_binance` |
| L5 order book | RTE | 5012 | `.rte.getOrderBook[]` |
| Rolling VWAP | RTE | 5012 | `.rte.getVwap[]` |
| Order book imbalance | RTE | 5012 | `.rte.getImbalanceAll[]` |
| Variance-covariance | RTE | 5012 | `.rte.getCorrelationTable[]` |
| Analytics validity | RTE | 5012 | `.rte.getVwap[]`, `.rte.getVcov[]` (return `isValid`) |
| FH latency metrics | TEL | 5013 | `telemetry_latency_fh` |
| E2E latency metrics | TEL | 5013 | `telemetry_latency_e2e` |
| FH health status | TEL | 5013 | `.tel.vsFhStatus[]` |
| System memory | TEL | 5013 | `.tel.vsSystemResources[]` |
| Throughput | TEL | 5013 | Derived from `telemetry_latency_fh.cnt / 5` |

**Trade-off acknowledged:** The reference architecture recommends:
> "Processes accessed by users should be isolated from those performing core system calculations to improve stability and resource control."

For this exploratory project with a single user, direct querying is acceptable. A UI cache or gateway would be introduced if user count grows.

### Dashboard Specification

Three dashboards are defined:

**Dashboard 1: Market Overview**

| Panel | Data Source | Content | Update |
|-------|-------------|---------|--------|
| Last Price | RDB | Most recent trade price per symbol | 1 sec |
| Rolling Avg Price | RTE | `avgPrice` from `.rte.getVwap[]` | 1 sec |
| Trade Count | RTE | `tradeCount` from `.rte.getVwap[]` | 1 sec |
| Price Chart | RDB | Time series of recent prices (last 5 min) | 1 sec |
| VWAP Validity | RTE | `isValid` from `.rte.getVwap[]` | 1 sec |
| Order Book | RTE | `.rte.getOrderBook[]` | 1 sec |
| OBI Indicator | RTE | `.rte.getImbalanceAll[]` | 1 sec |

**Dashboard 2: Latency Monitor**

| Panel | Data Source | Content | Update |
|-------|-------------|---------|--------|
| FH Parse Latency | TEL | p50/p95/max from `telemetry_latency_fh` | 1 sec |
| FH Send Latency | TEL | p50/p95/max from `telemetry_latency_fh` | 1 sec |
| E2E Latency | TEL | p50/p95/max from `telemetry_latency_e2e` | 1 sec |
| Latency Time Series | TEL | Rolling chart of percentiles (last 5 min) | 1 sec |
| Throughput Chart | TEL | `cnt / 5` from `telemetry_latency_fh` (events/sec) | 1 sec |

**Dashboard 3: System Health**

| Panel | Data Source | Content | Update |
|-------|-------------|---------|--------|
| FH Health Grid | TEL | `.tel.vsFhStatus[]` | 1 sec |
| System Memory | TEL | `.tel.vsSystemResources[]` | 1 sec |
| Data Volume | TEL | `.tel.vsDataVolume[]` | 1 sec |
| Analytics Validity | RTE | `isValid` from `.rte.getVwap[]`, `.rte.getVcov[]` | 1 sec |
| Data Freshness | RDB | Time since last trade per symbol | 1 sec |
| Gap Indicator | RDB | Detected `fhSeqNo` gaps (if any) | 1 sec |

### Validity Display

Analytics validity (from ADR-004) is displayed using visual indicators:

| Condition | Display |
|-----------|---------|
| `isValid = true` | Green indicator; normal value display |
| `isValid = false`, partial data | Yellow indicator; value shown with warning |
| `isValid = false`, insufficient data | Red indicator; value greyed out or hidden |

Additionally:
- VWAP shows `tradeCount` to indicate sample size
- Var-covar shows `buckets` count to indicate data coverage
- Tooltip explains validity meaning on hover

### Update Latency Expectations

| Metric | Source | Target Latency (data age) |
|--------|--------|---------------------------|
| Last price | RDB | < 2 seconds |
| Rolling avg | RTE | < 2 seconds |
| Telemetry | TEL | < 7 seconds (5s bucket + 2s query) |

These targets are achievable with 1-second polling and account for:
- Poll interval (up to 1 second)
- Query execution time (< 100ms expected)
- Rendering time (< 100ms expected)
- TEL bucket delay (5 seconds for telemetry data)

The reference architecture notes:
> "A minimum update interval of approximately 200 milliseconds is generally sufficient, as this aligns with the threshold at which the human eye can perceive simple visual changes."

1-second polling is well within acceptable bounds for human observation.

### Scalability Stance

**Scalability is out of scope for the current phase.**

The reference architecture describes a UI caching pattern for scale:
> "A common approach to scaling visualisation workloads is the introduction of a server-side UI cache."

For this project:
- Single user (developer) assumed
- Direct RDB/RTE/TEL queries acceptable
- No UI cache implemented
- No gateway process implemented

If multiple users or higher query rates are needed:
- Introduce UI cache process
- Cache pre-computes common queries
- Dashboards query cache instead of RDB/RTE/TEL

### Intended Usage

Dashboards are intended for:
- Development validation
- Real-time behaviour inspection
- Latency and performance reasoning
- Architecture exploration

They are **not** intended as:
- End-user trading interfaces
- Production monitoring (no alerting)
- Historical analysis tools

### Symbol Scope

Initial dashboards display data for:
- BTCUSDT
- ETHUSDT
- SOLUSDT

Each symbol has dedicated panels or can be filtered via dashboard controls.

## Rationale

KX Dashboards with polling was selected because:

- **Native integration** - No additional tooling or infrastructure required
- **Rapid development** - Pre-built components for time-series visualisation
- **Sufficient for purpose** - Human-scale observation doesn't require streaming
- **Alignment with project goals** - Exploratory, not production-grade
- **Consistency** - Data stays within KDB-X ecosystem

## Alternatives Considered

### 1. Streaming dashboards
Rejected for current phase:
- Adds complexity (subscription management, reconnection)
- Not required for single-user development use
- Premature optimisation

May be introduced if real-time push updates become necessary.

### 2. External BI tools (Grafana, Tableau)
Rejected:
- Requires additional infrastructure
- Integration overhead
- Distracts from core kdb exploration

Grafana may be considered for production monitoring in future phases.

### 3. Custom web UI
Rejected:
- Significant development effort
- Out of scope for exploratory project
- KX Dashboards sufficient for current needs

### 4. Query on-demand only (no dashboards)
Rejected:
- Poor developer experience
- Manual queries tedious for continuous observation
- Dashboards add minimal overhead

### 5. Dedicated UI cache/gateway
Rejected for current phase:
- Adds process complexity
- Single user doesn't require isolation
- Direct queries acceptable at current scale

May be introduced if user count grows.

### 6. Faster polling (200ms)
Rejected:
- Higher query load
- No perceptible benefit for human observation
- TEL data only updates every 5 seconds anyway

### 7. Slower polling (5 seconds)
Rejected:
- Sluggish feel during development
- Delays in observing system behaviour
- 1-second provides better responsiveness for RDB/RTE data

### 8. Separate throughput table in TEL
Rejected:
- Throughput is trivially derived from existing `cnt` field
- No need for redundant storage
- `events_per_sec = cnt / 5`

### 9. Separate analytics health table in TEL
Rejected:
- RTE already exposes validity via query functions
- Dashboard already queries RTE for analytics data
- No duplication needed

## Consequences

### Positive

- Rapid feedback during development
- Clear visibility into raw data (RDB), analytics (RTE), and telemetry (TEL)
- Tight integration with kdb ecosystem
- Minimal infrastructure requirements
- Validity indicators aid interpretation
- Simple polling model easy to debug
- No redundant tables (throughput derived, analytics health from RTE)

### Negative / Trade-offs

- Polling introduces redundant queries (even when data unchanged)
- Dashboards not optimised for large user populations
- Fine-grained UI customisation is limited
- Direct queries mix user and system load
- No alerting capability (human must observe)
- Telemetry has 5-second delay due to bucket size
- Dashboard must query both TEL and RTE for complete health picture

These trade-offs are acceptable for the current phase.

## Future Evolution

| Enhancement | Trigger |
|-------------|---------|
| Streaming dashboards | Need for sub-second updates |
| UI cache layer | Multiple concurrent users |
| Gateway process | Query isolation requirement |
| Grafana integration | Production monitoring needs |
| Custom web UI | Specific UX requirements |
| Alerting | Production deployment |
| Historical dashboards | HDB implementation (ADR-003 evolution) |

## Links / References

- `../kdbx-real-time-architecture-reference.md`
- `../kdbx-real-time-architecture-measurement-notes.md`
- `adr-001-timestamps-and-latency-measurement.md` (latency metrics definitions)
- `adr-004-real-time-rolling-analytics-computation.md` (RTE analytics, validity flags)
- `adr-005-telemetry-and-metrics-aggregation-strategy.md` (TEL tables, bucket alignment)
- `adr-006-recovery-and-replay-strategy.md` (gap detection display)
