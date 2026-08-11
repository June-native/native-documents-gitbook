---
hidden: true
noIndex: true
---

# Deposit & Withdraw

**Deposit & Withdraw** is the pipeline that helps users fund and withdraw assets on Native Core. A deposit can fund an available Core balance for trading or, when selected in the app, start earning in Native Pool.

### Key Components

* **Vault contract (Public Networks):** accepts user deposits, transfers them into Native Core, and releases assets on withdrawal.
* **Accounting service:** an independent, multi-node, distributed service to help process deposits and withdrawals.

### Deposit and Withdraw Flow

1. The user deposits on a source chain.
2. The accounting service credits the user on Native Core.
3. The user can move an available Core balance into Native Pool to start earning.

### Withdrawal Flow

1. The user submits a withdrawal request on Native Core; the balance is **locked atomically**.
2. After safe finality on Native Core, the signer network **co-signs** the release.
3. The vault releases and redeems the underlying assets to the user on the destination chain.
4. Once the release is confirmed, the service finalizes the **balance deduction**. If the release fails, the request is flagged for admin intervention (retry or revoke-and-unlock).

### Account Activation & Listing Fees

To prevent dusting attacks, the system limits total addresses, tokens, and markets. A small fee in the chain-native token (ETH/BNB) is charged via the deposit transaction to **activate a new account** or **list a new token**, with no additional token allowances required.

### Related

* [Native Pool / Core Earn](../modules/native-pool.md)
* [Unified Balance](unified-balance.md)
