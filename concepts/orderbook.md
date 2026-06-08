# Orderbook

The Orderbook provides real-time pricing and detailed data from private market makers, aggregated through the Native Swap Engine.

Swappers can access the latest information by querying the orderbook endpoint and analyzing the returned data, which includes price levels and token quantities.

**Example Response**:

```json
[
    {
        "base_symbol": "WETH",
        "quote_symbol": "USDT",
        "base_address": "0xC02aaA39b223FE8D0A0e5C4F27eAD9083C756Cc2",
        "quote_address": "0xdac17f958d2ee523a2206206994597c13d831ec7",
        "minimum_in_base": 0,
        "side": "bid",
        "levels": [
            [
                0.0001,
                3213.12345
            ],
            [
                12.75786733219471,
                3210.15
            ],
            [
                42,
                3210.12
            ]
        ]
    }
}
```

For further details about the orderbook endpoint, refer to the following link: [Swap API](../build-with-native/swap-aggregators/firmquote-swap-apis/), [PMM API](../build-with-native/market-makers/native-market-makers-api/)

**Note**: The price levels provided are non-cumulative. For example, to execute a trade of 30 ETH for USDT, the price would be calculated as:\
`(0.0001 * 3213.12345 + 12.75786733219471 * 3210.15 + (30 - 0.0001 - 12.75786733219471) * 3210.12) / 30 = 3,210.13276787883`
