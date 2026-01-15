# Tick to Signal

**Real-Time Crypto Analytics on KDB-X**

This project is a production-grade real-time cryptocurrency analytics system. It ingests live market data from Binance WebSocket streams via C++ feed handlers, and computes analytics in kdb+:

- **VWAP**, **Volatility** and **Correlation** from tick data
- **Order book imbalance** from L2 snapshots with 5 levels of depth

All metrics are delivered to downstream consumers with minimal latency.

The goal is to demonstrate that thoughtful architecture and code quality can achieve production-grade performance without enterprise-scale infrastructure.

## The System in Action

**Feed Handler Monitoring** — Health status and latency metrics for trade and quote ingestion.

![Feed Handler Monitoring](images/FeedhandlerMonitoringView.png)

**Dataflow Monitoring** — Data volume, system resources, and end-to-end latency breakdown.

![Dataflow Monitoring](images/DataFlowMonitoringView.png)

**Trades and Quotes Monitoring** — Order Book and OHLC 

![Trades and Quotes Monitoring](images/TradesQuotesView.png)

**Analytics** — VWAP, Volatility, Correlation, Order Book Imbalance

![Analytics Monitoring](images/AnalyticsView.png)


## Technology Choices

**C++** for the feed handlers—direct control over memory layout, deterministic performance, and efficient integration with WebSocket libraries and the kdb+ IPC interface. For latency-sensitive ingestion where every microsecond matters, C++ remains the right tool.

**kdb+** for analytics—its vector-oriented execution model and columnar data structures make it exceptionally efficient for the aggregations and windowed calculations that VWAP and order book imbalance require.

## Observability by Design

Telemetry is embedded throughout the system. Every event carries nanosecond timestamps at ingestion, microsecond measurements for parsing and IPC, and sequence numbers for gap detection. You cannot optimize what you cannot measure.


## Repository Structure

| Folder | Contents |
|--------|----------|
| [paper/](paper/) | White paper (PDF) |
| [decisions/](decisions/) | Architecture Decision Records |
| [specs/](specs/) | API and data schemas |
| [reference/](reference/) | External references |
| [notes/](notes/) | Implementation notes |

## Exploring with AI

The documentation in this repository is structured for AI-assisted exploration. All architecture decisions, specifications, and implementation notes are written in markdown—making them ideal for use with Large Language Models like Claude.

### Using Claude Projects

To create an AI assistant that understands this system:

1. Go to [claude.ai](https://claude.ai) → **Projects** → **New Project**
2. Create a personal project (Name and describe your project)
3. Fill Instructions
4. Add files from Github
5. Start asking questions


## Code

The implementation lives in a separate repository: [tick-to-signal](https://github.com/PhilSing24/tick-to-signal). The code repository is private. 
If you're interested in the implementation or have questions, feel free to reach out via [LinkedIn](https://www.linkedin.com/in/phdamay) or open an issue in this repository.

## White Paper

**Tick Data to Signals: Real-Time Crypto Analytics on KDB-X**

Draft available at [paper/Real_time_Crypto_KDB.pdf](paper/Real_time_Crypto_KDB.pdf).


## Architecture: Divergences from Reference Patterns

### Acknowledged Trade-offs

| Pattern | Reference Recommendation | Project Decision | Rationale |
|---------|-------------------------|------------------|-----------|
| **Batching** | Batch messages for throughput | Tick-by-tick publishing | Prioritizes latency measurement clarity over throughput |
| **User isolation** | Separate UI cache layer | Direct queries to RDB/RTE/TEL | Acceptable for single-user development; simplifies architecture |
| **Timer vs event-driven** | Event-driven processing preferred | Mixed (hot path event-driven, cold path timer-based) | Var-covar and cleanup are computationally expensive; timer-based makes sense |

### Patterns Not Implemented

| Pattern | Reference Recommendation | Project Status | Impact |
|---------|-------------------------|----------------|--------|
| **FH local recovery log** | Feed handler maintains local log for replay after TP disconnect | Not implemented | Gaps during FH↔TP disconnect are permanent |
| **Replicated feed handlers** | Multiple FH instances for resilience | Single instance | Single point of failure at ingestion boundary |
| **Status propagation** | Pub/sub staleness cascade across components | Validity flags only (no automatic propagation) | Downstream must query validity rather than subscribe |
| **Hot path separation** | Dedicated low-latency data flow via separate TP | Single TP for all data | All data treated uniformly; no latency-optimized path |


