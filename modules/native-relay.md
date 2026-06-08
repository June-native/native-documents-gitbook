---
description: The Integrator SDK/API That Streams Core Prices and Routes Orders into Native
---

# Native Relay

**Native Relay** is the distribution layer of the Native stack: an integrator SDK/API that **streams [Native Core](native-core.md) prices and routes orders into Native** on behalf of wallets, aggregators, and solvers. It lets existing distribution partners tap Native liquidity **without forcing their users to change behavior**.

Relay is also the home of Native's original RFQ-based distribution model (see [Compatibility with the V2 model](#compatibility-with-the-v2-model) below).

### Integration Modes

Relay meets each partner where they already operate. See [Integration Modes](../concepts/integration-modes.md) for the full breakdown.

* **RFQ mode:** the partner requests a **firm quote**; Relay returns firm-quote calldata backed by latency-adjusted Native Core CLOB liquidity, and mirrors the trade on Core once it settles on the public network. Lowest-friction path for existing DEX aggregators.
* **Intent mode:** the user commits an intent on a public network (on-chain or gasless); Relay detects the commitment, executes on Native Core, and realizes the settlement back on the public network. Used for meta-aggregators, intent platforms, solvers, and bridge UIs.

### Who It's For

* **DEX aggregators** (1inch, OKX DEX, KyberSwap, etc.) — Native Core appears as a DEX liquidity source via firm quotes.
* **Wallets & meta-aggregators** — Native Core appears as a standalone swap/bridge provider.
* **Intent platforms & solvers** (e.g. UniswapX, CoW Protocol) — Native Core acts as a solver, filling winning intents and settling on the source chain.

### Compatibility with the V2 Model

Native's earlier design delivered liquidity through an RFQ/credit-pool model: private market makers (PMMs) quoted firm prices and drew inventory from a shared, single-sided **Credit Pool**, executed via an auto-sign on-chain orderbook. That distribution capability now lives inside **Native Relay's RFQ mode** — the same firm-quote, backward-compatible integration partners already use — while pricing, matching, and settlement move on-chain to Native Core and Native Pool.

{% hint style="info" %}
The detailed V2 documentation (Credit Pool, Swap Engine, credit-based swap, single-sided liquidity, etc.) is preserved under the [Legacy (V2)](../legacy/about-native-v2.md) section for reference.
{% endhint %}

### Build with Native Relay

Existing RFQ swap APIs for aggregators and integrators are documented under Swap Aggregators.

{% content-ref url="../build-with-native/swap-aggregators/README.md" %}
[README.md](../build-with-native/swap-aggregators/README.md)
{% endcontent-ref %}
