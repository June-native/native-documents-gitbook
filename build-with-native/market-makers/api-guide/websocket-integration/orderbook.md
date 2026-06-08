# Orderbook

To indicate current pricing for Native APIs and aggregators, you must send price levels for each supported pair on every network. The recommended frequency is **every second for each pair on every supported network**.

Below is the message format expected by Native:

```
{ 
  "messageType": "orderbook",
  "message": {
    "chainId": number,  // e.g. 1 (mainnet), 137 (polygon)
    "baseTokenAddress": string,  // e.g. 0xc02aaa39b223fe8d0a0e5c4f27ead9083c756cc2 (WETH)
    "quoteTokenAddress": string,  // e.g. 0xdac17f958d2ee523a2206206994597c13d831ec7 (USDT)
    "side": "buy" | "sell",
    "levels": [
      {
        // string representation of the quantity of current level e.g. (2.5 for 2.5 WETH)
        quantity: string,

        // string representation of the price per unit at that level 
        // (e.g. 3500 for "up to 2.5 WETH at 3500 USDT per WETH")
        price: string
      }
    ]
  }
}
```

{% hint style="info" %}
**Note**: The price levels must be returned in a non-cumulative manner.
{% endhint %}

For example, given the following price levels for the `WETH-USDT` pair:

```
[
   { quantity: "0.001", price: "1600" }, // The first 0.001 ETH is priced at 1600 USDT / WETH
   { quantity: "1", price: "1610" }, // The next 1 ETH is priced at 1610 USDT / WETH
   { quantity: "2", price: "1612" } // The next 2 ETH is priced at 1612 USDT / WETH
]
```

If someone wants to trade 3 WETH for USDT, the price calculation would be: `0.001 * 1600 + 1 * 1610 + 1.999 * 1612 = 4833.988`

{% hint style="info" %}
**Note:**

1. **Price caching:** Native will cache these price levels for a maximum of 5 seconds (or until new levels are published). Prices will be considered expired and invalid if you do not post new price levels within 5 seconds.
2. **Minimum order quantity:** The first level always represents the minimum amount you're looking to buy. If you want to support arbitrarily small amounts, set the first level quantity to "0".
{% endhint %}
