# Tick to Signal - Documentation

Documentation for the [tick-to-signal](https://github.com/PhilSing24/tick-to-signal) real-time crypto analytics system.

## White Paper

**Tick Data to Signals: Real-Time Crypto Analytics on KDB-X**

Draft available at [paper/tick-data-to-signals-v1.0-draft.pdf](paper/tick-data-to-signals-v1.0-draft.pdf).

Source maintained in Overleaf.

## Architecture Decision Records

| ADR | Title |
|-----|-------|
| [001](decisions/adr-001-Timestamps-and-latency-measurement.md) | Timestamps and Latency Measurement |
| [002](decisions/adr-002-Feed-handler-to-kdb-ingestion-path.md) | Feed Handler to kdb+ Ingestion Path |
| [003](decisions/adr-003-Tickerplant-Logging-and-Durability-Strategy.md) | Tickerplant Logging and Durability Strategy |
| [004](decisions/adr-004-Real-Time-Analytics-Computation.md) | Real-Time Analytics Computation |
| [005](decisions/adr-005-Telemetry-And-Metrics-Aggregation-Strategy.md) | Telemetry and Metrics Aggregation Strategy |
| [006](decisions/adr-006-Recovery-And-Replay-Strategy.md) | Recovery and Replay Strategy |
| [007](decisions/adr-007-Visualisation-And-Consumption-Strategy.md) | Visualisation and Consumption Strategy |
| [008](decisions/adr-008-Error-Handling-Strategy.md) | Error Handling Strategy |
| [009](decisions/adr-009-L5-Order-Book-Architecture.md) | L5 Order Book Architecture |
| [010](decisions/adr-010-Log-Management-And-Lifecycle.md) | Log Management and Lifecycle |

## Specs

- [api-binance.md](specs/api-binance.md) - Binance WebSocket API reference
- [trades-schema.md](specs/trades-schema.md) - Trade message schema
- [quotes-schema.md](specs/quotes-schema.md) - Quote message schema

## Reference

- [DataIntellect_Real_Time_KDBX.pdf](reference/DataIntellect_Real_Time_KDBX.pdf) - Building Real-Time Event-Driven KDB-X Systems
- [kdbx-real-time-architecture-reference.md](reference/kdbx-real-time-architecture-reference.md) - AI-friendly text version

## Notes

- [kdbx-real-time-architecture-measurement-notes.md](notes/kdbx-real-time-architecture-measurement-notes.md) - Latency measurement implementation notes

## License

Private repository.
