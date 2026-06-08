# Multihop

Native Relay API and Router support multihop between all trading pairs in `/orderbook` return, making swapping with Native liquidity more versatile.

To enable, add `allow_multihop=true` when requesting `/indicative-quote` or `/firm-quote`.

{% hint style="info" %}
Please note that `/orderbook` only return base trading pairs and does not return any virtual pairs enabled by multihop. To calculate price and depth, please combine two orderbooks locally.
{% endhint %}

### Example

Example orderbooks:

```json
[
   // WBTC->USDT
  {
    "base_symbol": "WBTC",
    "quote_symbol": "USDT",
    "base_address": "0x2260fac5e5542a773aa44fbcfedf7c193bc2c599",
    "quote_address": "0xdac17f958d2ee523a2206206994597c13d831ec7",
    "minimum_in_base": 0,
    "side": "bid",
    "levels": [
      [
        0.5,
        122739.77003987515
      ],
      [
        1,
        122728.46555507004
      ],
      [
        1,
        122713.39290866326
      ],
      [
        0.6261126319999994,
        122701.13799830334
      ]
    ]
  },
  // USDT->USDC
  {
    "base_symbol": "USDT",
    "quote_symbol": "USDC",
    "base_address": "0xdac17f958d2ee523a2206206994597c13d831ec7",
    "quote_address": "0xa0b86991c6218b36c1d19d4a2e9eb0ce3606eb48",
    "minimum_in_base": 0,
    "side": "bid",
    "levels": [
      [
        295958.17538750934,
        1.0003537268714557
      ]
    ]
  },
  //...
```

```bash
## Get Indicative Quote for WBTC->USDC
curl "https://v2.api.native.org/swap-api-v2/v1/indicative-quote?from_address=0xbf381E1cBfdb0D02F3800010e490130D3dC73118&src_chain=ethereum&dst_chain=ethereum&token_in=0x2260fac5e5542a773aa44fbcfedf7c193bc2c599&token_out=0xa0b86991c6218b36c1d19d4a2e9eb0ce3606eb48&slippage=0.5&amount=0.00001&allow_multihop=true" \
     -H "apiKey: $YOUR_KEY_HERE"
```

```bash
## Get Firm Quote for WBTC->USDC
curl "https://v2.api.native.org/swap-api-v2/v1/firm-quote?from_address=0xbf381E1cBfdb0D02F3800010e490130D3dC73118&amount=0.001&token_in=0x2260fac5e5542a773aa44fbcfedf7c193bc2c599&token_out=0xa0b86991c6218b36c1d19d4a2e9eb0ce3606eb48&version=3&src_chain=ethereum&dst_chain=ethereum&allow_multihop=true" \
     -H "apiKey: $YOUR_KEY_HERE"
```

Tx: [https://www.tdly.co/shared/simulation/52280a28-72a7-41fb-a741-896101281cac](https://www.tdly.co/shared/simulation/52280a28-72a7-41fb-a741-896101281cac)

{% hint style="warning" %}
Please note that in multihop. The contract function that gets called is different. Normally, it should be `tradeRFQT`, but in multihop, it will be `multicall`.

To track which function the swap tx is calling. Use version 4 or above and check `txRequest.function`.
{% endhint %}

{% hint style="info" %}
Multihop currently support only major tokens being the intermediate. Limited to:

USDT, USDC, WBNB, WETH, WBTC
{% endhint %}
