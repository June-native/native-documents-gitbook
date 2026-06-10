---
hidden: true
noIndex: true
---

# Auto LP

**Auto LP** is an automated, transparent market-making service that **bootstraps order-book depth** for tokens on [Native Core](../modules/native-core.md) from day one — solving the cold-start liquidity problem for new markets without requiring a dedicated market maker.

It is conceptually similar to position managers like RangeProtocol or Uniswap V3 managers, but operates on a CLOB.

### Architecture: Offchain Service, On-Chain Config

* **On-chain config & inventory:** a market owner deploys a configuration to Native Core (validated through consensus) that locks inventory and defines market-making parameters.
* **Offchain execution:** the Auto LP service reads that on-chain config and automatically places and manages orders on the CLOB, as fast as possible — it runs offchain and does **not** slow down block production.
* **Fully verifiable:** every order action and any timing delay is trackable on-chain, eliminating suspicion of manipulation.

### Platform-Owned Liquidity & NLP Tiers

Auto LP underpins Native's platform liquidity strategy, which is organized into pool tiers based on token profile:

* **Small Token Pool** — open, automated, and permissionless. Anyone can participate; liquidity provision is fully automated with no dedicated MM. High gain, high risk. Issuers can deposit their own token and add incentives to attract LPs. Analogous to launching a pool on an AMM, but on Native Core.
* **Major Token Pool** — credentialed and collateralized. Only KYB'd, credentialed market makers participate; they post collateral and receive pool liquidity allocations based on margin. Lower return, significantly lower (but non-zero) risk, with platform and MMs sharing responsibility.

### Monitoring

Auto LP performance is tracked on PnL, fill rate, and volume — all order actions and timing are publicly auditable on-chain.

### Related

* [Native Pro](../modules/native-pro.md)
* [Central Limit Order Book (CLOB)](central-limit-orderbook.md)
