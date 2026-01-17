# ADR-006: Recovery and Replay Strategy

## Status
Accepted (Updated 2026-01-18)

## Date
Original: 2025-12-17
Updated: 2025-12-29, 2026-01-06, 2026-01-07, 2026-01-09, 2026-01-11, 2026-01-16, 2026-01-17, 2026-01-18

## Context

In a typical production-grade KDB-X architecture, recovery and replay are achieved via:
- Durable tickerplant logs
- Replay of persisted data on restart
- Deterministic rebuild of downstream state (WDB, MLE)

The reference architecture describes multiple recovery patterns:

| Pattern | Description |
|---------|-------------|
| Recovery via TP log | Replay persisted log to rebuild WDB/MLE state |
| Recovery via WDB query | Component queries WDB for recent data |
| Recovery via on-disk cache | Load snapshot, replay delta |
| Upstream replay | Source provides missed data on reconnect |

This project is explicitly **exploratory and incremental** in nature.

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

The project implements a **production-grade recovery model** with automatic log replay on startup using a **single log file** that preserves chronological order across all tables.

### Recovery Capability Summary

| Level | Capability | Status |
|-------|------------|--------|
| TP log format | `-11!` compatible binary logs | ✓ Implemented |
| Single log file | Temporal consistency across tables | ✓ Implemented |
| WDB auto-replay | Automatic replay on startup | ✓ Implemented |
| MLE auto-replay | Automatic replay on startup | ✓ Implemented |
| Log management | Cleanup, diagnostics, verification | ✓ Implemented (on-demand) |
| Connection resilience | Auto-reconnect with exponential backoff | ✓ Implemented |
| Memory management | RDB retention cleanup | ✓ Implemented |
| Safe exit handling | WDB preserves data on exit | ✓ Implemented |
| FH recovery from Binance | Replay missed data from exchange | ✗ Not available |
| Gap detection | Identify missed events via sequence numbers | ✓ Available |
| Gap recovery | Fill gaps with missed data | ✗ Not implemented |

**Note:** RTE, TEL, and RDB subscribe to Chained TP and rebuild state from live data. They do not replay from TP logs.

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

### WDB Automatic Recovery

**On WDB startup:**
1. Check if today's log file exists and has content
2. If log exists, replay it using `-11!` streaming execute
3. Subscribe to TP for live updates
4. Live data appends to replayed data (no gaps, no duplicates)

```q
.wdb.replay:{[d]
  if[d ~ (::); d:.z.D];
  logFile:.wdb.logFile[d];
  if[not .wdb.logExists[logFile]; :0j];
  -1 "WDB: Replaying ",string[logFile];
  n:-11!logFile;
  -1 "WDB: Replayed ",string[n]," chunks";
  n
  };
```

### WDB Safe Exit Handling

**On unexpected exit (SIGTERM, error):**
- WDB flushes all in-memory data to disk (TMPSAVE)
- Data is preserved for recovery
- Logs location of preserved data

```q
.z.exit:{[x]
  -1 "WDB: Exit signal - emergency flush...";
  / Flush remaining data to TMPSAVE
  {.[` sv TMPSAVE,x,`;();,;.Q.en[.wdb.cfg.hdbDir]`. x]} each t;
  -1 "WDB: Data preserved in ",string[TMPSAVE];
  };
```

**Previous behavior (fixed):** Exit handler was destroying data by overwriting with empty tables.

### MLE Automatic Recovery

**On MLE startup:**
1. Check if today's log file exists and has content
2. If log exists, replay it using `-11!` streaming execute
3. Run cleanup to remove stale bars outside retention window
4. Subscribe to TP for live updates

### RDB Memory Management

**On RDB startup and runtime:**
- Configurable retention period (default: 60 minutes)
- Timer-based cleanup removes old rows
- Memory monitoring with warnings at thresholds

```q
.rdb.cfg.retentionMin:60;         / Keep 1 hour of data
.rdb.cfg.memWarningMB:1000;       / Warn at 1GB
.rdb.cfg.memCriticalMB:2000;      / Emergency cleanup at 2GB

.rdb.cleanup:{[]
  cutoff:.z.p - .rdb.cfg.retentionNs;
  delete from `trade_binance where time < cutoff;
  delete from `quote_binance where time < cutoff;
  };
```

### Connection Resilience

All kdb+ processes implement resilient connection handling with exponential backoff:

**Connection state tracking:**
```q
.proc.conn.handle:0N;
.proc.conn.state:`disconnected;        / disconnected, connecting, connected
.proc.conn.lastAttempt:0Np;
.proc.conn.retryCount:0;
```

**Exponential backoff configuration:**
```q
.proc.conn.cfg.baseDelayMs:1000;       / 1 second initial delay
.proc.conn.cfg.maxDelayMs:30000;       / 30 second max delay
.proc.conn.cfg.backoffMultiplier:1.5;  / 1.5x multiplier
```

