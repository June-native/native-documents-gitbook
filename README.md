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

Native is a Non-custodial, Autonomous Trading Infrastructure for Value Exchange.

It is the first vertically integrated liquidity stack that **decouples pricing logic from asset settlement**: a sovereign, on-chain matching engine provides high-performance price discovery and order matching, while permissionless vaults host assets for settlement across different chains.

The result is institutional-grade liquidity that is **permissionless, non-custodial, and fully transparent** — seamlessly integrated to any chain and any distribution surface, from wallets and aggregators to professional market makers.

## Why Native

The industry has spent years trying to force high-performance trading into general-purpose environments, leaving traders stuck with opaque off-chain pricing engines, high-latency RFQ models, or MEV-exploited AMMs.

Native is built to close that gap: an on-chain price discovery and execution system with ultra high-performance order matching, deterministic settlement and proactive liquidity.

## What We Are Building

<table data-card-size="large" data-view="cards"><thead><tr><th></th><th></th><th></th></tr></thead><tbody><tr><td><a href="modules/native-core.md"><strong>Native Core</strong></a></td><td>The on-chain sovereign state machine where price discovery and order matching are finalized.</td><td></td></tr><tr><td><a href="modules/native-pool.md"><strong>Native Pool</strong></a></td><td>Distributed smart contracts on EVM chains that custody assets and enforce settlement.</td><td></td></tr><tr><td><a href="modules/native-pro.md"><strong>Native Pro</strong></a></td><td>The institution integration module that exposes Core’s matching engine via direct, exchange-grade interfaces.</td><td></td></tr><tr><td><a href="modules/native-relay.md"><strong>Native Relay</strong></a></td><td>The wallet/aggregator module that streams Core prices and routes orders into Native.</td><td></td></tr></tbody></table>
