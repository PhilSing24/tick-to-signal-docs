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
| L2 | Level 2 market depth (5 price levels per side) |
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

**Note:** Binance recommends a maximum backoff of 30 seconds (see `api-binance.md`). This implementation uses 8 seconds for faster recovery during development, which is acceptable for an exploratory project with low connection volume (3 symbols, single instance). Production systems at scale should consider the longer backoff to avoid rate limiting during widespread outages.

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
| Book in INVALID state | Recoverable | Publish invalid L2 quote, attempt re-sync | `publishInvalid()` called |
| JSON parse error | Transient | Log warning, skip message | Early return on parse error |
| IPC send failure | Connection | Log, reconnect TP | Result nullptr triggers reconnect |

**Quote handler state transitions on error:**

```
VALID ──sequence gap──► INVALID ──re-sync──► SYNCING ──► VALID
                            │
                            └──publish invalid L2 quote (isValid=0b)
```

**OrderBookManager state tracking:**
```cpp
enum class BookState {
    INIT,      // No data, buffering deltas
    SYNCING,   // Snapshot received, applying buffered deltas
    VALID,     // Book consistent, publishing L2
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
