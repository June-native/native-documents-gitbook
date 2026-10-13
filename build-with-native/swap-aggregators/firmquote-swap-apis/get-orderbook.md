# GET Orderbook

{% hint style="info" %}
Native Relay liquidity is highly efficient and often moves rapidly to meet the market mid-price.

Therefore, it is common for one side of the liquidity to be depleted completely. Or, the available liquidity switches from one direction to the opposite.

Whenever a trading direction has insufficient liquidity, the entire orderbook object (unique in base+quote addresses) **will be removed completely** from the endpoint. In such cases, quoting on this particular pair/direction is no longer available.
{% endhint %}

Response notes:

* Each level is `[base_quantity, price]` and is per-level, not cumulative. Prices are net of the fee configured for your API key.
* `side` is always `bid`: each item means selling `base` for `quote`. The opposite direction is a separate item.
* `minimum_in_base` is expressed in wei of the base token. Quotes below it fail with `171055`.
* Empty or expired books, and pairs not enabled for your API key, are omitted.
* Optional params: `chain` (name) or `chainId` (numeric), and `showNative` (see Native Tokens Support).

{% openapi src="../../../.gitbook/assets/Naive_API_swagger.json" path="/v1/orderbook" method="get" %}
[Naive_API_swagger.json](../../../.gitbook/assets/Naive_API_swagger.json)
{% endopenapi %}
