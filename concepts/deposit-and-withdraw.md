# Deposit & Withdraw

**Deposit & Withdraw** is the pipeline that keeps a user's [Unified Balance](unified-balance.md) on [Native Core](../modules/native-core.md) in sync with their assets on external chains. It is enforced by [Native Pool](../modules/native-pool.md) vault contracts together with an independent **signer network** and an accounting service.

### Key Components

* **Vault contract (EVM):** accepts user deposits, locks assets in the vault, and releases them on withdrawal.
* **Signer network:** a set of independent signers that co-confirm deposit crediting on Native Core and co-sign withdrawal releases on the external chain. Multi-party agreement is required before any funds move.
* **Accounting service:** detects on-chain events, submits the corresponding balance transactions on Native Core, and waits for safe finality before finalizing.

### Deposit Flow

1. The user deposits on a source chain; the vault locks the funds and emits an event.
2. The accounting service **provisionally credits** the user's unified balance on Native Core.
3. Once the source chain reaches **safe finality**, signers verify the state and the service **finalizes** the credit.
4. If confirmation fails (e.g. a reorg), the provisional credit is **revoked**.

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
