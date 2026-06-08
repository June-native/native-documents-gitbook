# Unified Balance

A **Unified Balance** is the single account balance a user holds on Native Core, usable across all markets. Instead of juggling funds across chains, pools, and trading pairs, users deposit once and trade everywhere with one balance.

### Balance Components

* **Available balance** — free to trade or withdraw.
* **Locked balance** — reserved by open orders.

### Lock / Unlock Mechanism

When an order is placed, the required funds are **locked atomically**; they are unlocked on cancel or released on fill.

### Staying in Sync

The on-chain unified balance stays in sync with deposits and withdrawals on external chains through the Deposit & Withdraw pipeline, which is enforced by Native Pool vaults and an independent signer network.
