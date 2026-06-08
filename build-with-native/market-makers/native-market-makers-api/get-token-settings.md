---
hidden: true
---

# GET Token settings

{% openapi src="../../../.gitbook/assets/nativeLend.json" path="/v1/lend/token-settings" method="get" %}
[nativeLend.json](../../../.gitbook/assets/nativeLend.json)
{% endopenapi %}

#### Example response

```json
[
    {
        "token_address": "0xa0b86991c6218b36c1d19d4a2e9eb0ce3606eb48",
        "lp_token_address": "0xfc6ced6fe2aa08a9bac61edc3fa2a91a10ac303e",
        "token_group": "stable",
        "token_symbol": "USDC",
        "chain": "ethereum",
        "collateral_factor": 0.8,
        "short_fee_bips": 100,
        "long_fee_bips": 0
    },
    {
        "token_address": "0xdac17f958d2ee523a2206206994597c13d831ec7",
        "lp_token_address": "0x9b705f534fc09212071bad509ba86a8042bdef10",
        "token_group": "stable",
        "token_symbol": "USDT",
        "chain": "ethereum",
        "collateral_factor": 0.8,
        "short_fee_bips": 100,
        "long_fee_bips": 0
    },
    {
        "token_address": "0xc02aaa39b223fe8d0a0e5c4f27ead9083c756cc2",
        "lp_token_address": "0xc41d25382889b1e484e28c4f9bbdbd6b6117e6b2",
        "token_group": "major",
        "token_symbol": "WETH",
        "chain": "ethereum",
        "collateral_factor": 0.8,
        "short_fee_bips": 521.14348553472,
        "long_fee_bips": 267.41221652736
    },
    {
        "token_address": "0x7122985656e38bdc0302db86685bb972b145bd3c",
        "lp_token_address": "0x3960f07204d2cfecbff63534aa8a1309ef937a77",
        "token_group": "alt",
        "token_symbol": "STONE",
        "chain": "ethereum",
        "collateral_factor": 0.8,
        "short_fee_bips": 100,
        "long_fee_bips": 0
    },
    {
        "token_address": "0x2260fac5e5542a773aa44fbcfedf7c193bc2c599",
        "lp_token_address": "0xcf7834b27d60809bae3260a3e24b47814809d68c",
        "token_group": "major",
        "token_symbol": "WBTC",
        "chain": "ethereum",
        "collateral_factor": 0.8,
        "short_fee_bips": 100,
        "long_fee_bips": 0
    },
    {
        "token_address": "0xcd5fe23c85820f7b72d0926fc9b05b43e359b7ee",
        "lp_token_address": "0x92d870bd1fa7f90f74ad724257947e6a8f22217f",
        "token_group": "alt",
        "token_symbol": "weETH",
        "chain": "ethereum",
        "collateral_factor": 0.8,
        "short_fee_bips": 100,
        "long_fee_bips": 0
    }
]
```
