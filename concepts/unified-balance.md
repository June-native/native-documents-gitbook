# Unified Balance

A **Unified Balance** is the single account balance a user holds on [Native Core](../modules/native-core.md), usable across all markets. Instead of juggling funds across chains, pools, and trading pairs, users deposit once and trade everywhere with one balance.

### Balance Components

Each account's balance is broken down into:

* **Available balance** — free to trade or withdraw.
* **Locked balance** — reserved by open orders, margin, or pending withdrawals.
* **Unrealized PnL** — for open perpetual positions (when perp markets are live).

### Lock / Unlock Mechanism

* When an order is placed, the required funds are **locked atomically**; they are unlocked on cancel or released on fill.
* When a withdrawal is submitted, the amount is locked atomically; it is unlocked if the withdrawal is cancelled or fails, and deducted once the withdrawal succeeds.

### Staying in Sync

The on-chain unified balance stays in sync with deposits and withdrawals on external chains through the [Deposit & Withdraw](deposit-and-withdraw.md) pipeline, which is enforced by [Native Pool](../modules/native-pool.md) vaults and an independent signer network.

### Related

* [Native Pool](../modules/native-pool.md)
* [Deposit & Withdraw](deposit-and-withdraw.md)
