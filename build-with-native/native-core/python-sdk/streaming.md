---
description: Stream live market and account data with the SDK's WsClient, on one connection.
---

# Streaming

`WsClient` is the SDK's third class, alongside `Info` and `Exchange`. It carries nine push feeds over a single [WebSocket](../websocket.md) connection and delivers frames on a background thread, so nothing here asks you to write `async`.

Reach for it when polling will not do. `/info` reads are capped at **one per second per client IP**, which cannot drive a quoting loop; the socket pushes instead of costing a read.

## 1. Connect

```python
from native_core import Info

info = Info("https://api.native.org")
ws = info.ws(on_reconnect=lambda: print("resubscribed"))   # reuses the loaded market table
ws.connect()
```

`Info.ws()` and `Exchange.ws()` hand the client the market table they already loaded, so `"ETH/USDT"` resolves without another request. `WsClient(base_url)` works standalone; a client that only ever subscribes by market id makes no HTTP request at all.

For a node you run yourself, pass the WebSocket URL directly (`WsClient("ws://localhost:8081")`) — the derivation from an HTTP base URL holds only for the public endpoints.

## 2. Subscribe

Nine feeds, four by market and five by address:

| By market | By address |
| --- | --- |
| `subscribe_trades(market)` | `subscribe_user_fills(address)` |
| `subscribe_l2_book(market)` | `subscribe_order_updates(address)` |
| `subscribe_bbo(market)` | `subscribe_open_orders(address)` |
| `subscribe_all_mids()` | `subscribe_spot_state(address)` |
| | `subscribe_spot_credit_state(address)` |

Each returns a `Subscription` you can pass to `unsubscribe(...)`. Both calls block until the server answers and raise `SubscriptionError` if it refuses — a silent stream is almost always a refused subscription, so the SDK makes that loud rather than letting it hide.

Ten subscriptions per connection, and one connection per client IP. Multiplex everything onto the one socket; see [rate limits](../websocket.md#limits) for the full table.

## 3. Consume

Two ways, and you can mix them on one client.

**Callbacks** run on the reading thread, so they must return quickly. Use them for cheap work like tracking top of book.

**`stream()`** yields frames from every subscription created *without* a callback. It buffers for you, so it is the right choice when handling a message takes real work.

```python
def on_bbo(data: dict) -> None:
    bid, ask = data["bbo"]                    # either side is null when nothing rests
    print(bid["px"] if bid else "-", ask["px"] if ask else "-")

ws.subscribe_bbo("ETH/USDT", on_bbo)          # callback
ws.subscribe_trades("ETH/USDT")               # no callback -> goes to stream()

for message in ws.stream(timeout=30):         # idle timeout, not a run length
    if message["channel"] == "trades":
        for trade in message["data"]:
            print(trade["side"], trade["sz"], trade["px"])   # "B" = the taker bought
```

Each `stream()` item is the full `{"channel", "data"}` envelope, because the channel is what tells the feeds apart. The buffer holds `queue_maxsize` frames (10,000 by default); a consumer slower than the feed overflows it and the oldest frames are dropped rather than stalling the reader. `ws.dropped_messages` counts them — check it, do not assume it is zero.

React on `bbo`, not on `l2Book`: the book is a throttled snapshot at 500 ms, while `bbo` pushes on every change to the top of book.

## 4. What survives a reconnect, and what does not

`reconnect` is on by default: the client reopens the socket and restores every subscription, then fires `on_reconnect`. That is where to re-read anything that gapped.

| Feed | Behaviour |
| --- | --- |
| `l2Book`, `bbo`, `allMids`, `openOrders`, `spotState`, `spotCreditState` | Complete state every frame, so the next frame repairs any gap. Nothing to backfill |
| `userFills` | The first frame after subscribing replays your most recent 100 fills, so an ordinary resubscribe backfills itself |
| `trades`, `orderUpdates` | No replay. A gap is a gap |

{% hint style="warning" %}
**Frames can be lost without a disconnect.** The three event feeds (`trades`, `userFills`, `orderUpdates`) must stay ordered, so a slow consumer is disconnected from those. The six snapshot feeds are conflated instead — you silently skip intermediate frames and stay connected, which costs resolution rather than correctness.

Separately, when the server's broadcast falls behind it drops that gap's **event** frames outright and re-pushes only the snapshot feeds. There is no disconnect and no `on_reconnect`, and `userFills` does not replay the way it does on a resubscribe. Anything that needs complete fills must poll `Info.recent_fills` and reconcile by `tid`.
{% endhint %}

Deduplicate fills by `(user, tid)`, not by `tid` alone: both sides of a trade receive the same `tid`. And serialize `tid` as a string before handing it to a browser — the value exceeds JavaScript's safe integer range, which collapses two fills into one silently.

## 5. Trade on the same socket

Optional, and `Exchange.order` over HTTP is still the recommended write path. When you want orders on the connection you already hold:

```python
built = exchange.build_order("ETH/USDT", True, "0.01", "1900", "gtc")   # validates, does not sign
resp = ws.post_action(exchange.sign_action(built["action"]))            # signs, then sends
```

`build_order` runs every local check; `sign_action` signs without sending and **consumes a nonce**, so the body is single-use. What comes back is an ordinary `/trade` response, read exactly as over HTTP: `accepted` is a verdict on the transaction, so check `is_order_failed` or take `next_action`. See [Accepted is not placed](core-concepts.md#accepted-is-not-placed).

{% hint style="warning" %}
**A write over the socket reports less than the same write over HTTP.** When the API answers with anything other than 2xx it discards the trade response, so `submission_status`, `tx_hash` and the `response` envelope are simply not there. `post_action` raises instead: `ClientError` for a refusal that cannot have executed (bad signature or nonce, or a 429), `SubmissionUncertain` for anything that might still land. Both uncertain cases carry the order handles — reconcile with `reconcile_by_cloid`, never resubmit. A write here is **never** retried automatically, not even on a rate limit.
{% endhint %}

`ws.post_info(payload)` runs an `/info` read over the same connection. It shares the one per-IP read budget with HTTP, so it is convenience, not extra quota.

## Field names differ from HTTP

The socket speaks the protocol's own vocabulary and the SDK passes it through rather than inventing a third one. One pair is a trap:

| Over HTTP | Over the socket |
| --- | --- |
| `available` / `locked` (`Info.user_balances`) | `total` / `hold` (`spotState`), where **`total` is available plus locked** |
| `price` / `quantity` / `order_count` | `px` / `sz` / `n` |

Reading `total` where you used to read `available` overstates your free balance whenever an order is resting. The full mapping is in the [WebSocket reference](../websocket.md).

## Runnable example

`examples/ws_feeds.py` ships in the source distribution and needs no key:

```bash
python examples/ws_feeds.py                      # testnet
python examples/ws_feeds.py https://api.native.org
```

## Next

{% content-ref url="api-reference.md" %}
[api-reference.md](api-reference.md)
{% endcontent-ref %}

{% content-ref url="../websocket.md" %}
[websocket.md](../websocket.md)
{% endcontent-ref %}
