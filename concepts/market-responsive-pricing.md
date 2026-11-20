---
hidden: true
noIndex: true
---

# Market-Responsive Pricing

Native pricing is **Market-Responsive**: market makers price each token accurately and promptly based on market risk, conditions, and inventory exposure. Because of this, quotes reflect liquidity sourced from on-chain DEXs, institutional venues, and/or direct redemption channels — ultimately responding to the real market rather than a static curve.

Professional market makers push those prices into Native through the [Orderbook Push API](../build-with-native/market-makers/). Integrators reach the resulting quotes through the [Native RFQ API](../modules/native-relay.md):

#### Order book

Native returns the best pricing for each pair, with levels representing the available liquidity at each price.

#### Firm quote (RFQ)

For integrators using [RFQ mode](integration-modes.md), the [Native RFQ API](../modules/native-relay.md) returns a [firm quote](firm-quote.md) with signed, executable calldata for a given swap. If the quote is favourable, the calldata is executed before its expiration to complete the swap.
