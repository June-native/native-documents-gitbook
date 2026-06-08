# Agents & Accounts

Native Core accounts are **non-custodial**: an account is identified by an owner address and is controlled by signatures, with no separate registration. Every trading action is a signed transaction, so the chain can verify exactly who authorized each order, cancel, or modify.

### Non-custodial Accounts

* The owner address is the account's root authority. Funds are credited to it through the [Deposit & Withdraw](deposit-and-withdraw.md) pipeline and tracked as a [Unified Balance](unified-balance.md).
* Trading actions are authorized by a recoverable signature over the canonical transaction payload — not over the JSON text. See the signing reference for the exact encoding.

### Agents (Delegated API Keys)

Rather than signing every order with the owner key, an owner can authorize an **agent** — a delegated key that signs on the owner's behalf. Agents are how bots and automated strategies operate without exposing the owner key.

* **Direct-owner vs agent signing:** a transaction is submitted either directly by the owner, or by an active agent on the owner's behalf. A key registered as an active agent cannot also submit in direct-owner mode.
* **`agent_epoch`:** agent submissions carry the epoch of the agent slot they were authorized under. A mismatch is rejected (`AgentEpochMismatch`), which lets an owner rotate or revoke agents cleanly.
* **Allowed actions:** agents may only perform action kinds permitted for agent signatures; disallowed actions are rejected (`AgentActionNotAllowed`). Admin/operator actions are never available to agents.
* **Querying agents:** the active agent slots for an owner are returned by the `userAgents` query.

### Related

* [Unified Balance](unified-balance.md)
* [Native Core](../modules/native-core.md)
* [Transaction Signing](../build-with-native/native-core/transaction-signing.md)
