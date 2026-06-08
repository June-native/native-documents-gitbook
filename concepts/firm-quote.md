# Firm Quote

A **firm quote** is a committed, executable price returned with signed calldata for on-chain execution. It is the basis of Native Relay's [RFQ integration mode](integration-modes.md): an integrator requests a quote via the `/firm-quote` endpoint and executes the returned calldata on-chain.

For further details, refer to the following link:

{% content-ref url="../build-with-native/swap-aggregators/firmquote-swap-apis/" %}
[firmquote-swap-apis](../build-with-native/swap-aggregators/firmquote-swap-apis/)
{% endcontent-ref %}

### **The Swap Process**

<figure><img src="../.gitbook/assets/Native RFQ.png" alt=""><figcaption></figcaption></figure>

1. Swappers query the `/orderbook` endpoint to obtain pricing and depth information and generate an initial quote.
2. Swappers submit the swap parameters, including the amount and token addresses, via the `/firm-quote` endpoint to receive a firm quote and executable calldata.
3. The calldata is executed on the blockchain to finalize the swap.

### **Key Considerations**

* Utilize the `/orderbook` or `/indicative-quote` endpoints for initial pricing estimates, typically displayed on the primary swap interface. The `/firm-quote` endpoint should be used for finalizing quotes before on-chain transactions.
* The `/firm-quote` endpoint may experience rate limits and slightly higher latency due to inventory control, additional checks, and signing processes.
* Specify the "expiry" parameter when requesting a firm quote. Note that longer expiries may reduce PMM liquidity, potentially resulting in less favorable quotes.
* Native aggregates liquidity from on-chain AMM DEX sources. Consequently, final signed orders may include swaps involving external liquidity. However, Swap API keys can be configured to exclusively utilize PMM quotes.
