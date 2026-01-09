# Binance WebSocket API — Feed Handler Specification

## Purpose

This document defines the external WebSocket API contract between Binance and the Feed Handler.
It describes connection endpoints, stream formats, message envelopes, payloads, limits, and
operational requirements.

This document is **descriptive**, not interpretive.
Transformation, normalisation, schema mapping, and timestamp semantics are defined elsewhere (see ADR-001).

---

## Notation

| Term | Definition |
|------|------------|
| WSS | WebSocket Secure |
| JSON | JavaScript Object Notation |
| ms | Milliseconds |

---

## Base Endpoints

The Binance WebSocket base endpoints are:

| Endpoint | Description |
|----------|-------------|
| `wss://stream.binance.com:9443` | Primary endpoint |
| `wss://stream.binance.com:443` | Alternative port |
| `wss://data-stream.binance.vision` | Market data only (backup) |

All endpoints provide equivalent market data streams.

---

## Connection Lifecycle

### Connection Duration

| Parameter | Value |
|-----------|-------|
| Maximum connection duration | 24 hours |
| Automatic disconnection | After 24 hours |

Connections are disconnected automatically after 24 hours. The feed handler must implement reconnection logic.

### Keepalive (Ping/Pong)

| Parameter | Value |
|-----------|-------|
| Server ping interval | Every 3 minutes |
| Pong response requirement | Must respond to ping frames |
| Timeout | Connection closed if no pong received within 10 minutes |

The WebSocket server sends a ping frame every 3 minutes. If no pong frame is received within 10 minutes, the connection is disconnected.

**Implementation note:** Most WebSocket libraries handle ping/pong automatically at the protocol level.

### Disconnection Reasons

| Reason | Description |
|--------|-------------|
| 24-hour limit | Automatic disconnect after maximum duration |
| Ping timeout | No pong response within 10 minutes |
| Network failure | Network connectivity issues |
| Server maintenance | Binance infrastructure updates |
| Rate limit exceeded | Too many requests/connections |

---

## Stream Access Modes

### Raw Streams

Raw streams are accessed by connecting directly to a stream URL:
```
wss://stream.binance.com:9443/ws/<streamName>
```

**Example (single stream):**
```
wss://stream.binance.com:9443/ws/btcusdt@trade
```

### Combined Streams

Multiple streams can be accessed via a single connection:
```
wss://stream.binance.com:9443/stream?streams=<streamName1>/<streamName2>/<streamName3>
```

**Example (multiple streams):**
```
wss://stream.binance.com:9443/stream?streams=btcusdt@trade/ethusdt@trade
```

Combined stream messages are wrapped in an envelope:
```json
{
  "stream": "btcusdt@trade",
  "data": {
    // payload
  }
}
```

### Stream Name Format

| Component | Format | Example |
|-----------|--------|---------|
| Symbol | Lowercase | `btcusdt` |
| Stream type | Lowercase | `trade` |
| Full stream name | `<symbol>@<streamType>` | `btcusdt@trade` |

**Important:** All symbols must be **lowercase** in stream names.

---

## Live Subscribing/Unsubscribing

Streams can be subscribed/unsubscribed dynamically via WebSocket messages.

### Subscribe Request
```json
{
  "method": "SUBSCRIBE",
  "params": [
    "btcusdt@trade",
    "ethusdt@trade"
  ],
  "id": 1
}
```

### Subscribe Response

**Success:**
```json
{
  "result": null,
  "id": 1
}
```

**Error:**
```json
{
  "error": {
    "code": 2,
    "msg": "Invalid request"
  },
  "id": 1
}
```

### Unsubscribe Request
```json
{
  "method": "UNSUBSCRIBE",
  "params": [
    "btcusdt@trade"
  ],
  "id": 2
}
```

### List Subscriptions
```json
{
  "method": "LIST_SUBSCRIPTIONS",
  "id": 3
}
```

**Response:**
```json
{
  "result": [
    "btcusdt@trade",
    "ethusdt@trade"
  ],
  "id": 3
}
```

---

## Trade Streams

Trade streams push raw trade information. Each trade has a unique buyer and seller.

### Stream Name
```
<symbol>@trade
```

**Examples:**
- `btcusdt@trade`
- `ethusdt@trade`

### Update Speed

Real-time (as trades occur).

### Payload Format
```json
{
  "e": "trade",
  "E": 1672515782136,
  "s": "BTCUSDT",
  "t": 12345,
  "p": "0.001",
  "q": "100",
  "T": 1672515782136,
  "m": true,
  "M": true
}
```

### Field Definitions

| Field | Type | Description |
|-------|------|-------------|
| `e` | string | Event type (always `"trade"`) |
| `E` | long | Event time (milliseconds since epoch) |
| `s` | string | Symbol (uppercase) |
| `t` | long | Trade ID |
| `p` | string | Price (decimal as string) |
| `q` | string | Quantity (decimal as string) |
| `T` | long | Trade time (milliseconds since epoch) |
| `m` | boolean | Is the buyer the market maker? |
| `M` | boolean | Ignore (deprecated field) |

### Field Notes

