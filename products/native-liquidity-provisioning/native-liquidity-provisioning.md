# Yield-Bearing Assets

### Background

Yield-bearing assets offer diversified returns (DeFi, RWA, CeFi), yet exit liquidity remains constrained. Direct withdrawals often require whitelisting and mandatory waiting queues lasting days, while AMM liquidity is expensive and vulnerable to slippage and often incorrect pricing.

The key distinction between yield-bearing assets and other generic assets lies in their core pricing logic. A yield-bearing asset often has implied pricing based on underlying yield-generating assets, which is often the long-term fair market price. AMMs are by nature not suitable for pegging to this implied fair pricing.

Native solves this by combining a private market-making engine with credit pool liquidity, delivering instant liquidity and superior implied fair pricing. The result is a smoother user experience, higher capital efficiency, and lower liquidity costs for issuers.

With Native, liquidity moves efficiently, with better prices and lower cost.

### How Native Works

Native delivers instant liquidity at optimized prices for any supported asset by integrating its Swap Engine (API), Credit Pool, and Market Makers—significantly lowering the cost and operational overhead of liquidity provision.

For users, swapping is straightforward:

1. Submit a swap intent (e.g., 1 stETH → ETH) via the Native Swap Engine API or a supported aggregator.
2. Receive an executable, optimized price.
3. Confirm. The swap settles instantly, like any standard token transfer.

### Performance & Advantages

* **Optimized Pricing:** Market makers quote based on mint/redeem values and real-time market data, ensuring executable, market-responsive prices.
* **Instant Settlement:** Liquidity for swaps is drawn from the Native Credit Pool, which acts as the market maker’s inventory. LPs supply single-sided assets permissionlessly and earn yields.
* **Continuous Liquidity:** Market makers periodically rebalance the pool, such as redeeming stETH for ETH or trading on secondary markets, to maintain readiness for future swaps.
* **Easy Access:** Native’s pricing and liquidity are available directly through its Swap Engine API and integrated across major aggregators and gateways.

<figure><img src="../../../.gitbook/assets/Case Study - System - 1218.png" alt=""><figcaption></figcaption></figure>

### Case Study: STONEUSD

STONEUSD is a yield-bearing asset backed by a delta-neutral funding rate arbitrage portfolio. Its rebalancing mechanism enables holders to continuously accumulate returns.

The STONEUSD contract maintains a fair price that governs direct minting and withdrawal. These functions, however, are restricted to allowlisted addresses that have completed KYB/KYC. For most users, swapping is therefore the primary method to acquire or exit the asset, making swap liquidity, pricing, and stability critical.

StakeStone, the issuer of STONEUSD, has partnered with Native to deliver optimal swap execution—offering superior liquidity, responsive pricing, and a seamless experience for both end-users and integration partners.



**Easily Accessible PMM Liquidity**

* The liquidity powered by Native's private market making (PMM) is easily accessible across various interfaces and aggregators, such as 1inch. Projects can also integrate it directly into their own frontends using the Native Swap API.



**Pricing Advantage and Stability**

* Leveraging its PMM pricing model and inventory support, Native provides users with an optimized price for swaps of various sizes, avoiding the price volatility inherent to AMM models. Currently, STONEUSD trades with a spread of less than 3 bps, which is among the best in the market.

<figure><img src="../../../.gitbook/assets/image (84).png" alt=""><figcaption></figcaption></figure>



**High Liquidity Utilization**

* With Native's PMM, all provided liquidity is actively utilized for trading at the optimal price point. This eliminates the common AMM issue where a significant portion of liquidity lies dormant at non-optimal price ranges. With an equivalent amount of liquidity, Native can offer significantly deeper tradable liquidity at competitive prices compared to typical AMM pools.

<figure><img src="../../../.gitbook/assets/image (85).png" alt=""><figcaption></figcaption></figure>



**Lower Cost for Supplying Liquidity**

* For asset issuers, establishing secondary market liquidity is essential. Native reduces this cost, as liquid assets supplied to its credit pool earn rewards at the same time. In contrast, AMM pools for yield-bearing assets often generate lower returns due to less prominent trading activity, making liquidity provision comparatively more expensive.
* Check [Yield dashboard](https://defillama.com/yields?project=Native+Credit+Pool\&chain=Ethereum) on Native

<figure><img src="../../../.gitbook/assets/Working Collection (19).png" alt=""><figcaption></figcaption></figure>
