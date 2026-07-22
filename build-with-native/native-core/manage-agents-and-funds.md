---
description: The owner-signed actions — approve or revoke an API wallet, withdraw, and de-risk a credit account.
---

# Manage agents & funds

A handful of actions move value or manage who can trade for you. They are **owner-signed**: your main wallet signs them under `auth_scheme: "eip712"`, never the API wallet. Each is a real `POST /trade` action the API accepts directly — sign it with your main wallet — or do it through the **[Native web app](https://app.native.org/markets/ETH-USDT?agentWallets=agents)**.

The API wallet's own allowlist is only `order` / `cancel` / `cancelAll` / `modify` / `batch` — it can trade, never move funds. See [Nonces & API Wallets](nonces-and-api-wallets.md).

## Approve or revoke an API wallet

* [`approveAgent`](post-trade.md#approveagent) — authorize an agent key in a slot (`0`–`3`). This is what creates an API wallet; the web app returns the connection bundle.
* [`revokeAgent`](post-trade.md#revokeagent) — retire a slot. The old agent's `agent_epoch` stops validating.

Both are owner EIP-712 signatures, not batchable, and carry no `agent_epoch`.

## Withdraw funds

* [`withdraw`](post-trade.md#withdraw) — move an asset off Native to a destination chain. Owner EIP-712; enforces per-chain/asset minimums and a withdraw fee. A credit account cannot withdraw.

Deposits are on-chain transfers from your main wallet in the web app, not a `/trade` action.

## De-risk a credit account

Only relevant if the protocol granted you a [credit account](account-types.md).

* [`settle`](post-trade.md#settle) — move a long position out of the credit account into a spot account's `available` balance.
* [`repay`](post-trade.md#repay) — spend a spot account's balance to cover a credit account's short.

Both are owner EIP-712, single-signature, and carry a required `cloid` used only for `txStatusByCloid` lookups.

## Signing

These use the **EIP-712 v4** scheme (MetaMask-compatible), not the legacy binary payload. The typed-data fields are in [Transaction Signing](transaction-signing.md#eip-712-signing-auth_scheme-eip712).

## Next steps

* [POST /trade](post-trade.md) — every action and field
* [Account Types](account-types.md) — spot vs credit accounts
