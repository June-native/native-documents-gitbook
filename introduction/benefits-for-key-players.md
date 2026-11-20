---
cover: ../.gitbook/assets/Frame 3.png
coverY: 0
---

# Benefits for Key Players

{% tabs %}
{% tab title="Asset Issuers" %}
Bootstrap firm, market-maker-backed liquidity:

* **Professional quoting:** Market makers push order books for your asset, and Native signs firm quotes from them with no last look.
* **Capital-efficient provision:** Native's market infrastructure helps market makers use capital efficiently, supporting deeper liquidity at lower cost.
* **Broad distribution:** Reach wallets, aggregators, and solvers through the Native RFQ API on every supported EVM chain.
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

* **Backward-compatible:** Integrate the Native RFQ API without changing any user flow.
* **Best execution, high success rate:** Access Native prices, and route trades with deep, transparent liquidity and ultra low latency.
* **Single point of access:** One integration reaches Native liquidity across all supported chains.
{% endtab %}
{% endtabs %}
