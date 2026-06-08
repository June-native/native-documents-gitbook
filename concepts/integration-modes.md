# Integration Modes

Native Core liquidity is accessible through three integration modes, each suited to a different venue type and user flow. Modes B and C are delivered through [Native Relay](../modules/native-relay.md) and are designed to be **backward-compatible** with how existing partners already operate.

### Mode A — Native (Direct)

The user deposits to [Native Core](../modules/native-core.md), trades on-chain, and withdraws.

* Full CLOB experience — limit orders, order management, order-book depth.
* Also supports **easy swap** — multihop market orders / one-click swaps for a simpler UX.
* Settlement is entirely on Native Core; cross-chain movement happens only at deposit and withdrawal.
* Used by the Native dapp and by wallets that add Native Core as a network.

### Mode B — Intent

The user commits an intent on a **public network** — either an on-chain commitment or a gasless signature — through the Native UI or an external intent entry point.

* [Native Relay](../modules/native-relay.md) (Intent adapter) detects the commitment, executes the trade on Native Core, and confirms.
* The adapter then realizes and pays the user on the public network.
* Used by meta-aggregators, intent platforms and solvers, and bridge UIs, where Native Core appears as a standalone provider or solver.

### Mode C — RFQ

The user requests a **firm quote** on a public network.

* [Native Relay](../modules/native-relay.md) (RFQ adapter) generates firm-quote calldata based on latency-adjusted Native Core CLOB liquidity.
* Once the user confirms and the trade settles on the public network, the adapter mirrors the trade on Native Core.
* Lowest-friction path for existing DEX aggregators — no taker-side flow change. This is also where the previous V2 RFQ model lives.

### Choosing a Mode

| Partner type                                | Recommended mode |
| ------------------------------------------- | ---------------- |
| DEX aggregators (1inch, OKX DEX, KyberSwap) | Mode C — RFQ     |
| Intent / gasless / solver platforms         | Mode B — Intent  |
| Surfaces wanting full CLOB access           | Mode A — Direct  |

### Related

* [Native Relay](../modules/native-relay.md)
* [Firm Quote](firm-quote.md)
