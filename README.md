---
cover: .gitbook/assets/Frame 4.png
coverY: 0
layout:
  width: default
  cover:
    visible: true
    size: full
    mask: none
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

Native is non-custodial liquidity infrastructure for on-chain swaps.

Professional market makers push their order books to Native. Native prices every swap from those books, signs a firm quote, and settles it on-chain on the same EVM chain, backed by the Native Pool there.

The result is RFQ liquidity that wallets, aggregators, and solvers can reach through one API, with no last look on the quotes Native signs.

## Why Native

Swap liquidity is usually a choice between AMMs that leak value to MEV and RFQ venues where makers can reject a trade after the user commits.

Native is built to close that gap: makers push firm books, Native quotes and signs on their behalf, and every quote it signs settles on-chain.

## What We Are Building

<table data-card-size="large" data-view="cards"><thead><tr><th></th><th></th><th></th></tr></thead><tbody><tr><td><a href="modules/native-relay.md"><strong>Native RFQ API</strong></a></td><td>The swap API for wallets, aggregators, and solvers: order books, indicative quotes, and signed firm quotes with executable calldata.</td><td></td></tr><tr><td><a href="modules/native-pool/"><strong>Native Pool</strong></a></td><td>Per-chain liquidity pools on each supported EVM chain. Supply assets to earn yield from the swaps they settle.</td><td></td></tr><tr><td><a href="build-with-native/market-makers/"><strong>Market Makers</strong></a></td><td>Push bids and asks to Native over HTTPS or WebSocket. Native prices and signs every quote.</td><td></td></tr></tbody></table>
