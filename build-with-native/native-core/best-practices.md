---
description: The core habits a live Native Core integration should follow.
---

# Best practices

## Orders & cancels

* **Give every action a unique `cloid`.** It is your only handle to reconcile an order after a network blip.
* **Refresh a stale quote by cancel + re-place**, not by chasing it with `modify`.

## Know the outcome

* **`accepted` is not success — read `response`.** An order that failed at execution still returns `submission_status: "accepted"` with no top-level `error`; the code is in `response.status.error`. Branching on `submission_status` alone records a rejected order as live.
* **The fill state is already in the `/trade` reply.** `response.status` is `open` / `filled` / `cancelled` / `error` — you don't need an [`orderStatus`](post-info.md#orderstatus) poll to learn which.
* **Only `RateLimited` is safe to resend** — back off `error.retry_after_ms` and resend the same signed action. Every other rejection needs a fresh action.
* **On a `timeout`, branch on `error.code` before you decide.** The `Handoff*` family (HTTP 503) never reached a node — resubmit it, or you drop every write for the duration of a leadership handoff. Everything else may still land: reconcile by `cloid`, never resubmit under a new nonce. See [Handle outcomes & timeouts](handle-timeouts.md).

## Signing & nonces

* **One nonce source per API wallet** — a monotonic millisecond clock. Never run two writers on one key; they collide. Shard across [distinct API wallets](nonces-and-api-wallets.md).
* **Resolve `agent_epoch` live** from [`userAgents`](post-info.md#useragents); don't hardcode it.

## Numbers

* **Send price and quantity as strings, never floats** — a binary float silently corrupts a price.
* **Snap to the market's precision before signing.** Take `price_decimals` / `base_quantity_decimals` from [`markets`](post-info.md#markets) and the minimum from [`quoteAssets`](post-info.md#quoteassets).

## Reads & rate limits

* **Query `/info` by the owner address**, not the API-wallet address — the agent address returns nothing.
* **Budget 1 request/second per IP on each endpoint.** Reads and writes hold separate buckets, so polling never eats your order rate — but neither bucket gives you a second request. Cache static metadata, poll on a fixed interval, and back off on `429` — see [rate limits](api-access.md#rate-limits-errors).
* **Once you need more than one read per second, stream instead of polling.** A [WebSocket](websocket.md) subscription costs nothing against the request budget.

## Streaming

* **Quote off `bbo`, not `l2Book`.** On mainnet `l2Book` is a five-second snapshot — read it for depth and shape, never for the price you act on. `bbo` pushes on every change to the top of book.
* **`orderUpdates` is the event stream; `openOrders` is the reconciliation.** `openOrders` is a full replacement at most every five seconds, so driving state off it silently drops every transition in between. Track lifecycle on `orderUpdates` and use `openOrders` to catch drift.
* **A partial fill arrives as `status: "open"`.** Compare `sz` (remaining) against `origSz` on every update — branching on `status` alone misses partials entirely.
* **Never wait on the stream to confirm a submission.** `orderUpdates` carries only orders that reached the matching engine; a bad nonce or signature comes back on the `/trade` response and will never arrive as a frame.
* **Deduplicate fills on `tid`.** After a reconnect the `userFills` snapshot overlaps the live stream by design. Resubscribe and rebuild from the snapshot packets rather than trying to patch the gap.
* **On a credit account, watch `pending_exposure`, not just `actual`.** It moves the moment an order rests or cancels, so it is your live risk ahead of settlement — see [`spotCreditState`](websocket.md#spotcreditstate).
* **Don't share the `post` channel between market data and trading.** One `post` may be in flight per IP, and a synchronous `/trade` holds that slot for up to a block — a cheap `info` query queued behind it just gets a `429`.

## Degraded states

* **When suspended or frozen, keep cancelling.** During `PlaceOrderSuspended` or on a frozen account only `cancel` / `cancelAll` go through — use them to reduce exposure; don't bundle a new order into the batch.
