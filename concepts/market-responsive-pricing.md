---
hidden: true
---

# Market-Responsive Pricing

Native pricing is **Market-Responsive**: market makers price each token accurately and promptly based on market risk, conditions, and inventory exposure. Because of this, quotes reflect liquidity sourced from on-chain DEXs, CEXs, and/or direct redemption channels — ultimately responding to the real market rather than a static curve.

On [Native Core](../modules/native-core.md), this pricing is expressed directly as maker orders on the [CLOB](central-limit-orderbook.md), where professional market makers connect through [Native Pro](../modules/native-pro.md) to post real-time bid/ask. Integrators reach the same liquidity through [Native Relay](../modules/native-relay.md):

#### On-chain order book

The Native Core order book returns the best pricing for each pair, with levels representing the available liquidity at each price. Orders match deterministically by price-time priority.

#### Firm quote (RFQ)

For integrators using [RFQ mode](integration-modes.md), [Native Relay](../modules/native-relay.md) returns a [firm quote](firm-quote.md) with signed, executable calldata for a given swap. If the quote is favourable, the calldata is executed before its expiration to complete the swap.
