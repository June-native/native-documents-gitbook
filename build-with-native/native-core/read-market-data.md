---
description: Read markets, prices, order books, and your own positions over POST /info — no signing required.
---

# Read market data

Every read is a `POST /info` call, unauthenticated, dispatched on a top-level `type`. This guide points you at the query for each task; the full response shapes are in the [POST /info reference](post-info.md). Query responses carry `query_height` and `app_hash` so you know which committed state you read.

Testnet: `API_URL=https://api-test.native.org`.

{% hint style="info" %}
**Streaming coming soon.** Reads are poll-based today; a WebSocket streaming API for live market data and order updates is on the roadmap.
{% endhint %}

## Discover markets

Before you can sign an order you need the `market_id` and the market's precision.

```bash
curl -sS -X POST "$API_URL/info" -H 'content-type: application/json' \
  -d '{"type":"markets"}'
```

* [`markets`](post-info.md#markets) — `market_id`, `price_decimals`, `base_quantity_decimals`, `max_price_sig_figs` for every pair.
* [`assets`](post-info.md#assets) — asset ids, symbols, `balance_decimals`.
* [`quoteAssets`](post-info.md#quoteassets) — the per-quote-asset **minimum notional** an order must clear.

## Prices and the book

* [`l2Book`](post-info.md#l2book) — aggregated bids/asks for one market (optional `depth`, default 20).
* [`markPrices`](post-info.md#markprices) — mark prices (all, or filtered by `asset_ids`).
* [`oracleStatus`](post-info.md#oraclestatus) — whether oracle marks are fresh.

## Your account

Pass your **owner** address as `user` (not the API-wallet address).

* [`userBalances`](post-info.md#userbalances) — `available` / `locked` per asset.
* [`openOrders`](post-info.md#openorders) — resting orders, with `filled_qty` / `remaining_qty`.
* [`userFills`](post-info.md#userfills) — fill history over a height range.
* [`orderStatus`](post-info.md#orderstatus) — the lifecycle of one order by `oid` or `cloid`. **This is where you see whether an order rested or filled** — `/trade` only tells you it landed.
* [`spotCreditPositions`](post-info.md#spotcreditpositions) / [`spotCreditAccount`](post-info.md#spotcreditaccount) — credit-account positions and credit line (see [Account Types](account-types.md)).

## Next steps

* [POST /info](post-info.md) — every query and its full response
* [Trade over REST](trade-over-rest.md) — place your first order
