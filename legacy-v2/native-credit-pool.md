---
description: A Liquidity Pool Where LP-supplied Capital Fuels Credit-based Trading
---

# Native Credit Pool

**Native Credit Pool** is a unified, single-sided asset supply pool that enables market makers to borrow assets for trading, thereby enhancing liquidity in the on-chain spot market.

### Single-Sided Liquidity Provision

With a [single-sided liquidity](single-sided-liquidity-pools.md) model, liquidity providers are no longer constrained by pair-based deposits and suffer the impermanent loss. Assets are borrowed and returned exactly as deployed, along with accrued credit interest.

<figure><img src="../.gitbook/assets/Single-Sided Liquidity Provision.png" alt=""><figcaption></figcaption></figure>

### Liquidity Pairing Mechanism

Native introduces a [**Liquidity Pairing**](liquidity-pairing.md) mechanism that dynamically allocates liquidity as needed to optimize capital efficiency, yield, and risk management.

<figure><img src="../.gitbook/assets/Liquidity Paring Mechanism.png" alt=""><figcaption></figcaption></figure>

### **Liquidity Tiers in Native Credit Pool**

Assets added to the **Native Credit Pool** are categorized into **three distinct tiers**:

* **Smart Pairing Liquidity**: Utilized for all trades with the higher yield and lowest risk. Provided primarily by retail liquidity providers (LPs).
* **Dedicated Pairing Liquidity**: Used specifically for trading the dedicated token. Provided by both retail LPs and asset issuers.
* **Bootstrapping Liquidity**: Acts as a buffer fund, utilized for all trades to ensure liquidity stability and credit-based trading. Provided by market makers.

### **Key Benefits**

* **Tiered Risk Management**: Liquidity is allocated across risk tiers, ensuring tailored exposure for LPs and optimal capital deployment.
* **Dynamic Liquidity Allocation**: Assets are allocated intelligently to meet trading demands, offering smart yield opportunities for liquidity providers.
* **Cost-Effective Capital Deployment**: Asset issuers benefit from efficient capital allocation and optimized incentives for liquidity provisioning.

### Related Concepts

* [Total Available Liquidity](total-available-liquidity.md)
* [Health Ratio](health-ratio.md)
