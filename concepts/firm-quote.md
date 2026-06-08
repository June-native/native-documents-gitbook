# Firm Quote

A **firm quote** is a committed, executable price returned with signed calldata for on-chain execution. It is the basis of Native Relay's [RFQ integration mode](integration-modes.md): an integrator requests a quote via the `/firm-quote` endpoint and executes the returned calldata on-chain.

For further details, refer to the following link:

{% content-ref url="../build-with-native/swap-aggregators/firmquote-swap-apis/" %}
[firmquote-swap-apis](../build-with-native/swap-aggregators/firmquote-swap-apis/)
{% endcontent-ref %}

#### **The Swap Process**

1. The integrator query the `/orderbook` endpoint to obtain pricing and depth information and generate an initial quote.
2. The integrator submit the swap parameters, including the amount and token addresses, via the `/firm-quote` endpoint to receive a firm quote and executable calldata.
3. The calldata is executed on the blockchain to finalize the swap.
