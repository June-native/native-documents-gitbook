# CLOB

The **Central Limit Order Book (CLOB)** is the matching mechanism at the heart of [Native Core](../modules/native-core.md). Unlike an AMM, which prices trades along a bonding curve, a CLOB matches discrete buy and sell orders by **price-time priority**.

### How It Works on Native Core

* **Order types:** limit orders (fill at a specified price or better) and market orders (fill at the best available price) are supported.
* **On-chain order book:** the full order book state is stored on-chain. Every placement, cancellation, and fill is the result of an on-chain transaction.
* **Auditable matching:** the order sequence is canonical, matchings are deterministic and auditable.

### Why It Matters

A CLOB gives traders the familiar order types and price precision of a centralized exchange, while keeping execution fully transparent. Combined with a dedicated app chain, it delivers professional-grade execution without the MEV and slippage characteristic of AMMs.
