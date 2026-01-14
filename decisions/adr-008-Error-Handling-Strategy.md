# ADR-008: Error Handling Strategy

## Status
Accepted (Updated 2026-01-14)

## Date
Original: 2025-12-18
Updated: 2026-01-03, 2026-01-06, 2026-01-11, 2026-01-14

## Context

The system consists of multiple components communicating via network:

```
Binance Trade Stream ──WebSocket──► Trade FH ──┬──IPC──► TP ──IPC──► RDB
                                               │              ──IPC──► RTE
Binance Depth Stream ──WebSocket──► Quote FH ──┘              ──IPC──► TEL
         ▲                                                         │
         └──REST── (snapshot)                          queries ────┴──► RDB, RTE
```

Each component can experience failures:

| Component | Potential Failures |
|-----------|-------------------|
| Trade FH | WebSocket disconnect, JSON parse error, IPC failure |
| Quote FH | WebSocket disconnect, REST failure, sequence gap, IPC failure |
| Tickerplant | Subscriber disconnect, invalid message, publish failure |
| RDB | TP connection loss, timer error, query error |
| RTE | TP connection loss, computation error |
| TEL | TP connection loss, query error, timer error |

The project is explicitly exploratory (ADR-003, ADR-006):
- TP logging provides durability
- Manual replay provides recovery
- Focus is on understanding real-time behaviour

A decision is required on how errors are handled across the system.

## Notation

| Acronym | Definition |
|---------|------------|
| FH | Feed Handler |
| IPC | Inter-Process Communication |
| L5 | Level 5 (top 5 price levels) |
| RDB | Real-Time Database |
| RTE | Real-Time Engine |
| TEL | Telemetry Process |
| TP | Tickerplant |

## Decision

The system adopts a **fail-fast with structured logging** error handling strategy appropriate for an exploratory project with production-quality observability.

### Philosophy

| Principle | Rationale |
|-----------|-----------|
| Fail fast | Surface problems immediately; don't hide failures |
| Log clearly | Make failures visible and diagnosable |
| Retry strategically | TP/Binance reconnect with backoff; don't retry silently elsewhere |
| Independent restart | Each process can be restarted without affecting others |
| Structured logging | Use spdlog for consistent, leveled, timestamped logging |

### Restart vs Reconnection

These are distinct concepts:

| Concept | Meaning | Status |
|---------|---------|--------|
| **Independent restart** | Can restart one process without restarting others | ✓ Supported |
| **Auto-reconnection (FH→Binance)** | Feed handler reconnects to Binance WebSocket | ✓ Implemented |
| **Auto-reconnection (FH→TP)** | Feed handler reconnects to TP | ✓ Implemented |
| **Auto-reconnection (q→TP)** | kdb+ processes reconnect to TP subscription | Manual restart required |
| **Auto-reconnection (TEL→RDB/RTE)** | TEL query handles reconnect on failure | ✓ Implemented (ADR-005) |

Independent restart is a benefit of the separate-process architecture (ADR-002). Auto-reconnection in feed handlers is implemented with exponential backoff. TEL maintains persistent IPC handles to RDB/RTE that automatically reconnect on query failure (see ADR-005).

### Error Categories

Errors are categorised by severity and response:

| Category | Examples | Response |
|----------|----------|----------|
| **Fatal** | Cannot load config, port unavailable | Log and exit |
| **Connection** | Lost connection to peer | Log and attempt reconnect with backoff |
| **Transient** | Parse error, invalid message | Log and skip message |
| **Recoverable** | Sequence gap, book invalid | Log, enter recovery state |
| **Silent** | Expected disconnects | Handle gracefully (no error log) |

### Per-Component Strategy

#### Trade Feed Handler (C++)

**Implementation:** `cpp/src/trade_feed_handler.cpp`

