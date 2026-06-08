# GET Orderbook

{% hint style="info" %}
Native Relay liquidity is highly efficient and often moves rapidly to meet the market mid-price.

Therefore, it is common for one side of the liquidity to be depleted completely. Or, the available liquidity switches from one direction to the opposite.

Whenever a trading direction has insufficient liquidity, the entire orderbook object (unique in base+quote addresses) **will be removed completely** from the endpoint. In such cases, quoting on this particular pair/direction is no longer available.
{% endhint %}

{% openapi src="../../../.gitbook/assets/Naive_API_swagger.json" path="/v1/orderbook" method="get" %}
[Naive_API_swagger.json](../../../.gitbook/assets/Naive_API_swagger.json)
{% endopenapi %}
