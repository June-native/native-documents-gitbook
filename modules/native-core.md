---
description: The On-Chain, High-Performance CLOB Where Pricing and Matching Are Finalized
---

# Native Core

**Native Core** is a high-performance central limit order book (CLOB) application chain — the sovereign state machine where Native's **pricing logic and order matching are finalized on-chain**. Core refers to the chain itself: consensus orders every transaction, and nodes execute matching deterministically, producing a canonical, auditable sequence of every order, fill, and balance update.

It is built for traders and validators who want centralized-exchange-grade performance with full on-chain verifiability.

### Why a Dedicated App Chain

General-purpose chains force a trade-off between performance and verifiability. By running on a purpose-built chain, Native Core removes gas competition and MEV uncertainty while keeping the entire matching process transparent.

* **Fully on-chain verifiable:** ordering, matching, and state updates are all on-chain and publicly auditable.
* **Deterministic matching:** price-time-priority CLOB with limit and market orders, partial fills, and self-trade prevention.
* **High throughput, low latency:** targeting >200k TPS with sub-second finality via a Jolteon (HotStuff-derivative) consensus and a permissioned validator set.
* **Spot-first, perp-ready:** designed to host spot and perpetual markets in a single venue.

### How an Order Flows

1. A client submits a signed trading action as an on-chain transaction (`POST /trade`).
2. Consensus assigns the transaction a canonical position in the sequence.
3. Nodes execute matching deterministically and finalize both order-book state and account balances.
4. Fills, cancellations, and balances are reflected on-chain and queryable (`POST /info`).

Order lifecycle: `Pending` → `Open` → `(PartiallyFilled)` → `Filled` / `Cancelled`.

### Related Concepts

* [Central Limit Order Book (CLOB)](../concepts/central-limit-orderbook.md)
* [Unified Balance](../concepts/unified-balance.md)
* [Auto LP](../concepts/auto-lp.md)

### Build on Native Core

Native Core exposes a business API for submitting trades and reading state.

{% content-ref url="../build-with-native/native-core/" %}
[native-core](../build-with-native/native-core/)
{% endcontent-ref %}
