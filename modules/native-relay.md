---
description: Firm-quote swap API for wallets, aggregators, and solvers
---

# Native RFQ API

The **Native RFQ API** gives wallets, aggregators, and solvers firm, executable quotes on each supported EVM chain.

Professional market makers push their order books to Native through the [Orderbook Push API](../build-with-native/market-makers/). Native prices each request from the latest valid books, signs a firm quote, and returns calldata for the Native Router. The swap settles on-chain on the same chain, backed by the [Native Pool](native-pool/) liquidity there.

* **Order book:** `GET /orderbook` publishes price levels for every pair Native quotes on a chain.
* **Indicative quote:** `GET /indicative-quote` returns a price for a given pair and amount.
* **Firm quote:** `GET /firm-quote` returns a signed quote and executable calldata. A signed quote is final for its validity window — there is no last look.

#### Who it's for

* **DEX aggregators** — Native as an RFQ liquidity source.
* **Wallets & swap widgets** — Native as a swap provider.
* **Solvers** — Native as a source of firm quotes to fill intents.

#### Compatibility with Native V2

The Native RFQ API keeps the V2 Swap API's firm-quote interface, so existing integrations continue to work.

### Build with the Native RFQ API

{% content-ref url="../build-with-native/swap-aggregators/" %}
[swap-aggregators](../build-with-native/swap-aggregators/)
{% endcontent-ref %}
