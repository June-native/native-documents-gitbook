# Slippage

The slippage parameter is required for both firm quotes and auto-sign orders.

Slippage is applied in the following scenarios:

1. When external liquidity is aggregated for a swap.
2. When executing auto-sign orders.

If the slippage limit is exceeded during a swap:

* **External liquidity**: The **Native Swap Engine** contract will revert the transaction on-chain, indicating insufficient liquidity.
* **Auto-sign orders**: An error will be returned by the **Native Swap Engine** backend API, indicating that the most recent orderbook levels do not meet the required quote.
