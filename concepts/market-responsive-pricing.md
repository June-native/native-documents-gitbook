# Market-Responsive Pricing

**Native Swap Engine** is powered by Market-Responsive Pricing from the private market makers. PMMs acturately and promptly price each token on market risk, conditions and inventory exposures. Due to the nature of this pricing mechanism, it sources liquidity from various onchain DEXs, CEXs and/or direct redemption channels, ultimately responding to the real market.

There are two ways to fetch pricing from **Native Swap Engine**:

#### Orderbook with auto-sign

An orderbook endpoint is provided to return the best pricing for each swap pairs. The levels are representing the amount of available liquidity at different price levels.

Some orderbooks offer additional fields for auto-sign orders, allowing swappers to submit swap orders which will be signed, and executed in a **low latency**, **high success** rate manner.

#### Quote Request

A backend API call is also provided to return a quote, along with a signed, executable calldata for any given swap orders.

If the quote is favourable, simple take the calldata and execute before the expiration timestamp to complete the swap.
