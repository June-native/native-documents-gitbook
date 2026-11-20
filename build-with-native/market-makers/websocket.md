---
description: Stream orderbook updates over a long-lived WebSocket. Same book format and validation as HTTP.
layout:
  width: wide
---

# WebSocket

For continuous pushing, keep a long-lived WebSocket open and stream updates as frames — several connections are allowed for HA, within the cap in [Rules](#rules). The stream is a complete interface of its own — this page describes it in full.

## Semantics

These rules apply to every push frame:

* Each book is a **complete orderbook** for one pair: per-level `price`/`qty` (not cumulative), bids price-descending best-first, asks price-ascending, best bid strictly below best ask; values are decimal strings.
* Push each pair in its assigned direction; EVM addresses must be lowercase, symbols match case-sensitively — a reversed book is rejected (`orientation_mismatch`), a wrong symbol is rejected (`symbol_mismatch`).
* An accepted book replaces your previous book for that pair **in full**; an empty side withdraws that side; both sides empty withdraws the pair; a pair the frame does not name keeps its book.
* Each book carries an absolute `expire_at_ms` (ms Unix, read on Native's clock); the remaining lifetime at receipt must be at least 5 s.
* A frame is **atomic**: either every book passes and is applied, or nothing is applied. The nonce is consumed only when a frame is applied; a failed frame has zero side effects. `nonce` must be strictly greater than the last **applied** one — one shared counter with HTTP.
* The size and rate limits of [Limits](errors-and-limits.md#limits) apply to frames, and reject codes are the ones in [Reject reasons](errors-and-limits.md#reject-reasons).

## Connect

`wss://api.native.org/v1/ws`, with your `Apikey` header on the upgrade request. The IP allowlist applies. Server-side clients only (browsers cannot set custom headers on a WebSocket).

## Send

Frames are JSON text. Push frame — full schema:

```json
{
  "op": "push",
  "nonce": 1731318400124,
  "orderbooks": [
    {
      "chain_id": 56,
      "base":  { "address": "0x698250914c54eb2f1bb7e0eb1e6a24c0f9b1c1a1", "symbol": "MEME" },
      "quote": { "address": "0x55d398326f99059ff775485246999027b3197955", "symbol": "USDT" },
      "bids": [
        { "price": "0.98", "qty": "12000" },
        { "price": "0.97", "qty": "25000" }
      ],
      "asks": [
        { "price": "1.02", "qty": "8000" },
        { "price": "1.03", "qty": "15000" }
      ],
      "expire_at_ms": 1731318406000,
      "book_id": "b-000124"
    }
  ]
}
```

| Field | Type | Required | Description |
| --- | --- | --- | --- |
| `op` | string | yes | `"push"` |
| `nonce` | integer | yes | Per-maker monotonic counter; strictly above the last applied one |
| `orderbooks` | array | yes | 1–20 orderbooks per frame |
| `orderbooks[].chain_id` | integer | yes | EVM chain id of the pair |
| `orderbooks[].base.address` | string | yes | Base token contract, EVM hex; **must be lowercase** |
| `orderbooks[].quote.address` | string | yes | Quote token contract, EVM hex; **must be lowercase** |
| `orderbooks[].base.symbol`, `orderbooks[].quote.symbol` | string | yes | The registered symbols, **case-sensitive** — use the exact strings from the mapping (onboarding sheet / `GET /v1/orderbook`) |
| `orderbooks[].bids`, `orderbooks[].asks` | array | yes | Levels, best price first; may be empty to withdraw that side |
| `orderbooks[].expire_at_ms` | integer | yes | Absolute expiry, ms Unix; at least receive time + 5 s |
| `orderbooks[].book_id` | string | yes | Your id for this book (non-empty, ≤ 64 chars, **unique within a frame**); echoed in the ack and in your statements |

Ping frame — keepalive. Two mechanisms are accepted and both are answered: this app-level frame (the server replies `pong`, echoing an optional `ts` field so you can measure RTT), or a WebSocket protocol-level Ping control frame (the server replies with a Pong control frame automatically):

```json
{ "op": "ping" }
```

## Receive

Ack frame — exactly one per push frame, in push order. Applied:

```json
{
  "type": "ack",
  "nonce": 1731318400124,
  "applied": true,
  "server_time_ms": 1731318400450,
  "trace_id": "01JD8Z9W3C"
}
```

Not applied — one bad book aborts the whole frame:

```json
{
  "type": "ack",
  "nonce": 1731318400125,
  "applied": false,
  "server_time_ms": 1731318400900,
  "trace_id": "01JD8ZA112F",
  "results": [
    {
      "book_id": "b-000124",
      "status": "skipped",
      "reason": "atomic_abort"
    },
    {
      "chain_id": 137,
      "base": "0xbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbb",
      "quote": "0xcccccccccccccccccccccccccccccccccccccccccc",
      "book_id": "b-000125",
      "status": "rejected",
      "reason": "crossed_book",
      "detail": "best bid 1.05 >= best ask 1.02"
    }
  ]
}
```

| Field | Description |
| --- | --- |
| `type` | Always `ack` |
| `nonce` | Echo of the push frame's `nonce` |
| `applied` | `true` — every book passed and was applied, and the nonce was consumed. `false` — nothing was applied and the nonce was **not** consumed |
| `server_time_ms` | Native's clock when the ack was built |
| `trace_id` | Trace id of this frame |
| `results` | Present only when `applied` is `false`; lists only the problem books, in push order, correlated by `book_id` |
| `results[].status` | `rejected` — failed validation (see `reason`); `skipped` — fine, but the frame as a whole was not applied (`atomic_abort`), or the nonce gate failed (`stale_nonce`). `accepted` never appears today; it may appear if partial acceptance is enabled later |
| `results[].reason`, `results[].detail` | On rejected/skipped results. `detail` names the offending level with its values (e.g. `level[3] qty 50 < min 100`). A rejected result repeats the pair identity (`chain_id`, `base`, `quote`) so the failure is self-contained |

Pong frame — answer to `ping`:

```json
{ "type": "pong" }
```

Error frame — protocol-level problem (oversize frame, malformed JSON, unknown `op`); the connection stays up:

```json
{ "type": "error", "code": "frame_too_large", "msg": "frame exceeds 64 KiB" }
```

## Rules

* **Connection cap, HA allowed.** Up to **3** connections per maker (default) — enough for active/standby setups; an extra connection is closed with code `4005`.
* **The server only sees the instant.** At any moment, a pair's book is whatever the last applied request carried — there is no per-connection state and no ordering between your connections. If your own instances interleave or race pushes, the resulting book is your responsibility.
* **Idle limit 60 s**, counted as no inbound frames of any kind — pushes count as liveness, so ping only when you are not pushing, at least every 30 s. Idle closes the connection with code `4003`. Both ping mechanisms count toward the inbound rate limit (negligible at one ping per 30 s).
* Max frame size 64 KiB.
* Inbound message rate ≤ **25 msg/s per connection** (all frames, ping included) — exceeding closes the connection with code `4004`.
* Acks arrive in the order you pushed, **within one connection**.

Demo (Node.js):

```javascript
const WebSocket = require("ws");

const ws = new WebSocket("wss://api.native.org/v1/ws", {
  headers: { "Apikey": "YOUR_API_KEY" },
});

ws.on("open", () => {
  ws.send(JSON.stringify({
    op: "push",
    nonce: 1731318400124,
    orderbooks: [{
      chain_id: 56,
      base:  { address: "0x698250914c54eb2f1bb7e0eb1e6a24c0f9b1c1a1", symbol: "MEME" },
      quote: { address: "0x55d398326f99059ff775485246999027b3197955", symbol: "USDT" },
      bids: [{ price: "0.98", qty: "12000" }],
      asks: [{ price: "1.02", qty: "8000" }],
      expire_at_ms: Date.now() + 5000,
      book_id: "b-000124",
    }],
  }));
});

ws.on("message", (raw) => console.log(raw.toString()));

// keepalive
setInterval(() => ws.send(JSON.stringify({ op: "ping" })), 25000);
```

## Close codes

| Code | Meaning |
| --- | --- |
| 4001 | Authentication failed |
| 4002 | Source IP not allowlisted |
| 4003 | Idle timeout |
| 4004 | Protocol violation (including exceeding the inbound message rate) |
| 4005 | Connection limit reached (per maker) |
| 1001 | Server restarting — reconnect and resume pushing; books recover from your next push |
