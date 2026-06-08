# Spot Margin

**Spot Margin** is how professional market makers onboard and trade on [Native Core](../modules/native-core.md) via [Native Pro](../modules/native-pro.md) — posting collateral and trading against **credited available margin**, without first running the full deposit/withdraw pipeline for every trade.

### How It Works

* **Collateral & credit:** an MM posts collateral (via B2B transfer or a Core deposit); available margin is credited against it.
* **Margin accounting:** the order lifecycle (open → pending → filled) deducts and releases margin in real time, with mark-to-market recalculation as prices update.
* **LTV controls:** each asset has a default loan-to-value ratio, with per-MM, per-asset overrides. Stablecoins (USDT/USDC) are the initial accepted collateral.
* **Market-level toggle:** margin can be enabled or disabled per market; disabling rejects new orders while leaving existing resting orders and positions in place.

### Settlement

Settlement covers the full round trip between the EVM vault ([Native Pool](../modules/native-pool.md)) and Native Core for both the integrator (Relay) side and the MM side — moving balances between margin, cash, and on-chain settlement as positions are closed and repaid.

### Funding Rate

In the initial phase, funding is handled as off-chain B2B billing (e.g. monthly), with on-chain accrual planned for the future.

### Related

* [Native Pro](../modules/native-pro.md)
* [Unified Balance](unified-balance.md)
