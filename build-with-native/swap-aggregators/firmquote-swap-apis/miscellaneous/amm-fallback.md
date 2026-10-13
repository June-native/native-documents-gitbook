# AMM Fallback

{% hint style="info" %}
The Native Router `externalSwap` calldata is only generated with `version=4`. For other versions the response `txRequest` contains the external AMM transaction directly, with `function` empty and `source` of `[0]`.
{% endhint %}

Native Core liquidity provides remarkable depth and competitiveness for major assets like WETH and WBTC.&#x20;

However, to use the Native Relay API in wallets or swap widgets, Native offers an AMM fallback option to support trading of nearly any tokens.

To enable this feature for your API key, contact Native team members through an existing channel.

### Quote and Swap

To use this feature:

* Make sure your API key is whitelisted for AMM fallback.
* Use `version=4` when calling `/firm-quote`. Note that version 4 will be deprecated soon; use `version=6` for everything else.

`/indicative-quote` and `/firm-quote` will automatically utilise AMM liquidity if the quoted pair is not available from Native's own liquidity. No adjustment is needed to the request.

Fallback is only attempted when all of the following hold:

* your API key is whitelisted for AMM fallback;
* `from_address` equals `to_address`; and
* `src_chain` equals `dst_chain`.

It is triggered when there is no pair (`171011`), the orderbook is empty or unavailable, the amount exceeds the available depth (`101010`), or the token is not found (`171015`).

{% hint style="info" %}
On the fallback path the widget fee is zero and the `fee` parameter is ignored. `is_black_address` is not populated.
{% endhint %}

{% hint style="warning" %}
Please note that in liquidity fallback, the contract function that gets called is different. Normally, it should be `tradeRFQT`, but in AMM fallback, it will be `externalSwap`.

To track which function the swap tx is calling, check `txRequest.function`.
{% endhint %}

### Get Tradable Token

AMM fallback enables trading for thousands of tokens; therefore, it is impractical to show individual trading pairs in the `/orderbook` endpoint. Instead, we offer a new endpoint to keep track of the tradable tokens.

Call `/widget-tokens?chains={chainName}` (comma-separate multiple chains, or use `chain=` for one) with your API key. The response is an object keyed by chain name. Unknown chain names are silently ignored.

* `isSupported: true` means the token is available from Native core liquidity; `false` means it is only available through AMM fallback.
* `isStablecoin`, `isFeatured` and `desc` are not reliable and should not be used.

Example:

```bash
## Get All Supported Tokens on Ethereum Including AMM Fallback
curl "https://v2.api.native.org/swap-api-v2/v1/widget-tokens?chains=ethereum" \
     -H "apiKey: $YOUR_KEY_HERE"
```

```json
{
  "ethereum": [
    {
      "name": "World Liberty Financial",
      "desc": "",
      "logo": "https://s2.coinmarketcap.com/static/img/coins/64x64/33251.png",
      "chain": "ethereum",
      "symbol": "WLFI",
      "address": "0xdA5e1988097297dCdc1f90D4dFE7909e847CBeF6",
      "decimals": 18,
      "isFeatured": false,
      "isStablecoin": true,
      "isSupported": true
    },
    {
      "name": "Skate",
      "desc": "",
      "logo": "https://static.native.org/logo/tokens/1_0x61DBbBb552dc893ab3aAd09F289f811E67cEf285.jpg",
      "chain": "ethereum",
      "symbol": "SKATE",
      "address": "0x61DBbBb552dc893ab3aAd09F289f811E67cEf285",
      "decimals": 18,
      "isFeatured": false,
      "isStablecoin": true,
      "isSupported": true
    },
    // ...
  ]
}
```
