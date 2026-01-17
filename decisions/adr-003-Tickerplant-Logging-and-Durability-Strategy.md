# ADR-003: Tickerplant Logging and Durability Strategy

## Status
Accepted (Updated 2026-01-17)

## Date
Original: 2025-12-17
Updated: 2025-12-29, 2026-01-06, 2026-01-09, 2026-01-16, 2026-01-17

## Context

In a canonical kdb real-time architecture, the tickerplant (TP) is responsible for:

- Sequencing inbound events
- Publishing updates to subscribers (WDB, MLE, CTP)
- Optionally providing a durability boundary via logging

This project ingests real-time Binance market data:
- Trade data via WebSocket trade stream
- Quote data via WebSocket depth stream with REST snapshot reconciliation
- Health metrics from feed handlers (operational monitoring)

A key design decision is whether and how the tickerplant should log incoming updates.

## Notation

| Acronym | Definition |
|---------|------------|
| CTP | Chained Tickerplant (batched publisher) |
| FH | Feed Handler |
| HDB | Historical Database |
| IPC | Inter-Process Communication |
| MLE | Machine Learning Engine |
| RDB | Real-time Database (query-only) |
| RTE | Real-Time Engine |
| TEL | Telemetry Process |
| TP | Tickerplant |
| WDB | Write-only Database (intraday writedown to HDB) |

## Decision

The tickerplant logs market data updates to a **single binary log file** per day, preserving chronological order across all tables. This follows the standard `tick.q` pattern.

### Log File Structure
```
logs/
  2025.12.29.log   # All market data events (trades + quotes) in arrival order
```

### Log File Naming

| Pattern | Example |
|---------|---------|
| `YYYY.MM.DD.log` | `2025.12.29.log` |

### Logging Configuration
```q
.tp.cfg.logEnabled:1b;
.tp.cfg.logDir:"logs";
```

### Implementation

The TP maintains a single file handle:
```q
.tp.logHandle    / Handle for combined log
.tp.logFile      / Current log file path
.tp.logCount     / Message counter
```

Logging logic in `.tp.log`:
```q
.tp.log:{[tbl;data]
  if[not .tp.cfg.logEnabled; :()];
  / Skip health updates from logging
  if[tbl = `health_feed_handler; :()];
  .tp.logHandle enlist (`.u.upd; tbl; data);
  .tp.logCount+:1;
  };
```

### Health Metrics (Not Logged)

The `health_feed_handler` table is **not logged** because:
- Ephemeral operational data (see ADR-005)
- TEL maintains 1-minute retention only
- Not needed for replay/recovery
- Feed handlers republish health every 5 seconds on restart

**Health data flow:**
```
Trade FH ───┐
            ├──► TP ──► CTP ──► TEL (subscription only, no logging)
Quote FH ───┘
```

### Rationale for Single File

| Benefit | Description |
|---------|-------------|
| Temporal consistency | Events replay in exact arrival order |
| Point-in-time recovery | Stop at message N = consistent state across ALL tables |
| Standard pattern | Matches canonical `tick.q` behavior |
| Simpler implementation | One handle, one file, one counter |
| Cross-table analytics | RTE state is correct during replay |

### End-of-Day Behaviour

- Log rotation occurs at midnight (new date = new file)
- `.tp.rotate[]` function closes old handle, opens new
- WDB writes data to HDB partitions during intraday and at EOD
- Logs are retained for replay/debugging

### Downstream Recovery Implications

| Component | On Restart | Recovery Source |
|-----------|------------|-----------------|
| WDB | Starts empty | Replay from log |
| MLE | State lost | Replay from log |
| CTP | Starts empty | N/A (no state) |
| RTE | State lost | Subscribes to CTP, rebuilds |
| TEL | State lost | Subscribes to CTP, rebuilds |
| RDB | Starts empty | Subscribes to CTP, rebuilds |
| TP | Logs reset | N/A |

### Gap Detection

The TP validates `fhSeqNo` for trade messages to detect IPC-level data loss:
- Counters: `.tp.gaps.trade`, `.tp.missed.trade`, `.tp.restarts.trade`
- Query via `.tp.status[]`
- Quote gap detection is deferred (high message rate causes false positives)

### Tables and Logging

| Table | Logged? | Subscribers |
|-------|---------|-------------|
| `trade_binance` | ✅ Yes | WDB, MLE, CTP |
| `quote_binance` | ✅ Yes | WDB, CTP |
| `health_feed_handler` | ❌ No (ephemeral) | CTP → TEL |

**Note:** WDB and CTP subscribe to both trades and quotes. MLE subscribes to trades only.

## Rationale

A single log file was selected because:

- **Temporal fidelity**: Events replay in true arrival order
- **Point-in-time recovery**: Can stop at any message for consistent cross-table state
- **Simpler debugging**: One file to inspect for audit/forensics
- **Cross-table analytics**: RTE computing on both feeds gets correct state during replay
- **Standard pattern**: Matches canonical kdb+tick design

## Alternatives Considered

### 1. Separate log files per table (previous design)
Rejected (2026-01-09):
- Breaks chronological order during replay
- WDB replays all trades, then all quotes (wrong order)
- RTE state incorrect during replay
- Complicates point-in-time recovery

### 2. No logging (original design)
Updated:
- Logging now enabled by default
- Provides replay capability
- Minimal latency impact

### 3. Per-symbol log files
Rejected:
- Too many files (3 symbols x 2 types = 6 files)
- Complicates replay
- Overkill for current scale

### 4. Separate logger process
Rejected:
- Adds complexity
- TP can handle logging efficiently
- Not needed at current scale

### 5. Log health_feed_handler table
Rejected:
- Ephemeral data (1-minute retention)
- Feed handlers republish on restart
- Not needed for replay
- Adds log volume for no benefit

## Consequences

### Positive

- Correct temporal ordering during replay
- Point-in-time recovery possible
- Simpler implementation (one file handle)
- Matches standard tick.q pattern
- Cross-table analytics work correctly during replay
- Health metrics don't pollute logs
- Clear separation: market data (logged) vs operational data (ephemeral)

### Negative / Trade-offs

- Cannot replay trades without quotes (or vice versa)
- Single file may be larger
- No per-table retention policies

These trade-offs are acceptable for temporal consistency.

## Replay Support

Replay uses standard `-11!` streaming execute:
```bash
# Replay all data for a date
q kdb/wdb.q   # Auto-replays on startup

# Manual replay
q
.wdb.replay[2025.12.29]
```

**Note:** MLE also auto-replays on startup, then runs cleanup to keep only the retention window.

## Links / References

- `reference/kdbx-real-time-architecture-reference.md`
- `adr-001-timestamps-and-latency-measurement.md` (per-event fields)
- `adr-002-feed-handler-to-kdb-ingestion-path.md` (market data flow)
- `adr-005-telemetry-and-metrics-aggregation-strategy.md` (health metrics)
- `adr-006-recovery-and-replay-strategy.md` (replay mechanism)
- `adr-009-order-book-architecture.md` (quote data source)
