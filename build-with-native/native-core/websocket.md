---
description: >-
  Stream live books, trades, fills, and order updates over one socket — and send
  /info and /trade requests down the same connection.
---

# WebSocket

One connection carries two kinds of traffic:

* **Subscriptions** — nine push channels covering markets and accounts. You subscribe; the server pushes.
* **`post` requests** — the same `POST /info` and `POST /trade` bodies, sent over the socket and answered on it.

The wire format follows the Hyperliquid WebSocket conventions — same envelopes, same channel names, same field names — so an existing Hyperliquid client maps onto the shapes below. The one thing to change is the market identifier: Native Core markets are addressed by id, not symbol.

## Endpoint

| Environment | URL |
| --- | --- |
| **Mainnet** | `wss://api.native.org/ws` |
| Testnet | `wss://api-test.native.org/ws` |

The socket is unauthenticated. Account channels are keyed by address and carry only data that is already public on chain; signing matters only for a `post` action, which carries the same signed body `POST /trade` takes.

```bash
wscat -c wss://api.native.org/ws
> {"method":"subscribe","subscription":{"type":"bbo","coin":"2"}}
```

## Requests

Four methods, all sent as one JSON text frame:

```json
{"method":"subscribe","subscription":{"type":"l2Book","coin":"2"}}
{"method":"unsubscribe","subscription":{"type":"l2Book","coin":"2"}}
{"method":"ping"}
{"method":"post","id":1,"request":{"type":"info","payload":{"type":"markets"}}}
```

Inside `subscription`, `type` selects the channel and one of two keys selects the topic:

