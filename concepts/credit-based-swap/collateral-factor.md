# Collateral Factor

It is important to note that credits are in USD, computed based on the number of balances and USD prices. Additionally, each asset will be assigned a collateral factor, which is also used in this calculation.

Each collateral asset is associated with a distinct collateral factor, which indicates the proportion of USD value that collateral or long positions contribute to the credit. In the example above, this factor is set at 100% for simplicity.

In actuality, this factor is generally lower than 100% and is subject to adjustments based on:

*   USD pegged assets, for example: USDT, USDC

    Highest collateral factor
*   Large cap, blue chip assets, for example: WBTC, WETH

    Medium to high collateral factor
*   Long-tail assets

    Low to no collateral factor

For instance, if USDT possesses a collateral factor of 90%, WBTC has a collateral factor of 80%, TOKEN1 is at 5%, and TOKEN2 stands at 0%, then a long position of each token will contribute 90 USD, 80 USD, 5 USD, and 0 USD to the credit, respectively, based on their corresponding collateral factors.
