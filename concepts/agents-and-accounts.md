# Non-custodial Accounts

Native Core accounts are **non-custodial**: an account is identified by an owner address and is controlled by signatures, with no separate registration. Every trading action is a signed transaction, so the chain can verify exactly who authorized each order, cancel, or modify.

#### Non-custodial Owner Accounts

* The owner address is the account's root authority.
* Trading actions are authorized by a recoverable signature over the canonical transaction payload.

#### Agents (Delegated API Keys)

Rather than signing every order with the owner key, an owner can authorize an **agent** — a delegated key that signs on the owner's behalf. Agents are how bots and automated strategies operate without exposing the owner key.
