---
description: Enabling PMM Pricing and Atomic Swaps with On-Chain Credit Management
layout:
  width: default
  title:
    visible: true
  description:
    visible: true
  tableOfContents:
    visible: true
  outline:
    visible: true
  pagination:
    visible: true
  metadata:
    visible: true
  tags:
    visible: true
  actions:
    visible: true
---

# Native Swap Engine

### Onchain Executable Orderbook

Native introduces a **high-frequency, auto-sign orderbook** that seamlessly aligns with prevailing market prices, offering reliable quotes to ensure low latency and a high success rate by utilizing real-time on-chain inventory.

<figure><img src="../.gitbook/assets/On-chain Executable Orderbook.png" alt=""><figcaption></figcaption></figure>

### **Credit-Based Swap**

#### **Example: Credit-Based Swap in User Action**

<figure><img src="../.gitbook/assets/Credit-Based Swap new.png" alt=""><figcaption></figcaption></figure>

{% hint style="info" %}
**Scenario**: Josh, a retail trader, wants to swap 1 ETH for USDC using a Native-powered aggregator/solver. PMM1, a trading house, has pledged **100 USDC as collateral**, securing a **$100 credit** from Native.<br>

**Step-by-Step Process**:

* (**T-1:** Josh entered a swap intent to the aggregator. The aggregator checks the latest price and depth from Native orderbook endpoint)
* **T0**: Josh submits a swap intent through the aggregator.
* **T1**: Native Swap Engine receives the intent. PMM1 offers a **quote of 3600 USDC** using the Native Credit Pool as inventory, by leveraging its **$100 credit**, even though it doesn’t hold USDC in its wallet.
* **T2**: Josh signs and approves the transaction to swap **1 WETH for 3600 USDC**.
* **T3**:
  * Native Credit Pool pays **Josh 3600 USDC** and receives **1 WETH**.
  * Native records a **long 1 WETH** and a **short 3600 USDC** position on PMM1’s credit record.
  * This entire process occurs **atomically in a single transaction**.
{% endhint %}

#### **Post-trade: Credit Risk Management**

<figure><img src="../.gitbook/assets/image (71).png" alt=""><figcaption></figcaption></figure>

{% hint style="info" %}
**What if Price Moves?**

* If the price of WETH drops to **$3598**, **$2** is cut from PMM credit
* If the price of WETH stays **above $3500**, PMM1’s credit covers its position. The 100 USD collateral is used to cover the gap.
* If the price drops **below $3500**, the 100 USDC collateral **and** the 1 WETH would be **subject to liquidation**.
{% endhint %}

{% hint style="info" %}
The above example assumes the [collateral factors](credit-based-swap/collateral-factor.md) for all the assets are set to 100%. [Learn more](credit-based-swap/collateral-factor.md)
{% endhint %}
