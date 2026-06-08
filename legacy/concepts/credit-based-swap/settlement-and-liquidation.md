# Settlement and Liquidation

With each credit-based swap, **PMM** will result in multiple long or short positions in **Native Credit Pool**. The **PMM** may choose to settle those positions and reclaim any available tokens. Conversely, in extreme circumstances where the amount of available credit falls below zero, an automated liquidation process will be initiated to settle the positions.

### Example Flow

**PMM** faciliated a swap between 3600 USDC and 1 WETH. Now PMM has a short position of 1 WETH and a long position of 3600 USDC.

Assuming WETH price remain at 3600 USD, no credits from collateral is now required or consumed.

#### Settlement

<figure><img src="../../../.gitbook/assets/PMM Settlement.png" alt=""><figcaption></figcaption></figure>

1. **PMM** now wants to settle the short position
2. 1 WETH repaid by the **PMM** to Credit Pool
3. Available credit of **PMM** goes up by 3600 USD
4. **PMM** will now have enough remaining credit to settle the 3600 USDC long position. Claiming the 3600 USDC from the Credit Pool.

**PMM** may opt to complete the settlement of multiple positions in a single transaction, provided that the settlement outcome does not yield a negative credit.

### Liquidation

<figure><img src="../../../.gitbook/assets/PMM Liquidation (1).png" alt=""><figcaption></figcaption></figure>

1. Assume **PMM** did not settle the position and initially provided 100 USDC as collateral for 100 USD credit.
2. The price of WETH suddenly increased from 3600 USD to 3700 USD.
3. This creates a shortfall of 100 USD between the long and short positions, which is now no longer covered by the collateral, resulting in 0 credit.
4. The liquidation process triggers, and a whitelisted liquidator will call the **Native Swap Engine** to request a liquidation with all the necessary information.
5. Once the liquidation is approved and signed, the liquidator can execute the transaction on-chain.
6. Liquidator will repay 1 WETH to settle the short position, while claiming 3600 USDC from the long position, along with 100 USDC from the **PMM** collateral.&#x20;
