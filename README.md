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

Native is a Non-custodial, Autonomous Trading Infrastructure for Value Exchange (NATIVE).

It is the first vertically integrated liquidity stack that **decouples pricing logic from asset settlement**: a sovereign, on-chain matching engine provides high-performance price discovery and order matching, while permissionless vaults host assets for settlement across different chains.

The result is institutional-grade liquidity that is **permissionless, non-custodial, and fully transparent** — seamlessly integrated to any chain and any distribution surface, from wallets and aggregators to professional market makers.

## Why Native

The industry has spent years trying to force high-performance trading into general-purpose environments, leaving traders stuck with opaque off-chain pricing engines, high-latency RFQ models, or MEV-exploited AMMs.

Native is built to close that gap: an on-chain price discovery and execution system with the performance of a centralized exchange and the transparency of a public network.

## What We Are Building

<table data-card-size="large" data-view="cards"><thead><tr><th></th><th></th></tr></thead><tbody><tr><td><strong>Native Core</strong></td><td>The on-chain sovereign state machine, where pricing logic and order matching are finalised.</td></tr><tr><td><strong>Native Pool</strong></td><td>The smart-contract vaults across various chains that host LP assets and enforce settlement.</td></tr><tr><td><strong>Native Pro</strong></td><td>The integration module that provides direct access to Native Core via exchange-grade interfaces.</td></tr><tr><td><strong>Native Relay</strong></td><td>The trade integrator SDK/API that streams Native Core prices and routes trades from wallets, aggregators, and solvers into Native.</td></tr></tbody></table>
