---
description: The Direct-Access Layer for Professional Market Makers
---

# Native Pro

**Native Pro** is the direct-access layer that exposes [Native Core](native-core.md)'s matching engine to **professional market makers** through direct, exchange-grade interfaces. It is the on-chain equivalent of a CEX market-making connection: real-time bid/ask, deterministic matching, and full order management — settled transparently on Native Core.

It is built for trading firms and quant teams who make markets on a CLOB (not AMM LPing) and want CEX-grade execution with on-chain guarantees.

### What It Provides

* **CEX-grade execution on-chain:** stream real-time bid/ask and place, amend, and cancel maker orders directly on the Native Core CLOB.
* **Deterministic matching:** price-time-priority with limit and market orders; consensus-ordered, so there is no opaque front-running of your quotes.
* **Capital-efficient margin:** onboard via [spot margin](../concepts/spot-margin.md) accounts — post collateral and trade against credited available margin without locking large static inventory.
* **Programmatic access:** standardized APIs/SDKs for low-latency, automated strategies.

### How It Fits

Market makers connect through Native Pro to provide depth; platform-owned [Auto LP](../concepts/auto-lp.md) supplies baseline liquidity for new markets, and professional MMs layer additional depth on top. Taker flow arrives both directly on Native Core and through [Native Relay](native-relay.md) integrators.

### Build with Native Pro

Pricing, signing, and market-maker APIs are documented under Market Makers.

{% content-ref url="../build-with-native/market-makers/" %}
[market-makers](../build-with-native/market-makers/)
{% endcontent-ref %}