| Error | Category | Handling | Implementation |
|-------|----------|----------|----------------|
| Cannot load config | Fatal | Log, exit with code 1 | `config.load()` returns false |
| No symbols configured | Fatal | Log, exit with code 1 | Check `config.symbols.empty()` |
| Cannot connect to Binance | Connection | Log, retry with exponential backoff | `runWebSocketLoop()` try/catch |
| Cannot connect to TP | Connection | Log, retry with exponential backoff | `connectToTP()` loop with backoff |
| WebSocket disconnect (Binance) | Connection | Log, reconnect with backoff | Exception in `runWebSocketLoop()` |
| TP connection lost | Connection | Log, reconnect to TP | IPC returns nullptr, call `connectToTP()` |
| JSON parse error | Transient | Log warning, skip message | `doc.Parse()` check, continue |
| Missing required field | Transient | Log warning, skip message | `HasMember()` checks, early return |
| TradeId gap detected | Transient | Log warning, continue | `validateTradeId()` logs, continues |
| IPC send failure | Connection | Log, reconnect TP | Result nullptr triggers reconnect |

**Reconnection backoff (Binance):**
```cpp
int delay = INITIAL_BACKOFF_MS;  // 1000ms
for (int i = 0; i < attempt && delay < MAX_BACKOFF_MS; ++i) {
    delay *= BACKOFF_MULTIPLIER;  // 2x
}
delay = std::min(delay, MAX_BACKOFF_MS);  // Cap at 8000ms
```

**Sequence validation:**
```cpp
void TradeFeedHandler::validateTradeId(const std::string& sym, long long tradeId) {
    auto it = lastTradeId_.find(sym);
    if (it != lastTradeId_.end()) {
        long long last = it->second;
        if (tradeId < last) {
            spdlog::warn("OUT OF ORDER: {} last={} got={}", sym, last, tradeId);
        } else if (tradeId == last) {
            spdlog::warn("DUPLICATE: {} tradeId={}", sym, tradeId);
        } else if (tradeId > last + 1) {
            long long missed = tradeId - last - 1;
            spdlog::warn("Gap: {} missed={} (last={} got={})", sym, missed, last, tradeId);
        }
    }
    lastTradeId_[sym] = tradeId;
}
```

**Signal handling:**
```cpp
static void signalHandler(int signum) {
    spdlog::info("Received {} ({})", sigName, signum);
    if (g_handler) {
        g_handler->stop();  // Thread-safe atomic flag
    }
}
```

#### Quote Feed Handler (C++)

**Implementation:** `cpp/src/quote_feed_handler.cpp`

| Error | Category | Handling | Implementation |
|-------|----------|----------|----------------|
| Cannot load config | Fatal | Log, exit with code 1 | `config.load()` returns false |
| Cannot connect to Binance | Connection | Log, retry with backoff | Same as trade FH |
| Cannot connect to TP | Connection | Log, retry with backoff | Same as trade FH |
| WebSocket disconnect (Binance) | Connection | Log, reconnect, books reset | Exception triggers reconnect |
| TP connection lost | Connection | Log, reconnect to TP | IPC returns nullptr |
| REST snapshot failure | Recoverable | Log, retry with backoff | `restClient_.fetchSnapshot()` error |
| Sequence gap detected | Recoverable | Log, transition to INVALID, re-sync | `OrderBookManager` state machine |
| Book in INVALID state | Recoverable | Publish invalid L5 quote, attempt re-sync | `publishInvalid()` called |
| JSON parse error | Transient | Log warning, skip message | Early return on parse error |
| IPC send failure | Connection | Log, reconnect TP | Result nullptr triggers reconnect |

**Quote handler state transitions on error:**

```
VALID ──sequence gap──► INVALID ──re-sync──► SYNCING ──► VALID
                            │
                            └──publish invalid L5 quote (isValid=0b)
```

**OrderBookManager state tracking:**
```cpp
enum class BookState {
    INIT,      // No data, buffering deltas
    SYNCING,   // Snapshot received, applying buffered deltas
    VALID,     // Book consistent, publishing L5
    INVALID    // Sequence gap detected
};
```

**Error logging in snapshot fetch:**
```cpp
SnapshotData snapshot = restClient_.fetchSnapshot(sym, SNAPSHOT_DEPTH);
if (!snapshot.success) {
    spdlog::error("Snapshot failed for {}: {}", sym, snapshot.error);
    bookMgr_->invalidate(symIdx, "Snapshot fetch failed");
    return;
}
```

#### Tickerplant (q)

**Implementation:** `kdb/tp.q`

