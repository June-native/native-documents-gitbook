# Order Types & Time in Force

When placing an order on the [Native Core](../modules/native-core.md) [CLOB](central-limit-orderbook.md), you choose an **order type** (how the order is priced) and a **time-in-force** (how long it stays active and how it is allowed to fill).

### Order Types

* **Limit** — fills at a specified price or better. If it cannot fully fill on entry, the remainder rests on the order book.
* **Market** — fills at the best available price. A market order does not rest on the book; any unfillable remainder is cancelled.

### Time in Force (TIF)

| TIF | Name | Behavior |
| --- | --- | --- |
| `gtc` | Good-Til-Cancelled | Rests on the book until filled or explicitly cancelled. |
| `ioc` | Immediate-Or-Cancel | Fills as much as possible immediately; cancels any remainder. |
| `fok` | Fill-Or-Kill | Must fill in full immediately, or the entire order is cancelled. |
| `alo` | Add-Liquidity-Only (post-only) | Only rests as a maker order; rejected if it would immediately match and take liquidity. |

### Valid Combinations

The protocol enforces which combinations are allowed:

* **Market** orders are valid only with `ioc` or `fok`.
* `gtc` and `alo` are valid only for **limit** orders.
* A `price` must be present for limit orders and for protected market orders.

### What Happens When an Order Cannot Fill

Outcome-stage rejections are observable through [order status](../build-with-native/native-core/post-info.md) within the recent query window:

* `IocCancel` — an `ioc` order had unfilled remainder that was cancelled.
* `FokCancel` — a `fok` order could not fill in full and was cancelled entirely.
* `MarketOrderNoLiquidity` — a market order found no liquidity to fill against.
* `BadAloPx` — an `alo` (post-only) order would have crossed the book and taken liquidity, so it was rejected.

### Related

* [Central Limit Order Book](central-limit-orderbook.md)
* [Markets, Assets & Decimals](markets-and-assets.md)
* [Native Core](../modules/native-core.md)