| Field | Notes |
|-------|-------|
| `E` (Event time) | Server-side timestamp when event was generated |
| `T` (Trade time) | Timestamp when trade was executed |
| `E` vs `T` | Often equal for trades; both are server-side |
| `t` (Trade ID) | Unique per symbol; not guaranteed contiguous |
| `p`, `q` | Strings to preserve decimal precision |
| `m` | `true` = buyer is maker (sell order was on book); `false` = buyer is taker |
| `M` | Deprecated; always ignore |

### Example Messages

**BTC trade:**
```json
{
  "e": "trade",
  "E": 1702900800000,
  "s": "BTCUSDT",
  "t": 3245678901,
  "p": "43521.50",
  "q": "0.01234",
  "T": 1702900800000,
  "m": false,
  "M": true
}
```

**ETH trade:**
```json
{
  "e": "trade",
  "E": 1702900800123,
  "s": "ETHUSDT",
  "t": 2876543210,
  "p": "2285.75",
  "q": "1.5000",
  "T": 1702900800123,
  "m": true,
  "M": true
}
```

---

## Other Available Streams (Reference Only)

The following streams are available but **not used in this project**:

| Stream | Name Format | Description |
|--------|-------------|-------------|
| Aggregate Trade | `<symbol>@aggTrade` | Aggregated trades at same price |
| Kline/Candlestick | `<symbol>@kline_<interval>` | OHLC candlesticks |
| Mini Ticker | `<symbol>@miniTicker` | 24hr mini ticker |
| Ticker | `<symbol>@ticker` | 24hr full ticker |
| Book Ticker | `<symbol>@bookTicker` | Best bid/ask |
| Partial Book Depth | `<symbol>@depth<levels>` | Top N bids/asks |
| Diff. Depth | `<symbol>@depth` | Order book updates |

---

## Connection Limits

| Limit | Value |
|-------|-------|
| Maximum connections per IP | 300 |
| Maximum streams per connection | 1024 |
| Maximum subscriptions per connection | 1024 |
| Connection rate limit | 5 connections per second per IP |

### Exceeding Limits

If connection limits are exceeded:
- New connection attempts are rejected
- Existing connections may be terminated
- IP may be temporarily banned

---

## Message Rate Limits

| Limit | Value |
|-------|-------|
| Outbound messages (client → server) | 5 per second |
| Inbound messages (server → client) | No explicit limit |

**Note:** The 5 messages/second limit applies to client-sent messages (subscribe, unsubscribe, etc.), not to incoming market data.

---

## Error Messages

### Error Response Format
```json
{
  "error": {
    "code": <error_code>,
    "msg": "<error_message>"
  },
  "id": <request_id>
}
```

### Error Codes

| Code | Description |
|------|-------------|
| 0 | Unknown property |
| 1 | Invalid value type |
| 2 | Invalid request |
| 3 | Invalid JSON |

### Example Error
```json
{
  "error": {
    "code": 2,
    "msg": "Invalid request: property 'params' must be an array"
  },
  "id": 1
}
```

---

## Reconnection Guidance

### Reconnection Behavior

| Aspect | Behavior |
|--------|----------|
| Historical replay | **Not provided** on reconnect |
| Catch-up mechanism | None |
| Resume point | Stream resumes from reconnect time |
| Missed data | Permanently lost |

**Critical:** Binance does not replay missed messages on WebSocket reconnect. Data during disconnection is permanently lost.

### Recommended Reconnection Strategy

| Step | Action |
|------|--------|
| 1 | Detect disconnection |
| 2 | Wait with exponential backoff (start 1s, max 30s) |
| 3 | Reconnect to endpoint |
| 4 | Resubscribe to streams |
| 5 | Resume processing (accept data gap) |

### Backoff Schedule Example

| Attempt | Wait Time |
|---------|-----------|
| 1 | 1 second |
| 2 | 2 seconds |
| 3 | 4 seconds |
| 4 | 8 seconds |
| 5 | 16 seconds |
| 6+ | 30 seconds (max) |

---

## Data Considerations

### Timestamp Precision

| Field | Precision | Unit |
|-------|-----------|------|
| `E` (Event time) | Milliseconds | ms since Unix epoch |
| `T` (Trade time) | Milliseconds | ms since Unix epoch |

### Clock Source

- All timestamps are from Binance server clocks
- No synchronization with client clocks
- Clock skew between Binance and local system is expected

### Trade ID Characteristics

| Aspect | Behavior |
|--------|----------|
| Uniqueness | Unique per symbol |
| Ordering | Generally increasing |
| Contiguity | **Not guaranteed** to be contiguous |
| Scope | Per symbol (not global) |

---

## Project Scope

### In Scope

| Item | Stream | Symbols |
|------|--------|---------|
| Trade streams | `<symbol>@trade` | `btcusdt`, `ethusdt` |

### Out of Scope

| Item | Reason |
|------|--------|
| Aggregate trades | Not required |
| Klines/Candlesticks | Not required |
| Order book depth | Not required |
| User data streams | Requires authentication |
| REST API | Not used for market data |

---

## Links / References

- [Binance WebSocket Streams Documentation](https://developers.binance.com/docs/binance-spot-api-docs/web-socket-streams)
- [Binance API General Info](https://developers.binance.com/docs/binance-spot-api-docs/general-info)
- `adr-001-timestamps-and-latency-measurement.md` (timestamp field mapping)
- `adr-002-feed-handler-to-kdb-ingestion-path.md` (connection handling)
- `adr-006-recovery-and-replay-strategy.md` (reconnection, data gaps)