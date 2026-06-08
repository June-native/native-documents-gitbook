# On-chain Verifiability

The core promise of [Native Core](../modules/native-core.md) is that trading is **verifiable**: the sequence of every transaction, the result of every match, and every balance change are produced by consensus and published on-chain, so anyone can independently audit them.

### How Trust Is Established

* **Consensus-ordered transactions:** orders, cancels, and modifies are submitted as transactions. Consensus assigns each a canonical position in the sequence before any matching happens.
* **Deterministic execution:** nodes execute matching deterministically over that canonical order, so the same sequence always produces the same order book and balances.
* **Finality:** once a block is finalized, its transactions and their effects are settled and cannot be reordered.

Because ordering is fixed by consensus rather than by an opaque operator, there is no room for hidden front-running or selective sequencing.

### Reading Verified State

Public query responses are anchored to a specific point in the chain's history so clients can reason about freshness and reproduce results:

* **`query_height`** — the execution height represented by the published read view.
* **`app_hash`** — the application state commitment at that height, which clients can use to verify that a response reflects canonical state.

If a node's read view falls behind execution, queries surface backpressure (e.g. `QueryLagBackpressure`) rather than returning stale data silently.

### Related

* [Central Limit Order Book](central-limit-orderbook.md)
* [Native Core](../modules/native-core.md)
