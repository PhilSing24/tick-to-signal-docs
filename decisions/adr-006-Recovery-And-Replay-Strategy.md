# ADR-006: Recovery and Replay Strategy

## Status
Accepted (Updated 2026-01-07)

## Date
Original: 2025-12-17
Updated: 2025-12-29, 2026-01-06, 2026-01-07

## Context

In a typical production-grade KDB-X architecture, recovery and replay are achieved via:
- Durable tickerplant logs
- Replay of persisted data on restart
- Deterministic rebuild of downstream state (RDB, RTE)

The reference architecture describes multiple recovery patterns:

| Pattern | Description |
|---------|-------------|
| Recovery via TP log | Replay persisted log to rebuild RDB/RTE state |
| Recovery via RDB query | Component queries RDB for recent data |
| Recovery via on-disk cache | Load snapshot, replay delta |
| Upstream replay | Source provides missed data on reconnect |

This project is explicitly **exploratory and incremental** in nature.

## Notation

| Acronym | Definition |
|---------|------------|
| FH | Feed Handler |
| HDB | Historical Database |
| IPC | Inter-Process Communication |
| RDB | Real-Time Database |
| RTE | Real-Time Engine |
| TP | Tickerplant |
| LOG | Log Manager Process |

## Decision

The project implements a **production-grade recovery model** with automatic log replay on startup.

### Recovery Capability Summary

| Level | Capability | Status |
|-------|------------|--------|
| TP log format | `-11!` compatible binary logs | ✔ Implemented |
| RDB auto-replay | Automatic replay on startup | ✔ Implemented |
| RTE auto-replay | Automatic replay on startup | ✔ Implemented |
| Log management | Cleanup, diagnostics, verification | ✔ Implemented |
| FH recovery from Binance | Replay missed data from exchange | ✗ Not available |
| Gap detection | Identify missed events via sequence numbers | ✔ Available |
| Gap recovery | Fill gaps with missed data | ✗ Not implemented |

### Log Format

**Critical:** TP logs must be initialized with `set ()` for `-11!` streaming replay compatibility.

```q
/ Log initialization (in TP)
.tp.initLog:{[f]
  exists:0 < @[hcount; f; 0j];
  if[not exists; f set ()];   / Initialize empty log for -11!
  hopen f
  };
```

**Log file structure:**
- Binary IPC format (serialized q objects)
- Each entry: `enlist (`.u.upd; tableName; data)`
- Separate files per data type (see ADR-003):
  - `logs/YYYY.MM.DD.trade.log` – Trade events (13 fields from TP)
  - `logs/YYYY.MM.DD.quote.log` – Quote events (29 fields from TP)
- Daily rotation at midnight

**Log entry format:**
```q
/ Written by TP
logHandle enlist (`.u.upd; `trade_binance; data)

/ Replayed by -11! which calls .u.upd for each entry
```

### RDB Automatic Recovery

**On RDB startup:**
1. Check if today's log files exist and have content
2. If logs exist, replay them using `-11!` streaming execute
3. Subscribe to TP for live updates
4. Live data appends to replayed data (no gaps, no duplicates)

```q
/ RDB replay on startup
.rdb.replay:{[d]
  -1 "RDB: Starting replay for ",string[d];
  
  / Replay trade log
  tradeLog:.rdb.logFile[d;`trade];
  tradeChunks:.rdb.replayFile[tradeLog];
  
  / Replay quote log
  quoteLog:.rdb.logFile[d;`quote];
  quoteChunks:.rdb.replayFile[quoteLog];
  
  total:tradeChunks + quoteChunks;
  -1 "RDB: Replay complete - ",string[tradeChunks]," trades, ",string[quoteChunks]," quotes";
  total
  };

/ Single file replay with error handling
.rdb.replayFile:{[f]
  if[not .rdb.logExists[f]; -1 "RDB: Log file not found"; :0j];
  replayed:.[{-11!x}; enlist f; {[e] -1 "RDB: Replay error - ",e; 0j}];
  replayed
  };
```

**Key implementation detail:** During replay, `.u.upd` adds `rdbApplyTimeUtcNs` just like during live operation. The log contains TP data (13 fields for trades), and RDB adds the 14th field during replay.

### RTE Automatic Recovery

**On RTE startup:**
1. Check if today's log files exist and have content
2. If logs exist, replay them using `-11!` streaming execute
3. Run cleanup to remove stale data outside retention window
4. Subscribe to TP for live updates

```q
/ RTE replay on startup (similar to RDB)
.rte.replay:{[d]
  -1 "RTE: Starting replay for ",string[d];
  
  / Replay trade log (for VWAP)
  tradeChunks:.rte.replayFile[.rte.logFile[d;`trade]];
  
  / Replay quote log (for imbalance)
  quoteChunks:.rte.replayFile[.rte.logFile[d;`quote]];
  
  total:tradeChunks + quoteChunks;
  -1 "RTE: Replay complete";
  total
  };
