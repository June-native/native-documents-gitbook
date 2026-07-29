---
hidden: true
---

# GET Market maker positions

{% openapi src="../../../.gitbook/assets/nativeLend.json" path="/v1/lend/mm-positions" method="get" %}
[nativeLend.json](../../../.gitbook/assets/nativeLend.json)
{% endopenapi %}

#### Example response

```json
{
    "availableCreditUsd": 9148.571380914098, // sum of credit USD value of all the positions (long USD * collateral factor - short USD). If this amount < 0, this account would be open for liquidation
    "unrealizedPnlUsd": 805.7590529398276, // sum of USD values of all the positions, without considering the collateral factor
    "creditItems": [
        {
            "tokenAddress": "0x35751007a407ca6feffe80b3cb397736d2cf4dbe",
            "tokenSymbol": "weETH",
            "mmAddress": "0x04f57b77cd9102a3774570de93008e9c57b9f146",
            "chain": "arbitrum",
            "tokenUsdPrice": 3597.532145405,
            "collateralFactor": 0.8,
            "amount": "12120017959845",
            "amountEther": 0.000012120017959845,
            "amountUsd": 0.043602154213428314,
            "pendingFee": "0",
            "pendingFeeEther": 0,
            "pendingFeeUsd": 0,
            "creditUsd": 0.034881723370742655 //(amountUsd + pendingFeeUsd) * collateralFactor (for long)
        },
        {
            "tokenAddress": "0x82af49447d8a07e3bd95bd0d56f35241523fbab1",
            "tokenSymbol": "WETH",
            "mmAddress": "0x04f57b77cd9102a3774570de93008e9c57b9f146",
            "chain": "arbitrum",
            "tokenUsdPrice": 3457.13070031436,
            "collateralFactor": 0.8,
            "amount": "-6537447221630394057",
            "amountEther": -6.537447221630394,
            "amountUsd": -22600.80949158325,
            "pendingFee": "-2104388931527",
            "pendingFeeEther": -0.000002104388931527,
            "pendingFeeUsd": -0.007275147580583726,
            "creditUsd": -22600.816766730833
        },
        {
            "tokenAddress": "0x912ce59144191c1204e64559fe8253a0e49e6548",
            "tokenSymbol": "ARB",
            "mmAddress": "0x04f57b77cd9102a3774570de93008e9c57b9f146",
            "chain": "arbitrum",
            "tokenUsdPrice": 0.8268962142857142,
            "collateralFactor": 0.8,
            "amount": "-3882167728383781221594",
            "amountEther": -3882.167728383781,
            "amountUsd": -3210.14979782272,
            "pendingFee": "-1249660688794880",
            "pendingFeeEther": -0.00124966068879488,
            "pendingFeeUsd": -0.0010333396927061642,
            "creditUsd": -3210.1508311624125
        },
        {
            "tokenAddress": "0xaf88d065e77c8cc2239327c5edb3a432268e5831",
            "tokenSymbol": "USDC",
            "mmAddress": "0x04f57b77cd9102a3774570de93008e9c57b9f146",
            "chain": "arbitrum",
            "tokenUsdPrice": 1.000731297142857,
            "collateralFactor": 0.8,
            "amount": "-23135662312",
            "amountEther": -23135.662312,
            "amountUsd": -23152.581355746872,
            "pendingFee": "-461848",
            "pendingFeeEther": -0.461848,
            "pendingFeeUsd": -0.4621857481228342,
            "creditUsd": -23153.043541494993
        },
        {
            "tokenAddress": "0xfd086bc7cd5c481dcc9c85ebe478a1c0b69fcbb9",
            "tokenSymbol": "USDT",
            "mmAddress": "0x04f57b77cd9102a3774570de93008e9c57b9f146",
            "chain": "arbitrum",
            "tokenUsdPrice": 0.99905,
            "collateralFactor": 0.8,
            "amount": "43265079293",
            "amountEther": 43265.079293,
            "amountUsd": 43223.977467671655,
            "pendingFee": "0",
            "pendingFeeEther": 0,
            "pendingFeeUsd": 0,
            "creditUsd": 34579.181974137326
        },
        {
            "tokenAddress": "0x2f2a2543b76a4166549f7aab2e75bef0aefc5b0f",
            "tokenSymbol": "WBTC",
            "mmAddress": "0x04f57b77cd9102a3774570de93008e9c57b9f146",
            "chain": "arbitrum",
            "tokenUsdPrice": 61897.832810023065,
            "collateralFactor": 0.8,
            "amount": "10574326",
            "amountEther": 0.10574326,
            "amountUsd": 6545.2786282668,
            "pendingFee": "0",
            "pendingFeeEther": 0,
            "pendingFeeUsd": 0,
            "creditUsd": 5236.222902613441
        }
    ],
    "collateralItems": [],
    "pendingRequest": {},
    "chain": "arbitrum",
    "timestamp": "2024-06-28T04:00:02.000Z",
    "mmAddress": "0x04f57b77cd9102a3774570de93008e9c57b9f146",
    "isPaused": false
}
```
