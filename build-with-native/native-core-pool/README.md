---
description: >-
  Deposit into the Native Core Pool vault, earn distributed yield, and withdraw
  — the two deposit routes, the signed withdrawal actions, and what to poll.
---

# Native Core Pool

Native Core Pool pays yield on assets held in its vault. Funds enter from an EVM chain or from a Native Core balance, sit in an earning balance, and leave through a signed withdrawal request.

This section is for teams building their own Pool experience — wallets, custodians, and front-ends that cannot route users through the Native web app. Reads require only HTTP. Deposits and withdrawals require a wallet that can sign EIP-712 typed data. The cross-chain deposit route also requires an EVM RPC endpoint.

{% content-ref url="deposit.md" %}
[deposit.md](deposit.md)
{% endcontent-ref %}

{% content-ref url="yield.md" %}
[yield.md](yield.md)
{% endcontent-ref %}

{% content-ref url="withdraw/" %}
[withdraw](withdraw/)
{% endcontent-ref %}

{% content-ref url="reference.md" %}
[reference.md](reference.md)
{% endcontent-ref %}

## The model

Each direction gives you one request to send and one query to poll.

|              | Deposit                                                        | Withdraw                            |
| ------------ | -------------------------------------------------------------- | ----------------------------------- |
| You send     | A vault `deposit()` on an EVM chain, or a Core `transfer`      | A signed `createWithdrawal` request |
| It lands in  | `earn_balance`                                                 | Your Native Core balance            |
| You poll     | `deposits`                                                     | `withdrawals`                       |
| Correlate on | `src_tx_hash` on the EVM route, your `cloid` on the Core route | `operation_id`                      |

A deposit cannot be cancelled or reversed. A scheduled withdrawal can be cancelled until it becomes claimable. An instant withdrawal cannot be cancelled at any point.

## Endpoints

Native Core Pool is served on mainnet at `https://api-ui.native.org`. There is no API key; requests are metered per `user_address`.

```bash
POOL_API_URL=https://api-ui.native.org
```

Every Pool operation is `POST /api/v3/earn` with a `type` field naming it.

**The HTTP status is always 200.** Branch on the `code` field in the body instead.

```json
{ "code": 0, "data": { }, "message": "success" }
```

`code: 0` is success. Anything else is a failure, and a failure body has no `data` key at all:

```json
{ "code": 131004, "message": "limit exceeds max 200" }
```

{% hint style="info" %}
Read `code` before you read `data`. A rate-limited or failed request that you treat as a 200 with an empty body reads as "no deposits yet" and never recovers. The full list is in [Error codes](reference.md#error-codes).
{% endhint %}

Every response carries a `trace_id` header. Include it when you report a problem.

## Rate limits

The budgets below apply to `POST /api/v3/earn`. Requests are metered **per `user_address`**, and each `type` holds its own budget.

| Request                                                                   | Budget       |
| ------------------------------------------------------------------------- | ------------ |
| `config`                                                                  | Not metered  |
| Reads: `account`, `deposits`, `withdrawals`, `withdrawal`, `yieldHistory`, `walletWithdrawal`, `walletWithdrawals` | 3 per second |
| Writes: `createWithdrawal`, `claimWithdrawal`, `cancelWithdrawal`, `createWalletWithdrawal` | 1 per second |

The window resets every second. A request over the budget returns `code: 201005` and carries no `data`.

```json
{ "code": 201005, "message": "rate reach limit" }
```

The Core-internal deposit route posts to `api.native.org/trade`, which is metered separately: **1 request per second per IP**, applied before the signature is checked. That endpoint returns HTTP `429` rather than the Pool envelope; see [Rate limits & errors](../native-core/api-access.md#rate-limits-errors).

## Amounts and time

Amounts are **integer atom strings**, never numbers and never decimals.

| Where                     | Precision                                                                   |
| ------------------------- | --------------------------------------------------------------------------- |
| Pool balances and amounts | The asset's `balance_decimals` from `config`, currently `8` for every asset |
| Cross-chain deposit call  | The source ERC20's own `decimals`                                           |

The two precisions differ, so a cross-chain deposit requires a conversion. 1 USDT is `1000000` to the ERC20 on Ethereum (6 decimals) and `100000000` once credited to the Pool (8 decimals).

Amounts on 18-decimal tokens exceed JavaScript's safe integer range. Parse every amount with `BigInt`, not `Number`.

Every `*_unix_ms` field is a millisecond timestamp.

## Discovery

Read the routing table at runtime. Operations list and delist assets and change limits, so treat every address, id, and limit below as a read rather than a constant.

{% hint style="info" %}
Addresses, transaction hashes and signatures in the examples throughout this section are placeholders, shortened to `0x1111…1111` form for readability. Substitute full-length values, and resolve every address at runtime rather than copying one into your code as a constant.
{% endhint %}

| What you need                                           | Where it comes from                                  |
| ------------------------------------------------------- | ---------------------------------------------------- |
| `asset_id`, `balance_decimals`, fees, withdrawal limits | [`config`](reference.md#config)                      |
| Vault address and the EIP-712 domain                    | [`config`](reference.md#config)                      |
| Depositable ERC20s and vault addresses per chain        | [Deposit & Withdraw](../deposit-withdraw/#discovery) |

`config.assets[]` lists every asset the Pool accepts.

```bash
curl -sS "$POOL_API_URL/api/v3/earn" \
  -H 'content-type: application/json' \
  -d '{"type":"config"}'
```

```json
{
  "code": 0,
  "data": {
    "native_chain_id": 696969,
    "vault_address": "0x1111…1111",
    "fee_wallet_address": "0x2222…2222",
    "snapshot_time_utc": "00:00:00.000",
    "withdrawal_paused": false,
    "eip712_domain": {
      "name": "Native Core Earn",
      "version": "1",
      "verifyingContract": "0x1111…1111"
    },
    "assets": [
      {
        "asset_id": 2,
        "symbol": "USDT",
        "balance_decimals": 8,
        "deposit_enabled": true,
        "scheduled_withdraw_enabled": true,
        "instant_withdraw_enabled": true,
        "withdraw_pending_seconds": 10,
        "instant_fee_bps": 5,
        "min_withdraw_amount": "0",
        "max_single_withdraw_amount": "0"
      }
    ]
  },
  "message": "success"
}
```

Three of these fields carry a meaning their name does not suggest:

* `vault_address` is both the recipient of a Core-internal deposit and the EIP-712 `verifyingContract`.
* `withdraw_pending_seconds` is the wait before a scheduled withdrawal can be claimed. It is also the entire window in which the withdrawal can be cancelled.
* `min_withdraw_amount` and `max_single_withdraw_amount` use `"0"` to mean **no limit**, not a limit of zero. Only positive values are enforced.

Field definitions are in [Reference](reference.md#config).

## Accounts

A Pool balance belongs to an address, and that address needs a Native Core account. The account is created by its owner's first deposit, which carries a one-time activation fee — see [Deposit](../deposit-withdraw/deposit.md#2-size-the-activation-fee).

An address can have **one withdrawal in flight per asset**. A USDT withdrawal does not block a USDC one. Check `active_withdraw_operation_id` on that asset's entry in [`account`](reference.md#account) `balances[]` before creating another. An empty string means nothing is in flight for that asset.

Credit accounts cannot deposit into or withdraw from the Pool. See [Account Types](../native-core/account-types.md).
