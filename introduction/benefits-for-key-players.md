---
cover: ../.gitbook/assets/Frame 3.png
coverY: 0
---

# Benefits for Key Players

{% tabs %}
{% tab title="Traders" %}
Get an institutional-grade experience with fully on-chain visibility:

* **High-performance CLOB execution:** Real order-book trading with limit and market orders, deterministic order matching, and sub-second finality.
* **On-chain transparency:** Every order, match, and balance update is publicly auditable.
* **Unified cross-chain balance:** Deposit from any supported network and hold a **single balance** for trading.
{% endtab %}

{% tab title="Asset Issuers" %}
Bootstrap real order-book liquidity and price discovery:

* **Real CLOB listing:** List and bootstrap liquidity with a real central limit order book on Native Core.
* **Capital-efficient provision:** Native's market infrastructure helps market makers use capital efficiently, supporting deeper liquidity at lower cost.
* **Chain abstraction:** Unified, multi-chain liquidity with Native Relay.
{% endtab %}

{% tab title="Market Makers" %}
Push your book. Native quotes and signs:

* **You push, we quote:** Push bids and asks for your assigned pairs. Native prices every firm quote from your latest valid book, applies its own fee, and signs the quote — no last look, no callback.
* **Inbound only:** Native never fetches prices from you or asks you to sign. A signed quote is final for its full validity.
* **Two transports, one semantics:** HTTPS or WebSocket. Same book format, same validation.

See [External Market Maker Orderbook Push API](../build-with-native/market-makers/).
{% endtab %}

{% tab title="Wallets, Aggregators & Solvers" %}
Connect your existing order flow:

* **Backward-compatible:** Integrate via Native Relay mode RFQ or mode intent without changing any userflow.
* **Best execution, high success rate:** Access Native Core prices, and route trades with deep, transparent liquidity and ultra low latency.
* **Single point of access:** One integration reaches the entire Native Core liquidity across all supported chains.
{% endtab %}
{% endtabs %}
