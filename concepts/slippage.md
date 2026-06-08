# Slippage

Slippage is the difference between the expected price of a swap and the price at which it actually executes. A slippage parameter sets the maximum tolerated difference before a swap is rejected.

For swaps routed through [Native Relay](../modules/native-relay.md), slippage is applied in the following scenarios:

1. When external liquidity is aggregated for a swap.
2. When executing against streamed order-book levels.

If the slippage limit is exceeded during a swap:

* **External liquidity**: the on-chain swap contract reverts the transaction, indicating insufficient liquidity.
* **Order-book execution**: an error is returned by the backend API, indicating that the most recent order-book levels do not meet the required quote.
