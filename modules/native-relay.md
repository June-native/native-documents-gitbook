---
description: Enables Direct-Access for Professional Market Makers
---

# Native Relay

**Native Relay** is an integrator SDK/API that streams prices and routes trades into Native Core.

It lets existing distribution partners tap into the deep Native Core liquidity **without changing any user flow**.

#### Modes

Relay meets each partner where they already operate. See [Integration Modes](../concepts/integration-modes.md) for the full breakdown.

* **RFQ mode:** Lowest-friction path for existing DEX aggregators.
* **Intent mode:** Used for meta-aggregators, intent platforms, solvers, and cross-chain UIs.

#### Who It's For

* **DEX aggregators** — Native Core as a DEX liquidity source.
* **Wallets & Meta-aggregators** — Native Core as a swap service provider.
* **Intent platforms & solvers** — Native Core as a solver.

#### Compatibility with Native V2

Native's earlier V2 Swap API product now lives inside **Native Relay's mode RFQ.**&#x20;

The same firm-quote, backward-compatible integration partners already use — while pricing, matching, and settlement move on-chain to Native Core.

### Build with Native Relay

Existing RFQ swap APIs for aggregators and integrators are documented under Swap Aggregators.

{% content-ref url="../build-with-native/swap-aggregators/" %}
[swap-aggregators](../build-with-native/swap-aggregators/)
{% endcontent-ref %}
