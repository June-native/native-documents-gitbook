# Liquidity Relaying

Native Core liquidity is accessible through Native Relay in two integration modes, each suited to a different venue type and user flow. Both are **backward-compatible** with how existing partners already operate.

### Mode — Intent

The user commits an intent on a **public network** — through the Native UI or an external intent entry point.

* Native Relay (Intent adapter) detects the commitment, executes the trade on Native Core, and confirms.
* The adapter then realizes and pays the user on the public network.
* Used by meta-aggregators, intent platforms and solvers, and bridge UIs, where Native Core appears as a standalone provider or solver.

### Mode — RFQ

The user requests a **firm quote** on a public network.

* Native Relay (RFQ adapter) generates firm-quote calldata based on latency-adjusted Native Core CLOB liquidity.
* Once the user confirms and the trade settles on the public network, the adapter mirrors the trade on Native Core.
* Lowest-friction path for existing DEX aggregators — no taker-side flow change. This is also where the previous V2 RFQ model lives.

### Choosing a Mode

| Partner type                        | Recommended mode |
| ----------------------------------- | ---------------- |
| DEX aggregators                     | Mode — RFQ       |
| Intent / gasless / solver platforms | Mode — Intent    |
| Surfaces wanting full CLOB access   | Mode — Direct    |

