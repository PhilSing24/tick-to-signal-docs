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
2. Download all files from this repository
3. Upload them to **Project Knowledge**
4. Add the custom instructions below
5. Start asking questions

### Suggested Project Instructions

```
You are an expert on the tick-to-signal project, a real-time market data pipeline for crypto trading analytics.

Architecture:
- C++ feed handlers ingest Binance WebSocket data (trades + L5 quotes)
- kdb+ tickerplant distributes data to subscribers
- RTE (Real-Time Engine) computes analytics: VWAP, OBI, var-covar, correlation
- KX Dashboard visualizes real-time metrics

Your knowledge base includes:
- paper/: Academic white paper on the system
- decisions/: Architecture Decision Records (ADR-001 through ADR-009)
- specs/: Data schemas and APIs
- reference/: External references
- notes/: Implementation notes

When answering:
- Cite the specific document (e.g., "per ADR-004")
- Cross-reference between documents when relevant
- If information isn't documented, say so explicitly
```

### Example Questions

- "How does the VWAP bucketing strategy work?"
- "What are the latency targets and how are they achieved?"
- "Explain the order book imbalance EMA smoothing"
- "Why was kdb+ chosen over alternatives?"
- "What happens when the RTE process restarts?"

## Code

The implementation lives in a separate repository: [tick-to-signal](https://github.com/PhilSing24/tick-to-signal). The code repository is private. 
If you're interested in the implementation or have questions, feel free to reach out via [LinkedIn](https://www.linkedin.com/in/phdamay) or open an issue in this repository.

## White Paper

**Tick Data to Signals: Real-Time Crypto Analytics on KDB-X**

Draft available at [paper/tick-data-to-signals-v1.0-draft.pdf](paper/tick-data-to-signals-v1.0-draft.pdf).
