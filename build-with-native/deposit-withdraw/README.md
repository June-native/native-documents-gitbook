---
description: Build your own funding flow into and out of Native Core — the vault contract, the signed withdrawal, and how to confirm each direction settled.
---

# Deposit & Withdraw

Funding spans two chains. A deposit starts on an EVM chain and finishes on Native Core; a withdrawal starts on Native Core and finishes on an EVM chain. Each direction gives you exactly one transaction to send and one thing to poll.

This section is for teams building their own deposit and withdrawal experience — wallets, aggregators, custodians, and trading front-ends that cannot route users through the Native web app. You need an EVM RPC endpoint and Native Core's two REST endpoints. No other Native-hosted service is involved.

{% content-ref url="deposit.md" %}
[deposit.md](deposit.md)
{% endcontent-ref %}

{% content-ref url="withdraw.md" %}
[withdraw.md](withdraw.md)
{% endcontent-ref %}

{% content-ref url="vault-contract.md" %}
[vault-contract.md](vault-contract.md)
{% endcontent-ref %}

## The model

Both directions are asymmetric: you act on one chain, and Native's settlement pipeline acts on the other. You never wait on your own second transaction.

|                               | Deposit                                 | Withdraw                               |
| ----------------------------- | --------------------------------------- | -------------------------------------- |
| You send                      | `deposit()` on the vault contract        | A signed `withdraw` action to `/trade` |
| Native settles the other side | credits the balance a few minutes later | releases tokens from the vault         |
| You confirm with              | `POST /info` `deposits`                  | vault `usedNonces(...)`                |
| Correlation key               | The `Deposit` event's `nonce`            | The `withdraw_nonce` you chose         |

Neither direction can be cancelled or reversed once the first transaction lands. A deposit that fails validation is held for manual review, not returned; a withdrawal debits the Native balance the moment the action executes, before anything moves on EVM.

## Endpoints

Native Core exposes two REST endpoints, and funding uses a handful of queries on each. The full contract for both is in [Native Core API](../native-core/README.md).

```bash
API_URL=https://api.native.org
```

{% hint style="info" %}
`/info` is rate limited to **1 request per second per IP**. Every polling loop in this section needs an interval at or above that, and a shared budget if you run several at once.
{% endhint %}

## Discovery

Resolve the routing table at runtime. Vaults are redeployed and asset listings change, so treat every address and limit as a read, never a constant.

| What you need                                  | Where it comes from                                                                             |
| ---------------------------------------------- | ----------------------------------------------------------------------------------------------- |
| Vault address per chain                        | `POST /info` [`accountingDepositContracts`](../native-core/post-info.md#accountingdepositcontracts) |
| Depositable tokens on a chain                  | vault `getSupportedUnderlyings()`                                                                 |
| `asset_id`, `balance_decimals`, withdrawal fee | `POST /info` [`assets`](../native-core/post-info.md#assets)                                        |
| Withdrawable `(chain, asset)` and minimums     | `POST /info` [`accountingWithdrawTokens`](../native-core/post-info.md#accountingwithdrawtokens)    |
| Gas-token price, to quote the activation fee   | `POST /info` [`markPrices`](../native-core/post-info.md#markprices)                                |

The current vault addresses are listed in [Vault Contract](vault-contract.md#addresses).

## Accounts

A Native Core account is created by its owner's first deposit. That first deposit carries a one-time activation fee, which [Deposit](deposit.md#2-size-the-activation-fee) covers in full — get it wrong and the funds are not credited.

Credit accounts cannot deposit or withdraw through this path. See [Account Types](../native-core/account-types.md).
