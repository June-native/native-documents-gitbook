---
cover: ../.gitbook/assets/Frame 3.png
coverY: 0
---

# Benefits for Key Players

{% tabs %}
{% tab title="Traders" %}
Traders get a centralized-exchange-grade experience with full on-chain guarantees, powered by [**Native Core**](../modules/native-core.md)**:**

* **CEX-grade execution on-chain:** Real order-book trading with limit and market orders, deterministic matching, and sub-second finality.
* **Unified cross-chain balance:** Deposit from any supported network and hold a **single balance** to trade.
* **Transparent, MEV-resistant ordering:** Every order, match, and balance update is consensus-verified and publicly auditable — no opaque front-running.
* **Easy swap or full CLOB:** Use one-click swap orders for simplicity, or the full order book for precision.
{% endtab %}

{% tab title="Asset Issuers" %}
Asset issuers can bootstrap real order-book liquidity and price discovery for their tokens:

* **Real CLOB listing:** List on a professional order book with proper depth instead of a passive AMM pool.
* **Automated bootstrapping:** Transparent, verifiable Auto LP maintains bid/ask depth for new tokens from day one. See [Auto LP](../concepts/auto-lp.md).
* **Capital-efficient provision:** Active utilization keeps capital working, lowering the cost of establishing secondary-market liquidity.
* **Chain abstraction:** Reach unified, multi-chain liquidity without redundant capital deployment on every chain.
{% endtab %}

{% tab title="Market Makers" %}
Professional market makers plug directly into Native via [**Native Pro**](../modules/native-pro.md)**:**

* **Direct, exchange-grade access:** Stream real-time bid/ask and place maker orders on the Native Core CLOB through the Native Pro direct-access layer.
* **Capital-efficient:** Onboard via credit margin — trade and make market instantly without maintaining large, static inventories.
{% endtab %}

{% tab title="LPs" %}
Liquidity providers earn by depositing with Native Core into [Native Pool](../modules/native-pool.md):

* **Unified capital experience:** One balance to earn yield and trading fees across pairs; single-asset liquidity provision designed to minimize market risk for LPs.
* **Deposit and forget about it:** Use your Native Core balance to earn directly without taking any extra steps to stake. While using the balance to trades directly.
{% endtab %}

{% tab title="Integrators" %}
Wallets, aggregators, and solvers connect their order flow to Native liquidity through [**Native Relay**](../modules/native-relay.md):

* **Backward-compatible distribution:** Integrate via RFQ firm quotes or intent/solver flows without forcing end users to change behavior.
* **Best execution, high success rate:** Stream Core prices and route orders into deep, transparent liquidity with low latency.
* **Multiple integration modes:** Choose the mode that fits your surface — direct, intent-based, or RFQ. See [Integration Modes](../concepts/integration-modes.md).
* **Single point of access:** One integration reaches Native Core liquidity across all supported chains.
{% endtab %}
{% endtabs %}
