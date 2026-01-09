# Tick to Signal

**Real-Time Crypto Analytics on KDB-X**

This project is a production-grade implementation of a real-time cryptocurrency analytics system. It ingests live market data from Binance WebSocket streams via C++ feed handlers, computes VWAP and order book imbalance metrics in kdb+, and delivers them to downstream consumers with minimal latency.

The system runs on modest hardware: a WSL environment with 12 CPU threads and 7.6GB of RAM. This constraint is intentional. The goal is not to throw resources at the problem, but to demonstrate that **thoughtful architecture and code quality can achieve production-grade performance without enterprise-scale infrastructure**.

## Why This Project?

Modern infrastructure offers abundant compute power. Cloud instances scale on demand, memory is cheap, and the instinct is often to throw more resources at a problem rather than solve it elegantly. This works—until it doesn't. Latency-sensitive systems expose the inefficiencies that brute force cannot hide.

Arthur Whitney, the creator of kdb+, writes dense code with a specific belief: if code fits entirely in the CPU's L1 cache, it runs orders of magnitude faster. A well-written kdb+ system can accomplish with 8GB of RAM what many enterprise Java or Python stacks struggle to achieve with far more.

This project applies that philosophy to a concrete problem: real-time crypto analytics.

## Technology Choices

**C++** for the feed handlers—direct control over memory layout, deterministic performance, and efficient integration with WebSocket libraries and the kdb+ IPC interface. For latency-sensitive ingestion where every microsecond matters, C++ remains the right tool.

**kdb+** for analytics—its vector-oriented execution model and columnar data structures make it exceptionally efficient for the aggregations and windowed calculations that VWAP and order book imbalance require.

## Observability by Design

This implementation is not presented as definitive. Improvements remain to be found. That is precisely why telemetry is embedded throughout the system. Every event carries nanosecond timestamps at ingestion, microsecond measurements for parsing and IPC, and sequence numbers for gap detection.

The architecture is designed not just to perform, but to reveal where it can perform better. **You cannot optimize what you cannot measure.**

## Repository Structure

| Folder | Contents |
|--------|----------|
| [paper/](paper/) | White paper (PDF) |
| [decisions/](decisions/) | Architecture Decision Records |
| [specs/](specs/) | API and data schemas |
| [reference/](reference/) | External references |
| [notes/](notes/) | Implementation notes |

## Code

The implementation lives in a separate repository: [tick-to-signal](https://github.com/PhilSing24/tick-to-signal)

## White Paper

**Tick Data to Signals: Real-Time Crypto Analytics on KDB-X**

Draft available at [paper/tick-data-to-signals-v1.0-draft.pdf](paper/tick-data-to-signals-v1.0-draft.pdf).
