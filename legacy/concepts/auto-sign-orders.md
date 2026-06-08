# Auto Sign Orders

Native V2 introduces **auto sign orders**, providing lower latency and higher success rates for swaps.

### **The Swap Process**

<figure><img src="../../.gitbook/assets/Auto Sign.png" alt=""><figcaption></figcaption></figure>

1. Swappers query the `/orderbook` endpoint to gather pricing and depth information.
2. Swappers submit the swap order to the auto-sign endpoint with parameters such as the amount, token addresses, and the required `permit2` signature for the Native Swap Engine.
3. The Native Executor Network verifies, signs, and executes the swap order on-chain, ensuring enhanced speed and success rates.
4. Swappers can monitor the order status and retrieve the final on-chain transaction ID via API endpoints.

### **Key Considerations**

* Auto-sign orders benefit from significantly lower latency and a higher success rate due to improved inventory and swap state control.
* Slippage is automatically implemented to increase success rates. If the liquidity at the most recent orderbook levels becomes unavailable, the Native Swap Engine will use the next available levels, provided the execution price remains within the defined slippage parameters.
* Gas fees are included in the quote and charged in the input token. For example, swapping USDT to ETH will result in gas fees being deducted in USDT.