**Connection function pattern:**
```q
.proc.connect:{[]
  if[not null .proc.conn.handle; :1b];       / Already connected
  if[not .proc.canRetry[]; :0b];             / Backoff not elapsed
  
  .proc.conn.state:`connecting;
  .proc.conn.lastAttempt:.z.p;
  
  h:@[hopen; `$"::",string[port]; {0N}];
  
  if[null h;
    .proc.conn.retryCount+:1;
    .proc.conn.state:`disconnected;
    :0b
  ];
  
  / Subscribe and complete connection
  .proc.conn.handle:h;
  .proc.conn.state:`connected;
  .proc.conn.retryCount:0;
  1b
  };
```

**Disconnect handling:**
```q
.z.pc:{[h]
  if[h = .proc.conn.handle;
    .proc.conn.handle:0N;
    .proc.conn.state:`disconnected;
    .proc.conn.retryCount:0;    / Reset for fresh backoff
  ];
  };
```

**Timer-based reconnection:**
```q
.z.ts:{[]
  if[null .proc.conn.handle; .proc.connect[]];
  / ... other timer logic ...
  };
```

### Components That Do NOT Replay

| Component | Recovery Method |
|-----------|-----------------|
| CTP | No state to recover (passes through) |
| RTE | Subscribes to CTP, rebuilds from live data |
| TEL | Subscribes to CTP, rebuilds from live data |
| RDB | Subscribes to CTP, rebuilds from live data + retention cleanup |

These components have no persistent state and rebuild naturally from the data stream.

### Failure Scenario Matrix

| Failure | Data Impact | Recovery Action | Gap |
|---------|-------------|-----------------|-----|
| FH disconnects from Binance | Events during disconnect lost | FH reconnects; resumes live stream | Permanent |
| FH disconnects from TP | Events during disconnect lost | FH reconnects to TP; resumes publishing | Permanent |
| FH crash | Events during downtime lost | Restart FH; reconnect to Binance | Permanent |
| TP crash | WDB/MLE/CTP lose subscription | Restart TP; subscribers reconnect | Recoverable via log |
| WDB crash | In-memory flushed to TMPSAVE | Restart WDB; **auto-replays from log** | Recoverable |
| MLE crash | Bar state lost | Restart MLE; **auto-replays from log** | Recoverable |
| CTP crash | RTE/TEL/RDB lose subscription | Restart CTP; downstream reconnects | No data loss (transient) |
| RTE crash | Analytics state lost | Restart RTE; reconnects to CTP | Rebuilds from live data |
| TEL crash | Telemetry state lost | Restart TEL; reconnects to CTP | Rebuilds from live data |
| RDB crash | Query data lost | Restart RDB; reconnects to CTP | Rebuilds from live data |

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
| Component restart (WDB/MLE) | None | **Auto-replay from log** |
| Component restart (RTE/TEL/RDB) | None | **Rebuilds from live data** |
| Full system restart | None (same day) | **Auto-replay from log** |
| Log file deleted | Historical data | Not recoverable |
| WDB unexpected exit | None | **Data flushed to TMPSAVE** |

## Rationale

This approach was selected because:

**Single log file:**
- Preserves chronological order across all tables
- Enables point-in-time recovery with consistent state
- Matches standard kdb+tick pattern

**Automatic replay (WDB/MLE):**
- Zero manual intervention on restart
- Production-grade reliability
- Simple implementation using `-11!` streaming execute

**Connection resilience:**
- Processes survive transient upstream failures
- Exponential backoff prevents thundering herd
- Timer-based reconnection is simple and robust

**Safe exit handling:**
- WDB preserves data on unexpected exit
- No data loss from crashes or SIGTERM
- Emergency flush logs data location

**RDB memory management:**
- Prevents unbounded memory growth
- Configurable retention period
- Memory monitoring with warnings

**No replay for RTE/TEL/RDB:**
- These subscribe to Chained TP
- Rebuild naturally from batched data
- Simpler architecture

## Consequences

### Positive

- **Zero data loss** on WDB/MLE restart (same day)
- **Zero data loss** on WDB unexpected exit (TMPSAVE)
- **Correct temporal ordering** during replay
- Automatic recovery without operator intervention
- Fast startup even with large logs
- RTE/TEL/RDB rebuild naturally from CTP
- Connection resilience throughout the pipeline
- RDB memory bounded by retention cleanup

### Negative / Trade-offs

- Log files require disk space (~70MB/hour typical)
- Replay doesn't recover gaps from FH downtime
- RTE/TEL/RDB lose state on restart (acceptable)
- Connection resilience adds complexity

These trade-offs are acceptable for a production-grade system.

## Links / References

- `reference/kdbx-real-time-architecture-reference.md`
- `adr-001-timestamps-and-latency-measurement.md` (sequence numbers, timing fields)
- `adr-003-tickerplant-logging-and-durability-strategy.md` (TP logging, single file)
- `adr-004-real-time-analytics-computation.md` (RTE state)
- `adr-005-telemetry-and-metrics-aggregation-strategy.md` (ephemeral telemetry)
- `adr-008-error-handling-strategy.md` (connection resilience details)
- `adr-010-log-management-and-lifecycle.md` (log management details)
- `adr-011-financial-machine-learning.md` (MLE bar recovery)
