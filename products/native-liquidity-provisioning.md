# Native Liquidity Provisioning

### Background

Yield-bearing assets offer diversified returns (DeFi, RWA, CeFi), yet exits liquidity remain constrained. Direct withdrawals often require whitelisting, mandatory waiting queue in days, while AMM liquidity is expensive and vulnerable to slippage and often incorrect pricing.

The key distinction between yield-bearing assets and other generic assets lies in its core pricing logic. Yield-bearing asset often has implied pricing based on underlying yield-generating assets, and often is the long term fair market pricing. AMMs are in nature not suitable to peg to this implied fair pricing.

Native solves this with private market-making engine and credit pool liquidity combined, delivering instant liquidity and superior implied fair pricing. The result is a smoother user experience, higher capital efficiency, and lower liquidity costs for issuers.

With Native, liquidity moves efficiently, with better prices and lower cost.

### How Native Works

Native delivers instant liquidity at optimized prices for any supported asset by integrating its Swap Engine (API), Credit Pool, and Market Makers—significantly lowering the cost and operational overhead of liquidity provision.

For users, swapping is straightforward:

1. Submit a swap intent (e.g., 1 stETH → ETH) via the Native Swap Engine API or a supported aggregator.
2. Receive an executable, optimized price.
3. Confirm. The swap settles instantly, like any standard token transfer.

#### Behind the Scenes

* Optimized Pricing: Market makers quote based on mint/redeem values and real-time market data, ensuring executable, market-responsive prices.
* Instant Settlement: Liquidity for swaps is drawn from the Native Credit Pool, which acts as the market maker’s inventory. LPs supply single-sided assets permissionlessly and earn yields.
* Continuous Liquidity: Market makers periodically rebalance the pool, such as redeeming stETH for ETH or trading on secondary markets, to maintain readiness for future swaps.
* Easy Access:Native’s pricing and liquidity are available directly through its Swap Engine API and integrated across major aggregators and gateways.

<figure><img src="../.gitbook/assets/Case Study - System - 1218.png" alt=""><figcaption></figcaption></figure>

### STONEUSD: A Case Study

STONEUSD is a yield-bearing asset backed by a delta-neutral funding rate arbitrage portfolio. Its rebalancing mechanism enables holders to continuously accumulate returns.

The STONEUSD contract maintains a fair price that governs direct minting and withdrawal. These functions, however, are restricted to allowlisted addresses that have completed KYB/KYC. For most users, swapping is therefore the primary method to acquire or exit the asset, making swap liquidity, pricing, and stability critical.

StakeStone, the issuer of STONEUSD, has partnered with Native to deliver optimal swap execution—offering superior liquidity, responsive pricing, and a seamless experience for both end-users and integration partners.

STONEUSD (ETH L1): 0x6A6E3a4396993A4eC98a6f4A654Cc0819538721e

STONEUSD (BNB Chain): 0x8B4E28607bdcacBf937f81F29E3DAFe7Bc1D7c0b

#### Steps of The Integration

* List the STONEUSD trading pair on Native (on both Ethereum and BNB Chain)

<figure><img src="../.gitbook/assets/Working Collection (13).png" alt=""><figcaption></figcaption></figure>

* Instead of bootstrapping AMM DEX liquidity, StakeStone supplies liquidity in stablecoins and ETH into the Native credit pool on Ethereum L1 and BNB Chain, serving as the initial market maker inventory.

<figure><img src="../.gitbook/assets/Working Collection (14).png" alt=""><figcaption></figcaption></figure>

#### Performance and advantage

**Easily Accessible PMM Liquidity**

* The liquidity powered by Native's private market making (PMM) is easily accessible across various interfaces and aggregators, such as 1inch. Projects can also integrate it directly into their own frontends using the Native Swap API.

<figure><img src="../.gitbook/assets/Working Collection (15).png" alt=""><figcaption></figcaption></figure>

**Pricing Advantage and Stability**

* Leveraging its PMM pricing model and inventory support, Native provides users with an optimized price for swaps of various sizes, avoiding the price volatility inherent to AMM models. Currently, STONEUSD trades with a spread of less than 3 bps, which is among the best in the market.

<figure><img src="../.gitbook/assets/Working Collection (16).png" alt=""><figcaption></figcaption></figure>

<figure><img src="../.gitbook/assets/Working Collection (17).png" alt=""><figcaption></figcaption></figure>

**High Liquidity Utilization**

* With Native's PMM, all provided liquidity is actively utilized for trading at the optimal price point. This eliminates the common AMM issue where a significant portion of liquidity lies dormant at non-optimal price ranges. With an equivalent amount of liquidity, Native can offer significantly deeper tradable liquidity at competitive prices compared to typical AMM pools.

<figure><img src="../.gitbook/assets/Working Collection (18).png" alt=""><figcaption></figcaption></figure>

**Lower Cost for Supplying Liquidity**

* For asset issuers, establishing secondary market liquidity is essential. Native reduces this cost, as liquid assets supplied to its credit pool earn rewards at the same time. In contrast, AMM pools for yield-bearing assets often generate lower returns due to less prominent trading activity, making liquidity provision comparatively more expensive.
* Check [Yield dashboard](https://defillama.com/yields?project=Native+Credit+Pool\&chain=Ethereum) on Native

<figure><img src="../.gitbook/assets/Working Collection (19).png" alt=""><figcaption></figcaption></figure>