* `coin` — market channels. The **`market_id` as a string** (`"2"`) — not a ticker, not a number. Ids come from [`markets`](post-info.md#markets).
* `user` — account channels. Your **owner** address, `0x`-prefixed and case-insensitive — not the API-wallet address.

One subscription covers one market or one address. To follow several markets, send several `subscribe` frames on the same connection; there is no all-markets form.

## Responses

Every push is `{"channel":"<name>","data":<payload>}`.

| Frame | Payload |
| --- | --- |
| Subscribe / unsubscribe ack | `{"channel":"subscriptionResponse","data":{"method":"subscribe","subscription":{"type":"trades","coin":"2"}}}` |
| Heartbeat reply | `{"channel":"pong"}` |
| Error | `{"channel":"error","data":"<reason>"}` |

Subscription arguments are validated when you subscribe, so a bad request fails loudly instead of ACKing and never pushing. Anything that cannot be turned into a valid subscription — malformed JSON, an unknown `method`, an unsupported `type`, a malformed address, a `market_id` that does not exist — returns one error shape that echoes your request back:

```json
{
  "channel": "error",
  "data": "Error parsing JSON into valid websocket request: {\"method\":\"subscribe\",\"subscription\":{\"type\":\"l2Book\",\"coin\":\"99999\"}}"
}
```

Two states have their own text:

```json
{"channel":"error","data":"Already subscribed: {\"type\":\"l2Book\",\"coin\":\"2\"}"}
{"channel":"error","data":"Already unsubscribed: {\"type\":\"trades\",\"coin\":\"0\"}"}
```

Treat the `subscriptionResponse` as the signal that a topic is live, and log `error` frames — a silent stream is almost always a subscription that was refused.

## Channels

Nine channels. All prices, sizes, and amounts are **display decimals**, already scaled by the market's `price_decimals` and `base_quantity_decimals` — the same values `POST /info` returns for the same state.

### Market channels — keyed by `coin`

#### `trades`

One frame per matched trade. `data` is an array.

```json
{
  "channel": "trades",
  "data": [
    {
      "coin": "2",
      "side": "B",
      "px": "1908.54",
      "sz": "0.0478",
      "time": 1785332190946,
      "tid": 277789649957552128,
      "users": [
        "0x0000000000000000000000000000000000000001",
        "0x0000000000000000000000000000000000000002"
      ]
    }
  ]
}
```

| Field | Meaning |
| --- | --- |
| `side` | Aggressor direction — `B` the taker bought, `A` the taker sold |
| `time` | Block timestamp, milliseconds |
| `tid` | Unique id for this trade. [`userFills`](#userfills) and [`POST /info userFills`](post-info.md#userfills) report the same value, so the feeds join on it. Parse it as a `BigInt`, not a JS `Number`. |
| `users` | `[buyer, seller]` |

#### `l2Book`

Full order-book snapshot — every frame is complete, there are no incremental diffs.

```json
{
  "channel": "l2Book",
  "data": {
    "coin": "2",
    "levels": [
      [
        { "px": "1903.49", "sz": "24.8254", "n": 1 },
        { "px": "1903.23", "sz": "234.0143", "n": 1 }
      ],
      [
        { "px": "1904.56", "sz": "50.6472", "n": 1 },
        { "px": "1904.84", "sz": "227.6955", "n": 1 }
      ]
    ],
    "time": 1785332059496
  }
}
```

`levels` is `[bids, asks]`, each best-first and capped at 10 levels per side. `n` is the number of resting orders at that price. The book is throttled — see [push cadence](#push-cadence) — so use `bbo` when you need the top of book at event speed.

#### `bbo`

Top of book only, pushed whenever it changes.

```json
{
  "channel": "bbo",
  "data": {
    "coin": "2",
    "time": 1785332059496,
    "bbo": [
      { "px": "1903.49", "sz": "24.8254", "n": 1 },
      { "px": "1904.56", "sz": "57.71", "n": 1 }
    ]
  }
}
```

`bbo` is `[best_bid, best_ask]`; a side with no liquidity is `null`.

#### `allMids`

Mid price for every market with liquidity on both sides, as one table. Keys are market ids; each value is `(best_bid + best_ask) / 2`.

```json
{
  "channel": "allMids",
  "data": {
    "mids": {
      "0": "1902.25",
      "2": "1904.025",
      "3": "569.195",
      "14": "0.9999",
      "45": "677.104"
    }
  }
}
```

`allMids` takes no `coin` — subscribe with `{"type":"allMids"}` alone. A market with only one side quoted is left out, and a mid carries one more decimal place than the market's `price_decimals`, since it is a midpoint.

### Account channels — keyed by `user`

#### `userFills`

Your fills, both sides of the book. The first packet after subscribing is a snapshot of your 100 most recent fills, marked `isSnapshot: true`; every fill after that streams as it happens, with the field absent.

```json
{
  "channel": "userFills",
  "data": {
    "user": "0x0000000000000000000000000000000000000001",
    "fills": [
      {
        "coin": "45",
        "px": "677.104",
        "sz": "0.554",
        "side": "B",
        "time": 1785332187696,
        "oid": 2170230549774592,
        "crossed": true,
        "fee": "0.00000554",
        "feeToken": "QQQB",
        "tid": 277789510371147776,
        "cloid": "0x00000000006179420000002d00000001"
      }
    ]
  }
}
```

| Field | Meaning |
| --- | --- |
| `side` | **Your** direction — `B` you bought, `A` you sold |
| `crossed` | `true` when you were the taker |
| `fee` / `feeToken` | The fee charged on **your** side of this fill, and the asset it was charged in. A zero-rate fill reports `fee: "0"` **with** `feeToken` present — this is the common case on zero-fee markets. `feeToken` is omitted only when the fill carries no fee record at all. |
| `tid` | Same id as [`trades`](#trades) and [`POST /info userFills`](post-info.md#userfills). Both sides of a trade share it, so a multi-account connection dedups on `(user, tid)`. |

A snapshot fill is built exactly like the live frame for the same trade — same `tid` — so you can deduplicate cleanly across the snapshot-to-stream boundary after a reconnect. The snapshot lists fills oldest-first.

#### `orderUpdates`

Lifecycle transitions of your accepted orders. `data` is an array of the transitions your account made in one block — batched, but **split across several frames once the array would exceed 56 KiB**. Don't treat one frame as the complete set for a block.

```json
{
  "channel": "orderUpdates",
  "data": [
    {
      "order": {
        "coin": "49",
        "side": "B",
        "limitPx": "490.681",
        "sz": "157.634",
        "oid": 2366974344560896,
        "timestamp": 1785918530096,
        "origSz": "157.634",
        "cloid": "0x04becbf0aa5ffcaaba6be4126f0debff"
      },
      "status": "open",
      "statusTimestamp": 1785918530096
    },
    {
      "order": {
        "coin": "49",
        "side": "A",
        "limitPx": "496.537",
        "sz": "0",
        "oid": 2366974277452805,
        "timestamp": 1785918529896,
        "origSz": "164.016",
        "cloid": "0x04abf5ad3ce48f8ba8cdb7b05fabf38f"
      },
      "status": "canceled",
      "statusTimestamp": 1785918530096
    },
    {
      "order": {
        "coin": "45",
        "side": "A",
        "limitPx": "720.305",
        "sz": "0",
        "oid": 2368550228460800,
        "timestamp": 1785923226596,
        "origSz": "0.463",
        "cloid": "0x00000000008e17d10000002d00000001"
      },
      "status": "filled",
      "statusTimestamp": 1785923226596
    }
  ]
}
```

* `order.sz` is the **remaining** quantity and `origSz` the original, so a partial fill shows as `status: "open"` with a shrunken `sz`.
* **A terminal transition carries no filled quantity.** The canceled order above reports `sz: "0"` against an `origSz` of `164.016` — that zero is the order leaving the book, not a claim that all of it traded. Read the traded amount from [`orderStatus`](post-info.md#orderstatus), which returns `filled_qty`.
* `status` names both the transition and, when an order died at execution, **why**:

| `status` | meaning |
| --- | --- |
| `open` | resting — including partially filled, which shows as a shrunken `sz` |
| `filled` | fully filled |
| `canceled` | taken off the book: your `cancel`, a `modify`, or the unfilled remainder of a partial fill |
| `selfTradeCanceled` | cancelled by self-trade prevention — see the note below, this one can carry fills |
| `badAloPxRejected` | post-only (ALO) order would have crossed the book |
| `iocCancelRejected` | IOC order found no fill |
| `fokCancelRejected` | FOK order could not be filled in full |
| `marketOrderNoLiquidityRejected` | market order found no liquidity |

  There is no bare `rejected` — every rejection names its reason.

  Values meaning "the order died without resting" end in `Rejected`, so **match
  the suffix, not the exact word** — `status.endsWith("Rejected")` stays correct
  as reasons are added. Post-only crossings are by far the most common
  rejection, and arrive as `badAloPxRejected`.

  Treat any status you do not recognise as **terminal-until-proven-otherwise**:
  read the order back with [`orderStatus`](post-info.md#orderstatus) and use its
  `filled_qty` and `is_resting` rather than assuming the order is still working.

* **`selfTradeCanceled` is a cancel, not a rejection, and it may have traded.**
  A crossing order can sweep several price levels and only then meet your own
  resting order, at which point it stops. The fills before that point are real.
  Read `filled_qty` from [`orderStatus`](post-info.md#orderstatus) before
  replacing the order — treating it like a rejection and re-sending the full
  size double-fills the traded part.

* **Two kinds of failure never appear here — and their absence is not a success
  signal.**
  * **Envelope failures** — bad nonce, expired transaction, bad signature. The
    transaction never executed.
  * **Order-validation failures** — tick size, minimum notional, insufficient
    balance. The transaction *did* execute, but the order was refused before it
    was assigned an order id, so no order status exists to push.

  Both are reported synchronously on the [`POST /trade`](post-trade.md)
  response, and that response is the only place they appear. Do **not** infer
  "the order is working" from silence on this channel, and note that
  [`orderStatus`](post-info.md#orderstatus) cannot disambiguate either: a
  refused order and an order that has not been published yet both read back
  `found: false`.

#### `openOrders`

Every resting order on your account, as a full replacement — across all markets, not only the ones you subscribed to. The first packet carries `isSnapshot: true`; later refreshes omit it.

```json
{
  "channel": "openOrders",
  "data": {
    "isSnapshot": true,
    "user": "0x0000000000000000000000000000000000000001",
    "orders": [
      {
        "coin": "0",
        "side": "B",
        "limitPx": "1900.11",
        "sz": "2",
        "oid": 1508735205769472,
        "timestamp": 1785331894628,
        "origSz": "2",
        "cloid": "0xe839158bc4e9b91fe0311d83cce1b117"
      }
    ]
  }
}
```

#### `spotState`

Your spot balances, as a full replacement.

```json
{
  "channel": "spotState",
  "data": {
    "user": "0x0000000000000000000000000000000000000001",
    "balances": [{ "coin": "USDT", "token": 2, "total": "15000.5", "hold": "400" }]
  }
}
```

`total` is available plus locked, `hold` is the locked part, and `token` is the `asset_id`.

{% hint style="info" %}
A credit account's balance list is empty — it trades on its credit line, not on spot balance. Its positions are on `spotCreditState` below. See [Account Types](account-types.md).
{% endhint %}

#### `spotCreditState`

A credit account's per-asset positions and credit line, as a full replacement. The first packet carries `isSnapshot: true`.

```json
{
  "channel": "spotCreditState",
  "data": {
    "isSnapshot": true,
    "user": "0x0000000000000000000000000000000000000001",
    "authorized": true,
    "status": "active",
    "creditUsdAtoms": 10000000000000000,
    "positions": [
      {
        "asset_id": 1,
        "symbol": "USDC",
        "actual_display": "7372385.97518792",
        "actual_qty": "737238597518792",
        "pending_exposure_display": "0",
        "pending_exposure_qty": "0"
      },
      {
        "asset_id": 2,
        "symbol": "USDT",
        "actual_display": "-10592539.71238006",
        "actual_qty": "-1059253971238006",
        "pending_exposure_display": "0",
        "pending_exposure_qty": "0"
      }
    ]
  }
}
```

| Field | Meaning |
| --- | --- |
| `authorized` | Whether the address is a credit account at all. Anyone may subscribe; a non-credit address gets `authorized: false`, an empty `positions`, and no `status` or `creditUsdAtoms`. |
| `status` | `active` or `frozen` |
| `creditUsdAtoms` | The credit line in `usd_atoms` — the same value [`spotCreditAccount`](post-info.md#spotcreditaccount) returns |
| `actual_qty` / `actual_display` | The **settled** signed position, raw atoms and display-scaled. Negative is short. |
| `pending_exposure_qty` / `pending_exposure_display` | **Unsettled** signed exposure from your resting orders. It moves when an order rests or cancels, before any fill settles into `actual`. |

The `positions` objects are the same shape [`spotCreditPositions`](post-info.md#spotcreditpositions) returns, so the poll and the stream read identically.

## Push cadence

Channels come in two kinds, and the kind decides when a frame goes out.

**Event-driven** channels emit the moment the event happens, on the block it happens in:

| Channel | Emits on |
| --- | --- |
| `trades` | each matched trade |
| `bbo` | each change to the top of book |
| `userFills` | each of your fills |
| `orderUpdates` | each of your order transitions |

**Snapshot** channels carry the full current state and are throttled — the values below are the current floors, the same on mainnet and testnet, and not guarantees:

| Channel | No faster than |
| --- | --- |
| `l2Book` | 500 ms |
| `allMids` | 5 s |
| `openOrders` | 500 ms |
| `spotState` | 200 ms |
| `spotCreditState` | 200 ms |

A throttled channel pushes when its market or account was **touched** in a block, not when the payload actually differs — so expect repeat frames with identical content, and diff before acting if that matters to you. A skipped beat costs you nothing: the next frame is a complete snapshot.

{% hint style="warning" %}
`l2Book` is a throttled snapshot — one frame every 500 ms at best — so it is a picture of depth, not a live price. React on `bbo`, which pushes on every change to the top of book.
{% endhint %}

## Requests over the socket

`post` carries the REST bodies down the same connection, so a bot that streams can also trade without opening a second transport.

```jsonc
// request
{
  "method": "post",
  "id": 42,
  "request": {
    "type": "info",
    "payload": { "type": "l2Book", "market_id": 2, "depth": 2 }
  }
}

// success
{
  "channel": "post",
  "data": {
    "id": 42,
    "response": { "type": "info", "payload": { … } }
  }
}

// failure
{
  "channel": "post",
  "data": {
    "id": 42,
    "response": { "type": "error", "payload": "<status and description>" }
  }
}
```

* `request.type` is `"info"` — the payload is a [`POST /info`](post-info.md) body — or `"action"`, where the payload is a signed [`POST /trade`](post-trade.md) body.
* `id` is echoed on every reply. Use a distinct one per request to match replies to requests.
* A success reply carries exactly what the HTTP endpoint would have returned. A failure reply flattens the HTTP error into a plain string — the status followed by the same `error.code` or message — so decode it against [Error responses](error-responses.md).

An `action` reply is the full trade response, so a business rejection arrives as a **successful** `action` whose `submission_status` says what happened — the same contract as over HTTP:

```json
{
  "channel": "post",
  "data": {
    "id": 3,
    "response": {
      "type": "action",
      "payload": {
        "submission_status": "accepted",
        "tx_hash": "0xe84f3cfbedf6f069136aef795deaab5b74aa26cd1eb14d0f1020839134a82dc8",
        "response": {
          "type": "order",
          "status": {
            "open": {
              "oid": 1964626153570560,
              "cloid": "0x4e5e6ffddbed6ed66c3d02cab8a4cac6"
            }
          }
        }
      }
    }
  }
}
```

Only a transport-level failure uses the `error` envelope. Branch on `submission_status`, never on the envelope — see [Handle outcomes & timeouts](handle-timeouts.md).

Three things to plan for:

* **One request at a time.** A second `post` sent before the first is answered is refused with `"429 Too many in-flight post requests"`. Wait for each reply.
* **`post` costs the same as HTTP.** Requests charge the same [per-IP budgets](api-access.md#rate-limits-errors) the REST endpoints apply — 1 request/second for `info` and 1 request/second for `action`, on independent buckets — and come back as `"429 Rate limited, retry after <n>ms"` when you exceed one. The socket is a convenience, not extra quota.
* **`post` bodies are capped at 64 KiB** by the WebSocket message limit, where `POST /trade` accepts 256 KiB. Send a large `batch` action over HTTP.

`explorer` requests are not served:

```json
{
  "channel": "post",
  "data": {
    "id": 5,
    "response": {
      "type": "error",
      "payload": "400 explorer requests are not supported"
    }
  }
}
```

## Connection lifecycle

The server closes any connection it has not **sent** a frame to for 60 seconds. Every outbound frame resets that clock — a feed update, a `subscriptionResponse`, a `pong`, a `post` reply.

**Send `{"method":"ping"}` every 30 seconds and don't think about it again.** The server never probes you, so the heartbeat is yours to drive; the reply is `{"channel":"pong"}`. A WebSocket protocol-level ping works too — the pong it triggers refreshes the same clock — so a client library with built-in keepalive is already covered.

Reconnect on disconnect, always. The server may drop a connection without warning, and a clean close arrives as a plain WebSocket close with no reason attached. Three causes are worth knowing:

* **Idle** — nothing was pushed to you for 60 seconds. Ping.
* **Slow consumer** — the event channels (`trades`, `userFills`, `orderUpdates`) queue per connection, and a client that cannot drain that queue is dropped rather than buffered indefinitely. Snapshot channels never cause this — see [delivery](#recovering-missed-data). If you are being dropped under load, read faster or subscribe to less.
* **Node unavailable** — a server that is not serving refuses the upgrade with HTTP `503` and closes any connection it still holds. Back off and retry.

## Limits

| Limit | Value | Enforced today |
| --- | --- | --- |
| Connections per IP | 1 | not yet — counted only |
| New connections per IP | 30 / minute | not yet — counted only |
| Subscriptions per connection | 10 | yes |
| Inbound messages per connection | 2000 / minute | not yet — counted only |
| In-flight `post` requests per IP | 1 | yes |
| `post` request rate per IP | 1/second `info`, 1/second `action` | yes |
| Message size, either direction | 64 KiB | yes |

The connection caps are being measured against real traffic before they start refusing. **Design to them anyway** — one connection carrying every subscription — because they will be enforced, and a client built on many sockets breaks when they are.

Over the subscription cap the server replies `{"channel":"error","data":"Too many subscriptions"}` and the earlier subscriptions keep running. Ten subscriptions is three fully-watched markets — `l2Book` + `bbo` + `trades` each — plus one account channel, so multiplex everything onto the single connection rather than opening one per topic.

The two `post` budgets are independent of each other and shared with the REST endpoints: a read never spends write quota, and a request costs the same whether it arrives over HTTP or the socket. Subscriptions cost nothing against them — they are inbound messages, and only the message cap applies.

## Recovering missed data

The two kinds of channel recover differently.

* **Snapshot channels self-heal.** `l2Book`, `bbo`, `allMids`, `openOrders`, `spotState`, and `spotCreditState` always carry complete state, so a dropped frame is corrected by the next one and a reconnect needs no backfill. They are also **conflated per topic**: only the newest frame for a given book or account is held for you, so falling behind costs you resolution, never correctness — and never the connection.
* **`userFills` backfills itself.** Its first packet after subscribing is your 100 most recent fills, so a short disconnect costs you nothing. For a longer gap, [`userFills`](post-info.md#userfills) over `POST /info` accepts `from_height` / `to_height` within the recent query window — 10000 blocks, roughly 8 minutes.
* **`trades` and `orderUpdates` can gap.** They are pure increments and are not replayed. Reconstruct from `POST /info` — [`userFills`](post-info.md#userfills) for your own activity, [`orderStatus`](post-info.md#orderstatus) for one order.

There is no resume cursor: subscriptions take no height or sequence argument. Reconnect, resubscribe, and treat the first snapshot packet as your new baseline. Frames for a given market or address always arrive in chain order.

`candle` is not served on this endpoint — a `candle` subscription is refused like any other unsupported type.

## Next steps

* [Stream over WebSocket](stream-over-websocket.md) — the five-step walkthrough
* [Best practices](best-practices.md#streaming) — the traps worth knowing before you go live
* [Error responses](error-responses.md) — every `error.code` a `post` can return
* [POST /info](post-info.md) — every poll-based read, and the backfill queries
* [POST /trade](post-trade.md) — every action you can send as a `post`
