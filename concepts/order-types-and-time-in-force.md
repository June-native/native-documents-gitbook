# Order Types

When placing an order on the Native Core [CLOB](central-limit-orderbook.md), you choose an **order type** (how the order is priced) and a **time-in-force** (how long it stays active and how it is allowed to fill).

#### Order Types

* **Limit** — fills at a specified price or better. If it cannot fully fill on entry, the remainder rests on the order book.
* **Market** — fills at the best available price. A market order does not rest on the book; any unfillable remainder is cancelled.

#### Time in Force

<table><thead><tr><th width="111.859375">TIF</th><th width="263.8046875">Name</th><th>Behavior</th></tr></thead><tbody><tr><td><code>gtc</code></td><td>Good-Til-Cancelled</td><td>Rests on the book until filled or explicitly cancelled.</td></tr><tr><td><code>ioc</code></td><td>Immediate-Or-Cancel</td><td>Fills as much as possible immediately; cancels any remainder.</td></tr><tr><td><code>fok</code></td><td>Fill-Or-Kill</td><td>Must fill in full immediately, or the entire order is cancelled.</td></tr><tr><td><code>alo</code></td><td>Add-Liquidity-Only (post-only)</td><td>Only rests as a maker order; rejected if it would immediately match and take liquidity.</td></tr></tbody></table>
