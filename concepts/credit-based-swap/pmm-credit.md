# PMM Credit

### What is PMM credit?

Credit is required when private market makers facilitate credit-based swaps within the **Native Swap Engine**. For each token swap, a Private Market Maker (**PMM**) is effectively longing the input token that the user provides while simultaneously shorting the output token that is borrowed from **Native Credit Pool** and subsequently distributed to users.

Consequently, in this process, credit is required to bridge the gap between the long and short positions. For instance, if a PMM is longing 3,000 USDC while shorting 1 WETH, with the price of WETH at 3600 USD, the PMM must maintain an additional 600 USD in collateral to ensure that the credit remains positive.

### How to calculate credit?

The available credits for the PMM will be computed using:

`Credit = longPositionUsd - shortPositionUsd + collateralUsd`

**Note:** `longPositionUsd` and `collateralUsd` are subject to collateral factors which varies between tokens. More details [here](collateral-factor.md).

### Collaterals

**PMM**s contribute liquidity in the form of base tokens to the Native credit pools, such as USDC, USDT, WETH, and WBTC. Upon the locking the Liquidity Provider (**LP**) tokens, these liquid asset deposits are transformed into collateral which subsequently contributes to the total available credits.

For instance, a **PMM** may deposit and secure 1 USDT in the Credit Pools, with the collateral factor of 90% resulting in 0.9 USD being added to the pool of available credits.

### Long Positions

Furthermore, assets held in long positions also contribute credits.

For example, should a **PMM** maintain a long position of 1 USDC, with the collateral factor being 80%, then 0.8 USD would similarly be added to the available credits.