```

**Key difference from RDB:** After replay, RTE runs cleanup to keep only the configured retention window (default 10 minutes). Old VWAP buckets are deleted; only the latest imbalance per symbol is kept.

### Log Manager (LOG) Process

A dedicated LOG process (port 5014) provides log file management:

| Function | Description |
|----------|-------------|
| `.log.list[]` | List all log files with sizes and chunk counts |
| `.log.summary[]` | Summary by date (trade/quote chunks, total size) |
| `.log.verify[file]` | Verify single log file integrity |
| `.log.verifyDate[date]` | Verify all logs for a date |
| `.log.cleanup[days]` | Delete logs older than N days |

See ADR-010 for detailed LOG process specification.

### Failure Scenario Matrix

| Failure | Data Impact | Recovery Action | Gap |
|---------|-------------|-----------------|-----|
| FH disconnects from Binance | Events during disconnect lost | FH reconnects; resumes live stream | Permanent |
| FH disconnects from TP | Events during disconnect lost | FH reconnects to TP; resumes publishing | Permanent |
| FH crash | Events during downtime lost | Restart FH; reconnect to Binance | Permanent |
| TP crash | RDB/RTE lose subscription | Restart TP; subscribers reconnect | Recoverable via log |
| RDB crash | All in-memory data lost | Restart RDB; **auto-replays from log** | Recoverable |
| RTE crash | Analytics state lost | Restart RTE; **auto-replays from log** | Recoverable |
| Full system restart | Everything lost | Restart all; RDB/RTE auto-replay | Recoverable |

### Gap Detection vs Gap Recovery

ADR-001 defines `fhSeqNo` (feed handler sequence number) and `tradeId` (Binance trade ID) for correlation.

| Capability | Status | Mechanism |
|------------|--------|-----------|
| Gap detection | ✔ Available | `fhSeqNo` gaps indicate missed FH events |
| Gap attribution | ✔ Available | Distinguish FH gaps from Binance gaps |
| Gap reporting | ✔ Available | Observable via telemetry (ADR-005) |
| Gap recovery | ✗ Not implemented | Gaps from FH downtime are permanent |

### Data Loss Acceptance

| Scenario | Data Lost | Recovery |
|----------|-----------|----------|
| Network blip (FH ↔ Binance) | Seconds of events | Not recoverable |
| Component restart (RDB/RTE) | None | **Auto-replay from log** |
| Full system restart | None (same day) | **Auto-replay from log** |
| Log file deleted | Historical data | Not recoverable |

### Performance Characteristics

**Replay speed:** ~450K msg/s (measured with trade data on local disk)

**Startup time with replay:**
- Empty logs: < 1 second
- 100K messages: ~0.2 seconds
- 1M messages: ~2 seconds

**Memory during replay:** Same as normal operation (RTE cleans up old buckets after replay)

## Rationale

This approach was selected because:

**Automatic replay:**
- Zero manual intervention on restart
- Production-grade reliability
- Simple implementation using `-11!` streaming execute
- Consistent with kdb+tick standard patterns

**LOG process:**
- Separates log management from data processing
- Enables retention policies
- Provides diagnostics without affecting TP/RDB/RTE
- Lightweight (minimal resource usage)

## Alternatives Considered

### 1. No replay capability (original design)
Revised:
- Data loss on every restart unacceptable
- Made debugging difficult
- Required live market for all testing

### 2. Manual replay only
Rejected:
- Requires operator intervention
- Risk of forgetting to replay
- Automatic is more reliable

### 3. RDB queries for RTE recovery
Rejected:
- Adds IPC dependency during startup
- RDB might not be ready
- Direct log replay is simpler

### 4. Use Binance REST API for gap fill
Rejected:
- Adds significant complexity
- REST and WebSocket data may differ
- Out of scope for current phase

### 5. Full checkpoint/snapshot recovery
Rejected:
- Significant implementation effort
- Replay from log is simpler
- Log replay is fast enough

### 6. Single combined log file
Rejected (see ADR-003):
- Must replay everything to recover anything
- Mixed data types complicate debugging
- Can't replay trades without quotes

## Consequences

### Positive

- **Zero data loss** on RDB/RTE restart (same day)
- Automatic recovery without operator intervention
- Production-grade reliability
- Fast startup even with large logs
- RTE state valid immediately after replay + cleanup
- LOG process enables retention management
- Consistent with kdb+tick standard patterns
- Separate log files allow selective replay

### Negative / Trade-offs

- Log files require disk space (~70MB/hour typical)
- Replay doesn't recover gaps from FH downtime
- Cross-day recovery requires manual intervention
- LOG process adds another component to manage
- RTE analytics may include old data briefly (until cleanup runs)

These trade-offs are acceptable for a production-grade system.

## Implementation Checklist

### Implemented

- [x] TP logging with `-11!` compatible format
- [x] Log file initialization with `set ()`
- [x] Separate log files per data type (trade, quote)
- [x] Daily log rotation
- [x] RDB automatic replay on startup
- [x] RTE automatic replay on startup
- [x] RTE cleanup after replay
- [x] LOG manager process
- [x] Log verification functions
- [x] Log cleanup functions
- [x] Integration with ctl.q control process

### Future Enhancements

- [ ] Cross-day replay (specify date range)
- [ ] Log compression
- [ ] Automated retention (scheduled cleanup)
- [ ] Log archival to HDB

### Out of Scope

- [ ] Binance REST API gap fill
- [ ] Real-time gap detection alerting
- [ ] Distributed recovery coordination

## Links / References

- `../kdbx-real-time-architecture-reference.md`
- `adr-001-timestamps-and-latency-measurement.md` (sequence numbers, timing fields)
- `adr-003-tickerplant-logging-and-durability-strategy.md` (TP logging, separate files)
- `adr-004-real-time-analytics-computation.md` (RTE bucketed design)
- `adr-005-telemetry-and-metrics-aggregation-strategy.md` (ephemeral telemetry)
- `adr-009-l5-order-book-architecture.md` (quote data, L5 depth)
- `adr-010-log-management-and-lifecycle.md` (LOG process details)
- `kdb/tp.q` (logging implementation)
- `kdb/rdb.q` (replay implementation)
- `kdb/rte.q` (replay implementation)
- `kdb/logmgr.q` (LOG process)
