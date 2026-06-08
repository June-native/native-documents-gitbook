---
hidden: true
---

# Deposit & Withdraw

**Deposit & Withdraw** is the pipeline that helps user start trading directly on Native Core. It is enforced by Native Pool vault contracts together with an independent accounting service.

### Key Components

* **Vault contract (Public Networks):** accepts user deposits, locks assets in the vault, and releases them on withdrawal.
* **Accounting service:** an independent, multi-node, distributed service to help processing the deposit and withdraws.

### Deposit and Withdraw Flow

1. The user deposits on a source chain.
2. The accounting service credits user on Native Core.

### Withdrawal Flow

1. The user submits a withdraw request on Native Core; the balance is **locked atomically**.
2. After safe finality on Native Core, the signer network **co-signs** the release.
3. The vault releases and redeems the underlying assets to the user on the destination chain.
4. Once the release is confirmed, the service finalizes the **balance deduction**. If the release fails, the request is flagged for admin intervention (retry or revoke-and-unlock).

### Account Activation & Listing Fees

To prevent dusting attacks, the system limits total addresses, tokens, and markets. A small fee in the chain-native token (ETH/BNB) is charged via the deposit transaction to **activate a new account** or **list a new token**, with no additional token allowances required.

### Related

* [Native Pool](../modules/native-pool.md)
* [Unified Balance](unified-balance.md)
