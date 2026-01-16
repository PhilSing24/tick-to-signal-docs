# ADR-006: Recovery and Replay Strategy

## Status
Accepted (Updated 2026-01-16)

## Date
Original: 2025-12-17
Updated: 2025-12-29, 2026-01-06, 2026-01-07, 2026-01-09, 2026-01-11, 2026-01-16

## Context

In a typical production-grade KDB-X architecture, recovery and replay are achieved via:
- Durable tickerplant logs
- Replay of persisted data on restart
- Deterministic rebuild of downstream state (WDB, RTE, MLE)

The reference architecture describes multiple recovery patterns:

| Pattern | Description |
|---------|-------------|
| Recovery via TP log | Replay persisted log to rebuild WDB/RTE/MLE state |
| Recovery via WDB query | Component queries WDB for recent data |
| Recovery via on-disk cache | Load snapshot, replay delta |
| Upstream replay | Source provides missed data on reconnect |

This project is explicitly **exploratory and incremental** in nature.

## Notation

| Acronym | Definition |
|---------|------------|
| FH | Feed Handler |
| HDB | Historical Database |
| IPC | Inter-Process Communication |
| MLE | Machine Learning Engine |
| RTE | Real-Time Engine |
| TEL | Telemetry Process |
| TP | Tickerplant |
| WDB | Write-only Database (intraday writedown to HDB) |

## Decision

The project implements a **production-grade recovery model** with automatic log replay on startup using a **single log file** that preserves chronological order across all tables.

### Recovery Capability Summary

| Level | Capability | Status |
|-------|------------|--------|
| TP log format | `-11!` compatible binary logs | ✓ Implemented |
| Single log file | Temporal consistency across tables | ✓ Implemented |
| WDB auto-replay | Automatic replay on startup | ✓ Implemented |
| RTE auto-replay | Automatic replay on startup | ✓ Implemented |
| MLE auto-replay | Automatic replay on startup | ✓ Implemented |
| Log management | Cleanup, diagnostics, verification | ✓ Implemented (on-demand) |
| FH recovery from Binance | Replay missed data from exchange | ✗ Not available |
| Gap detection | Identify missed events via sequence numbers | ✓ Available |
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
- **Single file per day** containing all market data in arrival order:
  - `logs/YYYY.MM.DD.log` — All events (trades + quotes interleaved)
- Daily rotation at midnight

**Log entry format:**
```q
/ Written by TP (trades and quotes interleaved chronologically)
logHandle enlist (`.u.upd; `trade_binance; tradeData)
logHandle enlist (`.u.upd; `quote_binance; quoteData)
logHandle enlist (`.u.upd; `trade_binance; tradeData)
...

/ Replayed by -11! which calls .u.upd for each entry in order
```

### WDB Automatic Recovery

**On WDB startup:**
1. Check if today's log file exists and has content
2. If log exists, replay it using `-11!` streaming execute
3. Subscribe to TP for live updates
4. Live data appends to replayed data (no gaps, no duplicates)

```q
/ WDB replay on startup
.wdb.replay:{[d]
  if[d ~ (::); d:.z.D];
  -1 "WDB: Starting replay for ",string[d];
  
  / Replay single log file (trades + quotes in chronological order)
  logFile:.wdb.logFile[d];
  chunks:.wdb.replayFile[logFile];
  
  -1 "WDB: Replay complete - ",string[chunks]," chunks";
  -1 "WDB: Tables now have ",string[count trade_binance]," trades, ",string[count quote_binance]," quotes";
  chunks
  };

/ Single file replay with error handling
.wdb.replayFile:{[f]
  if[not .wdb.logExists[f]; -1 "WDB: Log file not found"; :0j];
  replayed:.[{-11!x}; enlist f; {[e] -1 "WDB: Replay error - ",e; 0j}];
  replayed
  };
```

**Key implementation detail:** During replay, `.u.upd` adds `wdbRecvTimeUtcNs` just like during live operation. The log contains TP data (13 fields for trades), and WDB adds the 14th field during replay.

### RTE Automatic Recovery

**On RTE startup:**
1. Check if today's log file exists and has content
2. If log exists, replay it using `-11!` streaming execute
3. Run cleanup to remove stale data outside retention window
4. Subscribe to TP for live updates

```q
/ RTE replay on startup
.rte.replay:{[d]
  if[d ~ (::); d:.z.D];
  -1 "RTE: Starting replay for ",string[d];
  
  / Replay single log file (trades + quotes in chronological order)
  logFile:.rte.logFile[d];
  chunks:.rte.replayFile[logFile];
  
  -1 "RTE: Replay complete - ",string[chunks]," chunks";
  -1 "RTE: Trade buckets: ",string[count tradeBuckets],", Imbalance symbols: ",string[count .rte.imb.latest];
  chunks
  };
```

**Key benefit of single file:** During replay, trades and quotes arrive in the same order as live operation. This means:
- VWAP buckets are populated correctly
- Imbalance state reflects the correct sequence of order book updates
- Any cross-table analytics produce correct results

**After replay:** RTE runs cleanup to keep only the configured retention window (default 10 minutes). Old VWAP buckets are deleted; only the latest imbalance per symbol is kept.

### MLE Automatic Recovery

**On MLE startup:**
1. Check if today's log file exists and has content
2. If log exists, replay it using `-11!` streaming execute
3. Run cleanup to remove stale bars outside retention window
4. Subscribe to TP for live updates

**After replay:** MLE runs cleanup to keep only the configured retention window (default 24 hours). Bar state is restored.

### Log Manager (On-Demand)

Log management is performed on-demand via `logmgr.q`:

