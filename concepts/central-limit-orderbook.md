# Central Limit Order Book

The **Central Limit Order Book (CLOB)** is the matching mechanism at the heart of [Native Core](../modules/native-core.md). Unlike an AMM, which prices trades along a bonding curve, a CLOB matches discrete buy and sell orders by **price-time priority**: the best-priced order is filled first, and ties are broken by who arrived first.

### How It Works on Native Core

* **Order types:** limit orders (fill at a specified price or better) and market orders (fill at the best available price).
* **On-chain order book:** the full order book state is stored on-chain. Every placement, cancellation, and fill is the result of an on-chain transaction.
* **Consensus-ordered:** transactions are sequenced by consensus, then nodes execute matching deterministically — making the order sequence canonical, auditable, and resistant to opaque front-running.
* **Partial fills & self-trade prevention:** large orders can fill across multiple price levels; orders that would trade against the same account are prevented.

### Order Lifecycle

```
Pending → Open → (PartiallyFilled) → Filled / Cancelled
```

### Why It Matters

A CLOB gives traders the familiar order types and price precision of a centralized exchange, while keeping execution fully transparent. Combined with a dedicated app chain, it delivers professional-grade execution without the MEV and slippage characteristic of AMMs.

### Related

* [Native Core](../modules/native-core.md)
* [Market-Responsive Pricing](market-responsive-pricing.md)
