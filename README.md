---
cover: .gitbook/assets/Frame 4.png
coverY: 0
layout:
  width: default
  cover:
    visible: true
    size: full
  title:
    visible: true
  description:
    visible: true
  tableOfContents:
    visible: true
  outline:
    visible: true
  pagination:
    visible: true
  metadata:
    visible: true
  tags:
    visible: true
  actions:
    visible: true
---

# What is Native

**Native** is a non-custodial, autonomous trading infrastructure for value exchange. It is the first vertically integrated liquidity stack that **decouples pricing logic from asset settlement**: a sovereign, on-chain matching engine finalizes price discovery and order matching, while distributed vaults custody assets and enforce settlement across chains.

The result is institutional-grade liquidity that is **permissionless, non-custodial, and fully transparent** — exported to any chain and any distribution surface, from wallets and aggregators to professional market makers.

<figure><img src=".gitbook/assets/Overview.png" alt=""><figcaption></figcaption></figure>

## Why Native

The industry has spent years trying to force high-performance trading into general-purpose environments, leaving traders stuck with opaque off-chain pricing engines, high-latency RFQ models, or MEV-exploited AMMs:

* **AMMs are a poor fit for professional execution at scale.** Concentrated liquidity improves spreads but still breaks under large or tail-risk flows. Capital is fragmented across pools and chains, and standard order types (limit, IOC, post-only) cannot be expressed cleanly.
* **Most "on-chain CLOB" designs force a hard compromise.** They either stay on a general-purpose chain and sacrifice performance, or move off-chain and sacrifice verifiability. Liquidity and margin then end up siloed per venue and per chain.
* **Demand for on-chain price discovery is rising faster than the plumbing.** RWAs, perps, and prediction markets increasingly need transparent, order-book-style execution, but there has been no permissionless, composable, non-custodial liquidity layer that can serve many chains.

Native is built to close that gap: an on-chain price discovery and execution system with the performance of a centralized exchange and the transparency of a public chain.

## The Native Stack

Native is organized into four modules that work end-to-end:

<table data-view="cards"><thead><tr><th></th><th></th><th data-hidden data-card-target data-type="content-ref"></th></tr></thead><tbody><tr><td><strong>Native Core</strong></td><td>The on-chain, high-performance CLOB app chain where pricing logic and order matching are finalized — fully on-chain and auditable.</td><td><a href="modules/native-core.md">native-core.md</a></td></tr><tr><td><strong>Native Pool</strong></td><td>EVM vaults that custody assets and enforce settlement, giving users one unified balance to trade with.</td><td><a href="modules/native-pool.md">native-pool.md</a></td></tr><tr><td><strong>Native Pro</strong></td><td>The direct-access layer that exposes Core's matching engine to professional market makers via exchange-grade interfaces.</td><td><a href="modules/native-pro.md">native-pro.md</a></td></tr><tr><td><strong>Native Relay</strong></td><td>The integrator SDK/API that streams Core prices and routes orders from wallets, aggregators, and solvers into Native.</td><td><a href="modules/native-relay.md">native-relay.md</a></td></tr></tbody></table>

{% hint style="info" %}
Looking for the previous credit-pool and RFQ-based design? It now lives under [**Native Relay**](modules/native-relay.md), with detailed references preserved in the [Legacy (V2)](legacy-v2/about-native-v2.md) section.
{% endhint %}
