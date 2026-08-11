# Yield-Bearing Assets

### Performance & Advantages

* **Optimized Pricing:** Market makers quote based on mint/redeem values and real-time market data, ensuring executable, market-responsive prices.
* **Efficient Settlement:** Native's settlement and margin infrastructure supports market maker inventory and reliable swap execution.
* **Continuous Liquidity:** Market makers periodically rebalance the pool, such as redeeming stETH for ETH or trading on secondary markets, to maintain readiness for future swaps.
* **Easy Access:** Native’s pricing and liquidity are available directly through its Swap Engine API and integrated across major aggregators and gateways.



### Case Study: STONEUSD

STONEUSD is a yield-bearing asset backed by a delta-neutral funding rate arbitrage portfolio. Its rebalancing mechanism enables holders to continuously accumulate returns.

The STONEUSD contract maintains a fair price that governs direct minting and withdrawal. These functions, however, are restricted to allowlisted addresses that have completed KYB/KYC. For most users, swapping is therefore the primary method to acquire or exit the asset, making swap liquidity, pricing, and stability critical.

StakeStone, the issuer of STONEUSD, has partnered with Native to deliver optimal swap execution—offering superior liquidity, responsive pricing, and a seamless experience for both end-users and integration partners.

**Easily Accessible PMM Liquidity**

* The liquidity powered by Native's private market making (PMM) is easily accessible across various interfaces and aggregators, such as 1inch. Projects can also integrate it directly into their own frontends using the Native Swap API.

**Pricing Advantage and Stability**

* Leveraging its PMM pricing model and inventory support, Native provides users with an optimized price for swaps of various sizes, avoiding the price volatility inherent to AMM models. Currently, STONEUSD trades with a spread of less than 3 bps, which is among the best in the market.

<figure><img src="../../.gitbook/assets/image (84).png" alt=""><figcaption></figcaption></figure>

**High Liquidity Utilization**

* With Native's PMM, all provided liquidity is actively utilized for trading at the optimal price point. This eliminates the common AMM issue where a significant portion of liquidity lies dormant at non-optimal price ranges. With an equivalent amount of liquidity, Native can offer significantly deeper tradable liquidity at competitive prices compared to typical AMM pools.

<figure><img src="../../.gitbook/assets/image (85).png" alt=""><figcaption></figcaption></figure>

**Lower Cost for Supplying Liquidity**

* For asset issuers, establishing secondary market liquidity is essential. Native's capital-efficient market infrastructure can lower the cost of supporting secondary liquidity compared with traditional AMM approaches.

<figure><img src="../../.gitbook/assets/Working Collection (19).png" alt=""><figcaption></figcaption></figure>
