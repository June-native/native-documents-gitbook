---
hidden: true
---

# GET Available borrows

{% openapi src="../../../.gitbook/assets/nativeLend.json" path="/v1/lend/mm-available-borrow" method="get" %}
[nativeLend.json](../../../.gitbook/assets/nativeLend.json)
{% endopenapi %}

Market makers can use this data to learn the max size they can quote to construct the orderbook. `inventoryBalance` indicates the actual balance available in the Vault. `baseShortLimit` is the conservative amount that MM can quote as short token assuming long token has collateral factor `0`. If the long token has a higher collateral factor, the limit would be higher; the adjusted amount is `baseShortLimit/(1 - longTokenCollateralFactor)`. For a more specific example, refer [here](/broken/pages/yizUTrKNKwplKBFC0jMB#how-to-know-the-largest-order-size-that-can-be-quoted).

The actual limit would be `Math.min(inventoryBalance, baseShortLimit/(1 - longTokenCollateralFactor)`)

#### Example response

```json
{
    "0xa0b86991c6218b36c1d19d4a2e9eb0ce3606eb48": {
        "tokenSymbol": "USDC",
        // inventoryBalance: The inventory balance in the Lend Vault
        "inventoryBalanceWei": "201896139137", 
        "inventoryBalanceEther": 201896.139137,
        // baseShortLimit: The conservative amount that MM can quote as short token, assuming long token has collateral factor 0
        "baseShortLimitWei": "9991965330", 
        "baseShortLimitEther": 9991.965330,
        // do not use the collateral for this token itself on the short side, should use the collateral factor of the long token in the pair
        "collateralFactor": 0.8,
        "tokenUsd": 1.0008041129999998
    },
    "0xdac17f958d2ee523a2206206994597c13d831ec7": {
        "tokenSymbol": "USDT",
        "inventoryBalanceWei": "40592895009",
        "inventoryBalanceEther": 40592.895009,
        "baseShortLimitWei": "10012332740",
        "baseShortLimitEther": 10012.332740,
        "collateralFactor": 0.8,
        "tokenUsd": 0.9987682450860701
    },
    "0xc02aaa39b223fe8d0a0e5c4f27ead9083c756cc2": {
        "tokenSymbol": "WETH",
        "inventoryBalanceWei": "6123000233052323400",
        "inventoryBalanceEther": 6.123000233052323,
        "baseShortLimitWei": "2914794044523376000",
        "baseShortLimitEther": 2.914794044523376,
        "collateralFactor": 0.8,
        "tokenUsd": 3430.7741292353257
    }
}
```
