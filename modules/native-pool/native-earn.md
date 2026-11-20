# Native Pool Yield

Native Pool is Native's earn product, with a separate pool on each supported EVM chain: deposit supported assets to earn yield, and withdraw them when you need to.

Native Pool is designed for users who want to earn on assets that are actively used for trading liquidity but without the need for actively managing a position or perform complex liquidity provisions.

### How yield works

Yield is supported by demand for the pool's capital:

* **Market Makers:** verified market makers utilise the capital to support quoting and settlement, generating funding fees.
* **Swaps:** swaps quoted through the [Native RFQ API](../native-relay.md) settle against the pool on each EVM chain, generating trading fees.

Native distributes yield to eligible earning balances. The amount you receive is based on your eligible balance for each asset when a distribution is made. The app shows current yield information and asset-specific terms.

### Deposit and start earning

Deposit a supported asset into the Native Pool on its EVM chain. Once the deposit is complete, the balance is eligible for subsequent yield distributions. Minimum deposit requirements are displayed before you confirm a deposit.

### Withdraw your assets

Native Pool offers two withdrawal options:

* **Scheduled withdrawal:** move an amount from your earn balance into a pending withdrawal. Claim it after the pending period shown in the app.
* **Instant withdrawal:** receive the available amount immediately, less the fee shown before confirmation.

An amount in a scheduled withdrawal stops earning yield. Before it becomes claimable, you can cancel a pending request to return the amount to your earn balance. Only one scheduled withdrawal can be pending at a time.

### Existing positions

If you have a legacy position, use the Legacy entry in the app to review it. Eligible positions can be migrated into Native Pool.

### Important considerations

Yield is variable and depends on product activity. The availability, terms, and fees for each supported asset may change. Review the app carefully before you deposit or withdraw.
