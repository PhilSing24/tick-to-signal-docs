# ADR-008: Error Handling Strategy

## Status
Accepted (Updated 2026-01-18)

## Date
Original: 2025-12-18
Updated: 2026-01-03, 2026-01-06, 2026-01-11, 2026-01-14, 2026-01-16, 2026-01-17, 2026-01-18

## Context

The system consists of multiple components communicating via network:

```
Binance Trade Stream ──WebSocket──► Trade FH ──┬──IPC──► TP ──IPC──► WDB
                                               │              ──IPC──► MLE
Binance Depth Stream ──WebSocket──► Quote FH ──┘              ──IPC──► CTP
         ▲                                                           │
         └──REST── (snapshot)                           ┌────────────┤
                                                        ▼            ▼
                                                       RTE   TEL    RDB
```

Each component can experience failures:

| Component | Potential Failures |
|-----------|-------------------|
| Trade FH | WebSocket disconnect, JSON parse error, IPC failure |
| Quote FH | WebSocket disconnect, REST failure, sequence gap, IPC failure |
| Tickerplant | Subscriber disconnect, invalid message, publish failure |
| WDB | TP connection loss, timer error, HDB write error |
| MLE | TP connection loss, computation error |
| CTP | TP connection loss, subscriber disconnect |
| RTE | CTP connection loss, computation error |
| TEL | CTP connection loss, computation error |
| RDB | CTP connection loss, query error |

A decision is required on how errors are handled across the system.

## Notation

| Acronym | Definition |
|---------|------------|
| CTP | Chained Tickerplant (batched publisher) |
| FH | Feed Handler |
| IPC | Inter-Process Communication |
| L2 | Level 2 market depth (5 price levels per side) |
| MLE | Machine Learning Engine |
| RDB | Real-time Database (query-only) |
| RTE | Real-Time Engine |
| TEL | Telemetry Process |
| TP | Tickerplant |
| WDB | Write-only Database (intraday writedown to HDB) |

## Decision

The system adopts a **fail-fast with structured logging** error handling strategy appropriate for an exploratory project with production-quality observability.

### Philosophy

| Principle | Rationale |
|-----------|-----------|
| Fail fast | Surface problems immediately; don't hide failures |
| Log clearly | Make failures visible and diagnosable |
| Retry strategically | Exponential backoff for all connections |
| Independent restart | Each process can be restarted without affecting others |
| Structured logging | Use spdlog (C++) and standard q logging for consistent output |
| Graceful degradation | Continue in degraded mode when upstream unavailable |

### Connection Resilience Pattern

All kdb+ processes implement resilient connection handling with exponential backoff:

**State tracking:**
```q
.proc.conn.handle:0N;                    / Connection handle
.proc.conn.state:`disconnected;          / disconnected, connecting, connected
.proc.conn.lastAttempt:0Np;              / Time of last connection attempt
.proc.conn.retryCount:0;                 / Failed attempts counter
```

**Backoff configuration:**
```q
.proc.conn.cfg.baseDelayMs:1000;         / 1 second initial delay
.proc.conn.cfg.maxDelayMs:30000;         / 30 second maximum delay
.proc.conn.cfg.backoffMultiplier:1.5;    / 1.5x multiplier per retry
```

**Backoff calculation:**
```q
.proc.conn.getBackoffMs:{[]
  delay:.proc.conn.cfg.baseDelayMs * prd .proc.conn.retryCount#.proc.conn.cfg.backoffMultiplier;
  .proc.conn.cfg.maxDelayMs & `long$delay
  };
```

**Connection function (never throws):**
```q
.proc.connect:{[]
  if[not null .proc.conn.handle; :1b];              / Already connected
  if[not .proc.canRetry[]; :0b];                    / Backoff not elapsed
  
  .proc.conn.state:`connecting;
  .proc.conn.lastAttempt:.z.p;
  
  h:@[hopen; `$"::",string[port]; {-1 "Connection failed: ",x; 0N}];
  
  if[null h;
    .proc.conn.retryCount+:1;
    .proc.conn.state:`disconnected;
    -1 "Will retry in ",string[.proc.conn.getBackoffMs[]],"ms";
    :0b
  ];
  
  / Subscribe to upstream...
  .proc.conn.handle:h;
  .proc.conn.state:`connected;
  .proc.conn.retryCount:0;
  1b
  };
```

**Disconnect handler:**
```q
.z.pc:{[h]
  if[h = .proc.conn.handle;
    -1 "Upstream disconnected";
    .proc.conn.handle:0N;
    .proc.conn.state:`disconnected;
    .proc.conn.retryCount:0;    / Reset for fresh backoff sequence
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

### Standardized Health Check

All kdb+ processes implement a consistent `.health[]` function:

```q
.health:{[]
  st:$[.proc.conn.state = `connected; `ok;
       .proc.conn.state = `connecting; `degraded;
       `disconnected];
  
  `process`port`uptime`status`connState`retryCount`memMB!(
    `procname;
    .proc.cfg.port;
    `second$.z.p - .proc.startTime;
    st;
    .proc.conn.state;
    .proc.conn.retryCount;
    (`long$.Q.w[][`used]) % 1000000
  )
  };
```

### Error Categories

| Category | Examples | Response |
|----------|----------|----------|
| **Fatal** | Cannot load config, port unavailable | Log and exit |
| **Connection** | Lost connection to peer | Log, enter degraded mode, retry with backoff |
| **Transient** | Parse error, invalid message | Log and skip message |
| **Recoverable** | Sequence gap, book invalid | Log, enter recovery state |
| **Silent** | Expected disconnects | Handle gracefully (no error log) |

### Per-Component Strategy

#### Trade Feed Handler (C++)

| Error | Category | Handling |
|-------|----------|----------|
| Cannot load config | Fatal | Log, exit with code 1 |
| No symbols configured | Fatal | Log, exit with code 1 |
| Cannot connect to Binance | Connection | Log, retry with exponential backoff |
| Cannot connect to TP | Connection | Log, retry with exponential backoff |
| WebSocket disconnect | Connection | Log, reconnect with backoff |
| TP connection lost | Connection | Log, reconnect to TP |
| JSON parse error | Transient | Log warning, skip message |
| Missing required field | Transient | Log warning, skip message |

#### Quote Feed Handler (C++)

| Error | Category | Handling |
|-------|----------|----------|
| Cannot load config | Fatal | Log, exit with code 1 |
| Cannot connect to Binance | Connection | Log, retry with backoff |
| Cannot connect to TP | Connection | Log, retry with backoff |
| WebSocket disconnect | Connection | Log, reconnect, books reset |
| REST snapshot failure | Recoverable | Log, retry with backoff |
| Sequence gap detected | Recoverable | Log, transition to INVALID, re-sync |
| Book in INVALID state | Recoverable | Publish invalid L2 quote (isValid=0b) |

#### Tickerplant (q)

| Error | Category | Handling |
|-------|----------|----------|
| Port already in use | Fatal | Log, exit |
| Invalid table name | Transient | Signal error to caller |
| Subscriber disconnect | Silent | Remove from subscribers via `.z.pc` |
| Log write failure | Transient | Log error, continue publishing |

#### WDB (q)

| Error | Category | Handling |
|-------|----------|----------|
| Cannot connect to TP | Connection | Log, retry with exponential backoff |
| TP connection lost | Connection | Log, retry on timer |
| Timer callback error | Transient | Log, continue |
| HDB write failure | Recoverable | Log, retry next flush |
| Unexpected exit | Recoverable | Flush data to TMPSAVE (see ADR-006) |

#### MLE (q)

| Error | Category | Handling |
|-------|----------|----------|
| Cannot connect to TP | Connection | Log, retry with exponential backoff |
| TP connection lost | Connection | Log, retry on timer |
| Bar computation error | Transient | Log, skip bar |

#### CTP (q)

| Error | Category | Handling |
|-------|----------|----------|
| Cannot connect to TP | Connection | Log, retry with exponential backoff |
| TP connection lost | Connection | Log, retry on timer |
| Subscriber disconnect | Silent | Remove via `.z.pc` |

#### RTE, TEL, RDB (q)

| Error | Category | Handling |
|-------|----------|----------|
| Cannot connect to CTP | Connection | Log, retry with exponential backoff |
| CTP connection lost | Connection | Log, retry on timer |
| Computation error | Transient | Log, skip |
| Memory threshold (RDB) | Recoverable | Log warning, emergency cleanup |

### Graceful Shutdown

All components handle SIGTERM for graceful shutdown:

**C++ (signal handler):**
```cpp
signal(SIGTERM, signalHandler);
signal(SIGINT, signalHandler);
```

**kdb+ (.z.exit):**
```q
.z.exit:{[x]
  -1 "Shutting down...";
  / Cleanup logic (e.g., WDB flushes to TMPSAVE)
  };
```

**stop.sh:** Sends SIGTERM first, waits, then SIGKILL if needed.

### Connection Resilience Summary

| Process | Upstream | Resilient | Backoff |
|---------|----------|-----------|---------|
| WDB | TP:5010 | ✓ | Exponential |
| MLE | TP:5010 | ✓ | Exponential |
| CTP | TP:5010 | ✓ | Exponential |
| RTE | CTP:5014 | ✓ | Exponential |
| TEL | CTP:5014 | ✓ | Exponential |
| RDB | CTP:5014 | ✓ | Exponential |

All processes:
- Start in degraded mode if upstream unavailable
- Retry with exponential backoff
- Resume normal operation when connection restored
- Report connection state via `.health[]`

## Rationale

This approach was selected because:

- **Fail-fast** surfaces problems immediately for debugging
- **Structured logging** makes issues diagnosable
- **Independent processes** allow partial restart
- **Exponential backoff** prevents thundering herd and resource exhaustion
- **Timer-based reconnection** is simple and effective
- **State machine** handles book recovery systematically
- **Graceful degradation** allows processes to continue during transient failures
- **Standardized health checks** provide consistent monitoring interface

## Consequences

### Positive

- Clear visibility into failures via logging
- Independent process restart capability
- Systematic book recovery on gaps
- Graceful degradation (continue in degraded mode)
- Auto-reconnection with exponential backoff throughout pipeline
- Standardized `.health[]` for monitoring
- WDB preserves data on unexpected exit

### Negative / Trade-offs

- Log volume can be high during issues
- Some errors require operator intervention
- Connection resilience adds complexity

These trade-offs are acceptable for the current phase.

## Links / References

- `adr-001-timestamps-and-latency-measurement.md` (sequence numbers)
- `adr-002-feed-handler-to-kdb-ingestion-path.md` (connection architecture)
- `adr-003-tickerplant-logging-and-durability-strategy.md` (logging durability)
- `adr-006-recovery-and-replay-strategy.md` (replay on restart, WDB safe exit)
- `adr-009-order-book-architecture.md` (book state machine)