| Error | Category | Handling | Implementation |
|-------|----------|----------|----------------|
| Port already in use | Fatal | Log, exit | `\p` fails, q exits |
| Invalid table name | Transient | Signal error to caller | `.u.upd` type check |
| Subscriber disconnect | Silent | Remove from `.u.w` via `.z.pc` | Handled automatically |
| Publish to dead handle | Transient | Caught by `.z.pc`, handle removed | Automatic cleanup |
| Malformed message | Transient | Let q signal error (logged to console) | q error handling |
| Log write failure | Transient | Log error, continue publishing | `@[...]` wrapper |

**Subscriber disconnect handling:**
```q
/ Handle subscriber disconnect gracefully
.z.pc:{[h]
  .u.w:{x except h} each .u.w;
  -1 "Subscriber disconnected: handle ",string h;
  };
```

#### RDB (q)

**Implementation:** `kdb/rdb.q`

| Error | Category | Handling | Implementation |
|-------|----------|----------|----------------|
| Cannot connect to TP | Fatal | Log, exit | `hopen` fails, exit |
| TP connection lost | Connection | Log, process continues (no auto-reconnect) | Connection dies, query fails |
| Timer callback error | Transient | Log, continue (next timer fires) | Protected with `@[...]` |
| Query error | Transient | Log, skip operation | Error caught, logged |

**Protected TP connection:**
```q
.rdb.connect:{[]
  h:@[hopen; `::5010; {-1 "Failed to connect to TP: ",x; 0N}];
  if[null h; '"Cannot connect to tickerplant"];
  / ... subscription ...
  };
```

#### RTE (q)

**Implementation:** `kdb/rte.q`

| Error | Category | Handling | Implementation |
|-------|----------|----------|----------------|
| Cannot connect to TP | Fatal | Log, exit | `hopen` fails, exit |
| TP connection lost | Connection | Log, process continues | Connection dies silently |
| Unknown symbol | Silent | Initialize buffer automatically | Auto-create bucket structure |
| Computation error | Transient | Log, analytics may be stale | Protected computation |

**Auto-initialize unknown symbols:**
```q
/ VWAP add - auto-initialize if needed
.rte.vwap.add:{[s;time;price;qty]
  / Auto-initialize bucket if symbol not seen
  if[not s in exec distinct sym from vwapBuckets;
    -1 "Auto-initializing VWAP for ",string s;
  ];
  / ... add logic ...
  };
