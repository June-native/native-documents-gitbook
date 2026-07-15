---
description: Field names and terms used across the POST /trade and POST /info references.
---

# Notation

Terms and field names shared by the [`POST /trade`](post-trade.md) and [`POST /info`](post-info.md) references.

| Term              | Meaning                                                                                                                                           |
| ----------------- | ----------------------------------------------------------------------------------------------------------------------------------------------- |
| `side`            | Order direction: `bid` (buy) or `ask` (sell). `buy` / `sell` are accepted aliases.                                                               |
| `order_type`      | `limit` (priced, can rest) or `market` (protected — `price` is the worst acceptable fill, never rests).                                          |
| `tif`             | Time-in-force: `gtc`, `ioc`, `fok`, or `alo`. See [below](#time-in-force).                                                                       |
| `price`           | Human decimal **string**, quote units per 1 base. Limit price for a limit order; protection price for a market order.                            |
| `quantity`        | Human decimal **string**, base-asset size. Must be greater than zero.                                                                            |
| notional          | `price × quantity`, denominated in the market's quote asset.                                                                                     |
| minimum notional  | Per-quote-asset floor on order notional. An order below it is rejected with `MinTradeSpotNtl`. Read the floor from `POST /info` `quoteAssets` (`min_quantity`); a market order is checked against its protection price. |
| `cloid`           | Client order id — your `0x`-prefixed 16-byte handle. Optional on `order`; it is the reconcile / idempotency key. Same width for every action.    |
| `oid`             | Exchange order id — decimal integer string assigned by the matching engine when an order is placed. Provide `oid` **or** `cloid` to cancel/modify; if both are present, `oid` wins. |

### Time-in-force

| `tif` | Semantics                                                                                          |
| ----- | ------------------------------------------------------------------------------------------------- |
| `gtc` | Good-til-cancelled. Rests on the book until filled or cancelled. Limit only.                       |
| `ioc` | Immediate-or-cancel. Fills whatever crosses now; cancels any remainder. Does not rest.             |
| `fok` | Fill-or-kill. Fills the full quantity immediately or cancels the whole order — no partial fill.    |
| `alo` | Add-liquidity-only (post-only). Must rest as maker; rejected if it would cross the book on entry. Limit only. |

`market` orders take `ioc` or `fok`; `gtc` and `alo` are limit-only.

### Identifiers

| Term        | Meaning                                                                                                                          |
| ----------- | ------------------------------------------------------------------------------------------------------------------------------ |
| `market_id` | Protocol-assigned market id (`u32`). Discover via `POST /info` `markets`. Sent as a decimal string (or unsigned integer) on `/trade`, returned as an integer on `/info`. |
| `asset_id`  | Protocol-assigned asset id (`u32`). Discover via `POST /info` `assets`.                                                          |

Ids are assigned by the protocol, not chosen by the client. In the Python SDK, `Info.resolve_market_id("BASE/QUOTE")` maps a symbol (e.g. `"ETH/USDC"`) to its `market_id`, and a market argument accepts either the symbol or the id.

### Signing scope

| Term        | Meaning                                                                                                                          |
| ----------- | ------------------------------------------------------------------------------------------------------------------------------ |
| `nonce`     | Decimal `u64` Unix-millisecond timestamp, authority-scoped. See [nonces-and-api-wallets.md](nonces-and-api-wallets.md).          |
| `authority` | The address recovered from a `/trade` signature. Nonce validation and rate limits are keyed on it — one API wallet is one authority. |

{% hint style="info" %}
`price`, `quantity`, and notional are human display decimals sent as strings; Native Core executes on integer atoms. For the raw-atom / display conversion model, see:

{% content-ref url="decimals-units.md" %}
[decimals-units.md](decimals-units.md)
{% endcontent-ref %}
{% endhint %}
