---
description: The core habits a live Native Core integration should follow.
---

# Best practices

## Orders & cancels

* **Give every action a unique `cloid`.** It is your only handle to reconcile an order after a network blip.
* **Refresh a stale quote by cancel + re-place**, not by chasing it with `modify`.

## Know the outcome

* **`accepted` means landed, not filled.** Read [`orderStatus`](post-info.md#orderstatus) to see whether the order rested or filled.
* **Only `RateLimited` is safe to resend** — back off `error.retry_after_ms` and resend the same signed action. Every other rejection needs a fresh action.
* **Never resend a `timeout` under a new nonce** — reconcile by `cloid` instead. See [Handle outcomes & timeouts](handle-timeouts.md).

## Signing & nonces

* **One nonce source per API wallet** — a monotonic millisecond clock. Never run two writers on one key; they collide. Shard across [distinct API wallets](nonces-and-api-wallets.md).
* **Resolve `agent_epoch` live** from [`userAgents`](post-info.md#useragents); don't hardcode it.

## Numbers

* **Send price and quantity as strings, never floats** — a binary float silently corrupts a price.
* **Snap to the market's precision before signing.** Take `price_decimals` / `base_quantity_decimals` from [`markets`](post-info.md#markets) and the minimum from [`quoteAssets`](post-info.md#quoteassets).

## Reads & rate limits

* **Query `/info` by the owner address**, not the API-wallet address — the agent address returns nothing.
* **`/info` is 1 request/second per IP.** Cache static metadata, poll on a fixed interval, and back off on `429`.

## Degraded states

* **When suspended or frozen, keep cancelling.** During `PlaceOrderSuspended` or on a frozen account only `cancel` / `cancelAll` go through — use them to reduce exposure; don't bundle a new order into the batch.
