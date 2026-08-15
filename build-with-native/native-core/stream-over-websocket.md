---
description: >-
  From an open socket to a live book and your own fills — the fastest path to
  streaming Native Core.
---

# Stream over WebSocket

Polling [`POST /info`](post-info.md) costs you a request per read and a round trip of latency. One WebSocket connection replaces it: subscribe once, and the server pushes books, trades, fills, and order updates as they happen.

Examples use mainnet, `wss://api.native.org/ws`. For testnet, swap in `wss://api-test.native.org/ws` — see [environments](api-access.md#environments).

## 1. Connect

No authentication, no headers.

```bash
wscat -c wss://api.native.org/ws
```

Use **one** connection and multiplex every subscription onto it — you get 10 subscriptions per connection, and the per-IP connection cap is 1. See [limits](websocket.md#limits) for which caps refuse you today and which are still only being measured.

## 2. Subscribe to a market

Market channels are keyed by `coin` — the **`market_id` as a string** (`"2"`, not `"ETH"`). Read the ids from [`markets`](post-info.md#markets) once at startup.

```json
{"method":"subscribe","subscription":{"type":"bbo","coin":"2"}}
{"method":"subscribe","subscription":{"type":"trades","coin":"2"}}
{"method":"subscribe","subscription":{"type":"l2Book","coin":"2"}}
```

Each is acked with a `subscriptionResponse`, then data starts:

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

Pick the channel that matches how fast you need to react: `bbo` and `trades` push on every change, while `l2Book` is a throttled full snapshot. The [cadence table](websocket.md#push-cadence) has the numbers.

## 3. Subscribe to your account

Account channels are keyed by `user` — your **owner** address, not the API-wallet address.

```json
{"method":"subscribe","subscription":{"type":"userFills","user":"0x…"}}
{"method":"subscribe","subscription":{"type":"orderUpdates","user":"0x…"}}
{"method":"subscribe","subscription":{"type":"openOrders","user":"0x…"}}
```

`userFills` opens with a snapshot of your 100 most recent fills marked `isSnapshot: true`, then streams each new one. `orderUpdates` reports every transition of your accepted orders — `open`, `filled`, `canceled`, and a family of `*Rejected` codes naming why an order died at execution (`badAloPxRejected` and friends; see [orderUpdates](websocket.md#orderupdates)) — batched one frame per block.

For balances, subscribe to `spotState`. If you trade on a credit line, subscribe to `spotCreditState` instead — a credit account's spot balances are empty. See [Account Types](account-types.md).

## 4. Keep it alive

The server closes any connection it hasn't sent a frame to for 60 seconds. Send a ping every 30 seconds and stop worrying about it:

```json
{ "method": "ping" }
```

It answers `{"channel":"pong"}`. Your client library's built-in protocol-level ping works just as well.

## 5. Reconnect and backfill

Assume the connection will drop. On reconnect, resubscribe and take the first snapshot packet as your new baseline — `l2Book`, `openOrders`, `spotState`, and `spotCreditState` always carry complete state, and `userFills` replays its recent history for you.

Only `trades` and `orderUpdates` can leave a hole, because they are pure increments. Fill it from `POST /info`: [`userFills`](post-info.md#userfills) takes a `from_height` / `to_height` range within the recent query window, and [`orderStatus`](post-info.md#orderstatus) settles the fate of any single order by `oid` or `cloid`.

{% hint style="info" %}
Deduplicate fills on `tid`. The snapshot, the live frame, and `POST /info` all report one value for a trade. Parse it as a `BigInt`, not a JS `Number`.
{% endhint %}

## Trade on the same socket

You do not need a second transport to write. `post` carries [`POST /info`](post-info.md) and [`POST /trade`](post-trade.md) bodies over the open connection:

```json
{
  "method": "post",
  "id": 1,
  "request": {
    "type": "action",
    "payload": {
      "action": {
        "type": "order",
        "market_id": "2",
        "side": "bid",
        "order_type": "limit",
        "tif": "gtc",
        "price": "1900.00",
        "quantity": "1.0000",
        "cloid": "0x11111111111111111111111111111111"
      },
      "nonce": "1760000000000",
      "agent_epoch": "3",
      "signature": "0x…"
    }
  }
}
```

The reply echoes your `id` and carries the same response `POST /trade` would have returned, `submission_status` and all. Signing is unchanged — see [Transaction Signing](transaction-signing.md). One request at a time, and the same per-IP rate budget as HTTP applies.

## Next steps

* [WebSocket](websocket.md) — every channel, payload, and limit
* [Best practices](best-practices.md#streaming) — the traps worth knowing before you go live
* [Handle outcomes & timeouts](handle-timeouts.md) — what to do with each `submission_status`
* [Read market data](read-market-data.md) — the poll-based equivalents