```

#### TEL (q)

**Implementation:** `kdb/tel.q`

| Error | Category | Handling | Implementation |
|-------|----------|----------|----------------|
| Cannot connect to TP | Fatal | Log, exit | `hopen` fails, exit |
| TP connection lost | Connection | Log, process continues | Subscription lost |
| Query error (RDB/RTE) | Transient | Log, mark handle invalid, retry next cycle | `.tel.safeQuery` with persistent handles |
| Timer error | Transient | Log, skip bucket | Protected timer callback |

**Persistent handle management (see ADR-005):**
```q
/ Handle dictionary: port -> handle (0N = not connected)
.tel.h:(`int$())!`int$();

/ Get or create persistent handle to a port
/ Uses lazy connection - only connects when needed
.tel.getH:{[p]
  / Return existing valid handle
  if[.tel.hasValidH[p]; :.tel.h[p]];
  
  / Attempt connection with error handling
  h:@[hopen; `$"::",string p; {[port;err] 
    -1 "TEL: Failed to connect to port ",string[port]," - ",err; 
    0N
    }[p]];
  
  / Store result (valid handle or 0N)
  .tel.h[p]:h;
  
  / Log successful connection
  if[not null h; -1 "TEL: Connected to port ",string[p]," (handle ",string[h],")"];
  
  h
  };

/ Safe query using persistent handles
/ Automatically marks handle invalid on error for reconnection on next call
.tel.safeQuery:{[port;query]
  / Get or establish connection
  h:.tel.getH[port];
  if[null h; :()];
  
  / Execute query with error trapping
  res:@[h; query; {[p;q;err] 
    / Log the error with context
    -1 "TEL: Query failed on port ",string[p]," - ",err;
    / Mark handle as invalid for reconnection on next call
    .tel.h[p]:0N;
    `..tel.queryError
    }[port;query]];
  
  / Return empty on error (sentinel value check)
  if[res ~ `..tel.queryError; :()];
  
  res
  };
```

**Benefits of persistent handles:**
- Eliminates TCP handshake overhead per query cycle
- Automatic reconnection on next query after failure
- Handle status visible via `.tel.handleStatus[]`
- Graceful cleanup on shutdown via `.z.exit`

### Logging Strategy

**Implementation: spdlog (C++) and console output (q)**

**C++ Log Levels:**

| Level | Usage | spdlog API |
|-------|-------|------------|
| ERROR | Failures requiring attention | `spdlog::error()` |
| WARN | Recoverable issues | `spdlog::warn()` |
| INFO | Normal operations | `spdlog::info()` |
| DEBUG | Diagnostic detail | `spdlog::debug()` |
| TRACE | Verbose tracing | `spdlog::trace()` |

**Format (spdlog default):**
```
[YYYY-MM-DD HH:MM:SS.mmm] [Component] [LEVEL] Message
```

**Examples:**
```
[2026-01-06 08:59:54.341] [Trade FH] [info] Logger initialized (level: info)
[2026-01-06 08:59:54.344] [Trade FH] [info] Connected to TP (handle 3)
[2026-01-06 08:59:54.645] [Quote FH] [error] Sequence gap detected for BTCUSDT
[2026-01-06 08:59:54.827] [Quote FH] [info] BTCUSDT is now VALID
```

**Configuration (JSON):**
```json
{
    "logging": {
        "level": "info",
        "file": ""  // Empty = console only
    }
}
```

**Q log format (informal):**
```
2026.01.06D09:00:15.123 Connecting to tickerplant on port 5010...
2026.01.06D09:00:15.456 Subscribed to: trade_binance
```

### What IS Implemented

| Scenario | Status | Implementation |
|----------|--------|----------------|
| FH reconnect to Binance | ✓ Implemented | Exponential backoff in `runWebSocketLoop()` |
| FH reconnect to TP | ✓ Implemented | Exponential backoff in `connectToTP()` |
| Signal handling (SIGINT/SIGTERM) | ✓ Implemented | Global handler calls `stop()` |
| Structured logging (C++) | ✓ Implemented | spdlog with levels and timestamps |
| Configuration from JSON | ✓ Implemented | `config.hpp` loads JSON configs |
| Sequence gap detection | ✓ Implemented | `validateTradeId()` in trade FH |
| Quote book state machine | ✓ Implemented | `OrderBookManager` with INIT/SYNCING/VALID/INVALID |
| Health metrics publishing | ✓ Implemented | Both FHs publish every 5 seconds |
| TP subscriber disconnect | ✓ Implemented | `.z.pc` handler |
| TEL persistent handles | ✓ Implemented | `.tel.getH`, `.tel.safeQuery` with auto-reconnect |

### What Is NOT Implemented

Consistent with the exploratory nature:

| Scenario | Status | Rationale |
|----------|--------|-----------|
| Automatic TP reconnect in RDB/RTE/TEL | Not implemented | Manual restart acceptable |
| Automatic recovery on startup | Not implemented | Manual replay via `replay.q` |
| Persistent error logs to file | Optional | File logging configurable but console primary |
| External alerting | Not implemented | Human observation via dashboard |
| Circuit breakers | Not implemented | Overkill for 3 symbols |
| Dead letter queues | Not implemented | Dropped messages acceptable |
| Distributed tracing | Not implemented | Single-host deployment |
| Retry on JSON parse errors | Not implemented | Skip message, continue (fail fast on data) |

### Error Visibility

Errors are made visible through:

| Method | Purpose | Implementation |
|--------|---------|----------------|
| Console output (spdlog) | Immediate visibility during development | Default sinks, colored output |
| Optional file logs | Persistent audit trail | Configurable via JSON |
| Dashboard staleness | `isValid`, `fillPct` indicate data quality | TEL queries |
| Gap detection | `fhSeqNo` gaps observable in RDB | Sequence validation |
| Quote validity | `isValid` in `quote_binance` indicates L5 book state | OrderBookManager |
| Health metrics | FH uptime, message counts, connection state | `health_feed_handler` table |
| Process exit | Fatal errors surface immediately | Exit codes |
| Handle status | TEL connection state visible | `.tel.handleStatus[]` |

### Startup Validation

Each component validates its environment at startup:

| Component | Validation | Implementation |
|-----------|------------|----------------|
| Trade FH | Config loads, symbols present | `config.load()` check |
| Quote FH | Config loads, symbols present | `config.load()` check |
| Both FHs | Can connect to TP | Retry with backoff, exit if stopped |
| TP | Port available, log directory writable | q built-in checks |
| RDB | Can connect to TP, subscription succeeds | `hopen` check |
| RTE | Can connect to TP, subscription succeeds | `hopen` check |
| TEL | Can connect to TP, subscription succeeds | `hopen` check |

Failure during startup = immediate exit (fail fast).

### Configuration Management

**JSON Configuration Files:**

| File | Component | Contents |
|------|-----------|----------|
| `config/trade_feed_handler.json` | Trade FH | Symbols, TP host/port, logging, backoff |
| `config/quote_feed_handler.json` | Quote FH | Symbols, TP host/port, logging, backoff |

**Example Configuration:**
```json
{
    "symbols": ["btcusdt", "ethusdt", "solusdt"],
    "tickerplant": {
        "host": "localhost",
        "port": 5010
    },
    "reconnect": {
        "initial_backoff_ms": 1000,
        "max_backoff_ms": 8000
    },
    "logging": {
        "level": "info",
        "file": ""
    }
}
```

**Config Loading:**
```cpp
FeedHandlerConfig config;
if (!config.load(configPath)) {
    std::cerr << "Failed to load config, exiting\n";
    return 1;
}
```

## Rationale

This approach was selected because:

- **Observability**: Structured logging (spdlog) provides production-quality diagnostics
- **Reliability**: Automatic reconnection prevents manual restarts for transient issues
- **Simplicity**: Complex retry logic avoided where not needed
- **Visibility**: Fail-fast surfaces issues immediately
- **Consistency**: Aligns with ADR-003 (logging) and ADR-006 (manual replay)
- **Appropriate**: Error handling matches project maturity
- **Debuggable**: Clear, timestamped logs aid understanding
- **Independent**: Each process handles its own errors
- **Configurable**: JSON configs allow easy adjustment without recompilation
- **Efficient**: TEL persistent handles minimize IPC overhead (ADR-005)

## Alternatives Considered

### 1. No automatic reconnection (original design)
Updated:
- Frequent manual restarts were painful
- Exponential backoff is industry standard
- Implemented for FH components

### 2. Console output only (no spdlog)
Rejected:
- std::cout/cerr lack timestamps, levels, formatting
- spdlog adds minimal overhead
- Production-quality logging worth the dependency

### 3. Automatic reconnection for q processes
Rejected:
- Complex state management
- Manual restart is explicit and clear
- May be reconsidered for production

### 4. Message validation layer
Rejected:
- Schema is controlled end-to-end
- FH and TP are trusted components
- Validation overhead not justified

### 5. Shared error handling library
Rejected:
- Overkill for current scale
- Each component has different error scenarios
- Would add coupling between components

### 6. Health check HTTP endpoints
Rejected:
- Adds infrastructure complexity
- Dashboard provides sufficient visibility
- No external monitoring system to consume

### 7. Open/close IPC connections each query cycle (TEL)
Rejected (2026-01-07):
- 4 TCP handshakes per 5-second tick
- Unnecessary overhead (~4-12ms per tick)
- File descriptor churn
- Persistent handles are simple and more efficient

## Consequences

### Positive

- Simple, understandable error handling
- Problems surface immediately (fail fast)
- Easy to diagnose issues via structured logs
- Automatic reconnection reduces manual intervention
- Quote handler state machine handles L5 book recovery scenarios
- Independent processes can be restarted independently
- JSON configuration enables easy tuning
- spdlog provides production-quality observability
- Consistent error handling patterns across C++ components
- Health metrics provide operational visibility
- TEL persistent handles eliminate connection overhead

### Negative / Trade-offs

- Manual restart required for q components (RDB/RTE/TEL)
- No automatic recovery on startup (manual replay required)
- Transient errors cause data loss (by design)
- Log format in q is informal (no structured logging)
- spdlog dependency (acceptable - industry standard)
- JSON config dependency (rapidjson already used)
- Reconnection adds slight complexity to FH code
- Persistent handles require explicit management (close on shutdown)

These trade-offs are acceptable for the current phase.

## Implementation Checklist

### Implemented

- [x] Trade FH: Exit on config load failure
- [x] Trade FH: Reconnect to Binance with exponential backoff
- [x] Trade FH: Reconnect to TP with exponential backoff
- [x] Trade FH: Skip malformed JSON messages
- [x] Trade FH: Sequence gap detection with logging
- [x] Trade FH: Signal handling (SIGINT/SIGTERM)
- [x] Trade FH: spdlog integration with levels
- [x] Trade FH: JSON configuration loading
- [x] Trade FH: Health metrics publishing
- [x] Quote FH: All of the above plus:
- [x] Quote FH: REST snapshot error handling
- [x] Quote FH: Sequence gap detection and INVALID state
- [x] Quote FH: State machine (INIT → SYNCING → VALID ↔ INVALID)
- [x] Quote FH: OrderBookManager with per-symbol state tracking
- [x] TP: `.z.pc` handles subscriber disconnect
- [x] TP: `.u.sub` validates table name
- [x] TP: Logging to single file per day (trades + quotes)
- [x] RDB: Protected TP connection with error message
- [x] RDB: Subscription to both trades and quotes
- [x] RTE: Protected TP connection with error message
- [x] RTE: Auto-initialize unknown symbols (VWAP + imbalance)
- [x] TEL: Persistent handles with `.tel.getH`, `.tel.safeQuery`
- [x] TEL: Automatic reconnection on query failure
- [x] TEL: Handle status via `.tel.handleStatus[]`
- [x] TEL: Graceful shutdown via `.z.exit`
- [x] TEL: Subscription to health data

### Recommended Enhancements

- [ ] RDB: Automatic reconnect to TP
- [ ] RTE: Automatic reconnect to TP
- [ ] TEL: Automatic reconnect to TP (subscription)
- [ ] All q: Protected timer callbacks consistently
- [ ] All q: Structured logging (explore log4q or similar)
- [ ] FH: Configurable health publish interval
- [ ] FH: Optional file rotation for logs

### Out of Scope

- [ ] Automatic startup recovery (use manual replay)
- [ ] External log aggregation (e.g., ELK stack)
- [ ] Alerting system integration
- [ ] Health check HTTP endpoints
- [ ] Distributed tracing
- [ ] Circuit breakers

## Future Evolution

| Enhancement | Trigger |
|-------------|---------|
| Automatic q reconnection | Frequent manual restarts become painful |
| Automatic startup recovery | Production deployment |
| External log aggregation | Multiple instances, log analysis |
| Health endpoints | External monitoring integration |
| Alerting | Production deployment |
| Circuit breakers | High message rates, cascading failures |
| Distributed tracing | Multi-host deployment |

## Links / References

- `../kdbx-real-time-architecture-reference.md` (fail-fast principle)
- `adr-001-timestamps-and-latency-measurement.md` (fhSeqNo for gap detection)
- `adr-002-feed-handler-to-kdb-ingestion-path.md` (separate processes, independent restart)
- `adr-003-tickerplant-logging-and-durability-strategy.md` (TP logging)
- `adr-004-real-time-analytics-computation.md` (RTE natural rebuild)
- `adr-005-telemetry-and-metrics-aggregation-strategy.md` (health metrics, persistent handles)
- `adr-006-recovery-and-replay-strategy.md` (manual replay)
- `adr-007-visualisation-and-consumption-strategy.md` (observability via dashboard)
- `adr-009-order-book-architecture.md` (quote handler state machine, OrderBookManager)
- `cpp/include/config.hpp` (JSON configuration)
- `cpp/include/logger.hpp` (spdlog wrapper)
- `cpp/src/trade_feed_handler.cpp` (implementation)
- `cpp/src/quote_feed_handler.cpp` (implementation)
- `kdb/tel.q` (persistent handle implementation)