| Function | Description |
|----------|-------------|
| `.log.list[]` | List all log files with sizes and chunk counts |
| `.log.summary[]` | Summary by date (chunks, total size) |
| `.log.verify[file]` | Verify single log file integrity |
| `.log.verifyDate[date]` | Verify log for a date |
| `.log.cleanup[days]` | Delete logs older than N days |

**Usage:** Run `q logmgr.q` manually or schedule via cron.

See ADR-010 for detailed log management specification.

### Failure Scenario Matrix

| Failure | Data Impact | Recovery Action | Gap |
|---------|-------------|-----------------|-----|
| FH disconnects from Binance | Events during disconnect lost | FH reconnects; resumes live stream | Permanent |
| FH disconnects from TP | Events during disconnect lost | FH reconnects to TP; resumes publishing | Permanent |
| FH crash | Events during downtime lost | Restart FH; reconnect to Binance | Permanent |
| TP crash | WDB/RTE/MLE lose subscription | Restart TP; subscribers reconnect | Recoverable via log |
| WDB crash | All in-memory data lost | Restart WDB; **auto-replays from log** | Recoverable |
| RTE crash | Analytics state lost | Restart RTE; **auto-replays from log** | Recoverable |
| MLE crash | Bar state lost | Restart MLE; **auto-replays from log** | Recoverable |
| Full system restart | Everything lost | Restart all; WDB/RTE/MLE auto-replay | Recoverable |

### Gap Detection vs Gap Recovery

ADR-001 defines `fhSeqNo` (feed handler sequence number) and `tradeId` (Binance trade ID) for correlation.

| Capability | Status | Mechanism |
|------------|--------|-----------|
| Gap detection | ✓ Available | `fhSeqNo` gaps indicate missed FH events |
| Gap attribution | ✓ Available | Distinguish FH gaps from Binance gaps |
| Gap reporting | ✓ Available | Observable via telemetry (ADR-005) |
| Gap recovery | ✗ Not implemented | Gaps from FH downtime are permanent |

### Data Loss Acceptance

| Scenario | Data Lost | Recovery |
|----------|-----------|----------|
| Network blip (FH ↔ Binance) | Seconds of events | Not recoverable |
| Component restart (WDB/RTE/MLE) | None | **Auto-replay from log** |
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

**Single log file:**
- Preserves chronological order across all tables
- Enables point-in-time recovery with consistent state
- Matches standard kdb+tick pattern
- RTE analytics correct during replay

**Automatic replay:**
- Zero manual intervention on restart
- Production-grade reliability
- Simple implementation using `-11!` streaming execute
- Consistent with kdb+tick standard patterns

**On-demand log management:**
- Separates log management from data processing
- Enables retention policies
- Provides diagnostics without affecting TP/WDB/RTE
- No additional running process

## Alternatives Considered

### 1. Separate log files per table (previous design)
Rejected (2026-01-09):
- Breaks chronological order during replay
- WDB replays all trades, then all quotes (wrong sequence)
- RTE state may be incorrect during replay
- Cannot do point-in-time recovery across tables
- Complicates debugging (which file had the issue?)

### 2. No replay capability (original design)
Revised:
- Data loss on every restart unacceptable
- Made debugging difficult
- Required live market for all testing

### 3. Manual replay only
Rejected:
- Requires operator intervention
- Risk of forgetting to replay
- Automatic is more reliable

### 4. WDB queries for RTE recovery
Rejected:
- Adds IPC dependency during startup
- WDB might not be ready
- Direct log replay is simpler

### 5. Use Binance REST API for gap fill
Rejected:
- Adds significant complexity
- REST and WebSocket data may differ
- Out of scope for current phase

### 6. Full checkpoint/snapshot recovery
Rejected:
- Significant implementation effort
- Replay from log is simpler
- Log replay is fast enough

## Consequences

### Positive

- **Zero data loss** on WDB/RTE/MLE restart (same day)
- **Correct temporal ordering** during replay
- **Point-in-time recovery** possible across all tables
- Automatic recovery without operator intervention
- Production-grade reliability
- Fast startup even with large logs
- RTE state valid immediately after replay + cleanup
- Log management via on-demand tool
- Consistent with kdb+tick standard patterns
- Simpler implementation (one file per day)

### Negative / Trade-offs

- Log files require disk space (~70MB/hour typical)
- Replay doesn't recover gaps from FH downtime
- Cross-day recovery requires manual intervention
- Cannot replay trades without quotes (or vice versa)
- RTE analytics may include old data briefly (until cleanup runs)

These trade-offs are acceptable for a production-grade system.

## Implementation Checklist

### Implemented

- [x] TP logging with `-11!` compatible format
- [x] Log file initialization with `set ()`
- [x] **Single log file** per day (trades + quotes interleaved)
- [x] Daily log rotation
- [x] WDB automatic replay on startup
- [x] RTE automatic replay on startup
- [x] MLE automatic replay on startup
- [x] RTE cleanup after replay
- [x] Log manager (on-demand)
- [x] Log verification functions
- [x] Log cleanup functions
- [x] Integration with ctl.q control process
- [x] Graceful shutdown (SIGTERM before SIGKILL)

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

- `reference/kdbx-real-time-architecture-reference.md`
- `adr-001-timestamps-and-latency-measurement.md` (sequence numbers, timing fields)
- `adr-003-tickerplant-logging-and-durability-strategy.md` (TP logging, single file)
- `adr-004-real-time-analytics-computation.md` (RTE bucketed design)
- `adr-005-telemetry-and-metrics-aggregation-strategy.md` (ephemeral telemetry)
- `adr-009-order-book-architecture.md` (quote data, L5 depth)
- `adr-010-log-management-and-lifecycle.md` (log management details)
- `adr-011-financial-machine-learning.md` (MLE bar recovery)
