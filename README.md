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

**Native** is a **N**on-custodial, **A**utonomous **T**rading **I**nfrastructure for **V**alue **E**xchange.&#x20;

It is the first vertically integrated liquidity stack that **decouples pricing logic from asset settlement**: a sovereign, on-chain matching engine provides high performance price discovery and order matching, while permission-less vaults host assets for settlement across different chains.

The result is institutional-grade liquidity that is **permissionless, non-custodial, and fully transparent** — exported to any chain and any distribution surface, from wallets and aggregators to professional market makers.

## Why Native

The industry has spent years trying to force high-performance trading into general-purpose environments, leaving traders stuck with opaque off-chain pricing engines, high-latency RFQ models, or MEV-exploited AMMs.

Native is built to close that gap: an on-chain price discovery and execution system with the performance of a centralized exchange and the transparency of a public network.

## What We Are Building

Native Core finalizes pricing logic and order matching on-chain, while Native Pool anchors asset custody and settlement — with Native Pro and Native Relay connecting professional market makers and distribution partners end-to-end.

<table data-card-size="large" data-view="cards"><thead><tr><th></th><th></th></tr></thead><tbody><tr><td><strong>Native Core</strong></td><td>The on-chain sovereign state machine where pricing logic and order matching are finalized.</td></tr><tr><td><strong>Native Pool</strong></td><td>Distributed smart-contract vaults across various chains that custody assets and enforce settlement.</td></tr><tr><td><strong>Native Pro</strong></td><td>The MM integration module that exposes Core’s matching engine via direct, exchange-grade interfaces.</td></tr><tr><td><strong>Native Relay</strong></td><td>The trade integrator SDK/API that streams Core prices and routes orders from wallets, aggregators, and solvers into Native.</td></tr></tbody></table>
