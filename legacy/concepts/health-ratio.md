# Health Ratio

### Health Ratio

The Native TAL formula introduces a **Health Ratio**, which regulates access to the smart pairing liquidity of total available liquidity in Native Credit Pool.

* **Default Value**: Initially set at **1.0**.
* **Dynamic Adjustment**: May change based on factors affecting the on-chain liquidity of an asset.

### **How Does the Health Ratio Work?**

The Health Ratio **governs the maximum liquidity** an asset issuer can access:

* When an asset issuer deposits tokens into the **Native Credit Pool**, they can immediately access liquidity equivalent to the **bootstrapped liquidity** in the pool.
* To increase available liquidity, asset issuers can:
  1. **Improve their Health Ratio** by meeting performance or liquidity criteria.
  2. **Contribute more liquid assets** to pair with their single-sided LP tokens.
