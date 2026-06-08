---
description: Streams Prices and Liquidity to Existing Channels.
---

# Native Relay

**Native Relay** an integrator SDK/API that **streams Native Core prices and routes orders into Native** on behalf of wallets, aggregators, and solvers.&#x20;

It lets existing distribution partners tap Native liquidity **without forcing their users to change behavior**.

#### Relay Modes

Relay meets each partner where they already operate. See [Integration Modes](../concepts/integration-modes.md) for the full breakdown.

* **RFQ mode:** the partner requests a **firm quote**; Relay returns firm-quote calldata backed by Native Core CLOB liquidity. Lowest-friction path for existing DEX aggregators.
* **Intent mode:** the user commits an intent on a public network; Relay detects the commitment, uses Native Core to execute and settles back on the public network. Used for meta-aggregators, intent platforms, solvers, and bridge UIs.

#### Who It's For

* **DEX aggregators** (1inch, OKX DEX, KyberSwap, etc.) — Native Core appears as a DEX liquidity source via firm quotes.
* **Wallets & meta-aggregators** — Native Core appears as a standalone swap/bridge provider.
* **Intent platforms & solvers** (e.g. UniswapX, CoW Protocol) — Native Core acts as a solver, filling winning intents and settling on the source chain.

#### Compatibility with the Native V2

Native's earlier Swap API product now lives inside **Native Relay's RFQ mode** — the same firm-quote, backward-compatible integration partners already use — while pricing, matching, and settlement move on-chain to Native Core and Native Pool.

{% hint style="info" %}
The detailed V2 documentation (Credit Pool, Swap Engine, credit-based swap, single-sided liquidity, etc.) is preserved under the [Legacy (V2)](../legacy-v2/about-native-v2.md) section for reference.
{% endhint %}

### Build with Native Relay

Existing RFQ swap APIs for aggregators and integrators are documented under Swap Aggregators.

{% content-ref url="../build-with-native/swap-aggregators/" %}
[swap-aggregators](../build-with-native/swap-aggregators/)
{% endcontent-ref %}
