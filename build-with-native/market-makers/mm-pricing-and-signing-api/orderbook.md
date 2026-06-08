---
description: Price levels endpoint for Native APIs and aggregators
---

# Orderbook

To indicate current pricing for Native APIs and aggregators, you must provide a price level endpoint that returns the price levels of all supported pairs on a specified chain.

Native will call this endpoint **every second on every supported network** to retrieve the latest price levels.

The endpoint must adhere to the following format:

### Endpoint <a href="#orderbook-endpoint" id="orderbook-endpoint"></a>

```
GET <market_maker_base_url>/orderbook
```

### **Params** <a href="#params" id="params"></a>

<table><thead><tr><th width="151">Name</th><th>Description</th></tr></thead><tbody><tr><td><code>chainId</code></td><td>Chain ID of the blockchain network, in integers.<br><br>For example: 1 for Ethereum, 56 for BSC.</td></tr></tbody></table>

#### **Example**

```
<market_maker_base_url>/orderbook?chainId=1
```

### Response <a href="#response" id="response"></a>

You need to provide a response in the following format:

**Type:** Array of order pairs

<table><thead><tr><th width="241">Name</th><th>Description</th></tr></thead><tbody><tr><td><code>base_symbol</code></td><td>The symbol of the token you are buying or selling</td></tr><tr><td><code>base_address</code></td><td>The address of the token you are buying or selling</td></tr><tr><td><code>quote_symbol</code></td><td>The symbol of the token that will exchanged for the base token</td></tr><tr><td><code>quote_address</code></td><td>The address of the token that will exchanged for the base token</td></tr><tr><td><code>levels</code></td><td>This is an array of tuples in the format of [[int, int]]. The value at index 0 is the amount of tokens and the value at index 1 is the price you are willing to transact at.</td></tr><tr><td><code>side</code></td><td>"Ask" or "Bid"<br><br>Indicates whether you are buying or selling the base token</td></tr><tr><td><code>minimum_in_base</code></td><td>The minimum amount of base tokens that you are willing to transact</td></tr><tr><td><code>max_expiry</code></td><td>(optional) The maximum acceptable expiry time in seconds. Native API will never request a quote with a deadline that exceeds this number. Your liquidity will simply be skipped for this order.</td></tr></tbody></table>

#### **Example**

```
[
    {
        "base_symbol": "USDT",
        "base_address": "0xdac17f958d2ee523a2206206994597c13d831ec7",
        "quote_symbol": "AAVE",
        "quote_address": "0x7fc66500c84a76ad7e9c93437bfc5ac33e2ddae9",
        "levels": [
            [
                278.6807021305324,
                0.016506345666681244
            ],
            [
                2883.2639932781804,
                0.0165090675397643
            ],
            // ... more price levels
        ],
        "side": "ask",
        "minimum_in_base": 0.0001,
        "max_expiry": 60,
    },
    {
        "base_symbol": "AAVE",
        "base_address": "0x7fc66500c84a76ad7e9c93437bfc5ac33e2ddae9",
        "quote_symbol": "USDT",
        "quote_address": "0xdac17f958d2ee523a2206206994597c13d831ec7",
        "levels": [
            [
                4.6,
                60.58276133272444
            ],
            [
                47.6,
                60.572772968029
            ],
            // ... more price levels
        ],
        "side": "bid",
        "minimum_in_base": 0.0001
    }
    // ... more token pairs
]
```

{% hint style="info" %}
**Note**: The price levels must be returned in a non-cumulative manner.
{% endhint %}

For example, given the following price levels for the `WETH-USDT` pair:

```
[
    [0.001, 1600],
    [1,1610],
    [2,1612]
]
```

If someone wants to trade 3 WETH for USDT, the price would be calculated as follows: `0.001 * 1600 + 1 * 1610 + 1.999 * 1612 = 4833.988`

{% hint style="info" %}
The Native server is hosted in the AWS Tokyo region, and all market makers are required to respond to all endpoints in 100ms (end-to-end).
{% endhint %}
