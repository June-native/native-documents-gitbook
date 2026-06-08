# Markets, Assets & Decimals

Trading on [Native Core](../modules/native-core.md) is organized around **assets** and **markets**. Understanding how they are identified and how their numbers are scaled is essential for reading the order book and placing orders correctly.

### Assets

An asset is a token tracked on Native Core, identified by a numeric `asset_id` and a unique `symbol`.

* **`balance_decimals`** — the internal scale used for balances and settlement of that asset.
* **`issuer`** — the owner account bound to an asset (via the operator listing action). It is a record-keeping binding only and confers no admin privileges; genesis and legacy-listed assets show the zero address.

### Markets

A market is a tradable pair, identified by a numeric `market_id`, with a base asset and a quote asset. Each market carries its own scaling metadata:

* **`base_quantity_decimals`** — lot scale for order quantity in the base asset.
* **`price_decimals`** — decimal places for price (quote units per 1 base unit).
* **`max_price_sig_figs`** — maximum significant figures allowed for non-integer display prices.
* **`quote_balance_decimals`** — the quote asset's `balance_decimals`, included so clients can convert quote notional without a separate lookup.

### Quote Assets & Minimum Notional

Only allowlisted **quote assets** may be used as the quote side of a market. Each carries a **minimum order notional** (`min_quantity`) so dust orders are rejected. For example, the genesis allowlist uses USDC and USDT with a `10` minimum, ETH with `0.01`, and BNB with `0.02`. Orders below the configured minimum are rejected with `MinTradeSpotNtl`.

### Raw Atoms vs Display Values

Native Core executes on **integers only**. Public APIs accept and return human-readable decimal strings, while the signed transaction payload carries raw integer **atoms**:

```
raw_quantity = display_quantity * 10^base_quantity_decimals
raw_price    = display_price * 10^price_decimals
```

Submitting more fractional digits than a market's `price_decimals` or `base_quantity_decimals` allows is rejected (`invalid_price_precision` / `invalid_quantity_precision`). For the full conversion rules and worked examples, see the Decimal Units reference.

### Related

* [Central Limit Order Book](central-limit-orderbook.md)
* [Order Types & Time in Force](order-types-and-time-in-force.md)
* [Decimals Units](../build-with-native/native-core/decimals-units.md)
