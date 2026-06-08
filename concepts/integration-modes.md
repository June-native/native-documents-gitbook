# Liquidity Relaying

Native Core liquidity is accessible through Native Relay in two integration modes, each suited to a different venue type and user flow. Both are compatible with how existing on-chain swap infrastructures operate.

#### Mode — RFQ

The user requests firm quotes on public networks.

* Native Relay generates firm-quote calldata based on Native Core liquidity.
* Lowest-friction for existing DEX aggregators, wallets, and solvers.

#### Mode — Intent

The user commits an intent on public networks.

* Native Relay executes the trade on Native Core, and settles with the user on public network.
* Ideal for meta-aggregators, intent platforms, and cross-chain UI.

#### Mode — Direct

* Users deposit and trade directly on Native Core.
* Ideal for users who want full CLOB access with all the available order types. (limit, market)

#### Choosing a Mode

<table><thead><tr><th width="419.18359375">Partner type</th><th>Recommended mode</th></tr></thead><tbody><tr><td>DEX Aggregators / Wallets / Solvers</td><td>Mode — RFQ</td></tr><tr><td>Meta-Aggregator / Intent &#x26; Cross-chain Platforms</td><td>Mode — Intent</td></tr><tr><td>Venues wanting full CLOB access</td><td>Mode — Direct</td></tr></tbody></table>
