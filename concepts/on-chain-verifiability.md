# On-chain Verifiability

With Native Core, trading is **verifiable**: the sequence of every transaction, the result of every match, and every balance change are produced by consensus and published on-chain, so anyone can independently audit them.

#### How Trust Is Established

* **Publicly verifiable:** orders, cancels, and modifies are submitted as transparent transactions.
* **Deterministic execution:** nodes execute matching deterministically over the canonical order, so the same sequence always produces the same order book and balances.
* **Finality:** once a block is finalized, its transactions and their effects are settled and cannot be reordered.
