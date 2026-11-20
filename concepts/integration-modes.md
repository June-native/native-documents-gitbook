# Integration Modes

Native liquidity is accessible on public EVM chains in the following modes. Both are compatible with how existing on-chain swap infrastructure operates.

#### Mode — RFQ

The user requests a firm quote and executes it on-chain.

* The [Native RFQ API](../modules/native-relay.md) returns a signed firm quote and calldata for the Native Router.
* Lowest-friction path for existing DEX aggregators, wallets, and solvers.

See [FirmQuote Swap APIs](../build-with-native/swap-aggregators/firmquote-swap-apis/ "mention").

#### Mode — pAMM

The user quotes and swaps fully onchain.

* No offchain quote API and no RFQ signature. Call `PropAMMEngine.getQuote`, then `NativeRouter.tradePAMM`.
* Single hop only. There is no RFQ fallback on the direct path.

See [pAMM Swap APIs](../build-with-native/swap-aggregators/pamm-swap-apis/ "mention").

#### Choosing a Mode

<table><thead><tr><th width="419.18359375">Partner type</th><th>Recommended mode</th></tr></thead><tbody><tr><td>DEX Aggregators / Wallets / Solvers</td><td>Mode — RFQ</td></tr><tr><td>DEX Aggregators wanting an onchain-only AMM path</td><td>Mode — pAMM</td></tr></tbody></table>
