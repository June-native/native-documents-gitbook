---
description: Deposit into the Native Core Pool vault, earn distributed yield, and withdraw — the two deposit routes, the signed withdrawal actions, and what to poll.
---

# Native Core Pool

Native Core Pool pays yield on assets held in its vault. Funds enter from an EVM chain or from a Native Core balance, sit in an earning balance, and leave through a signed withdrawal request.

This section is for teams building their own Pool experience — wallets, custodians, and front-ends that cannot route users through the Native web app. Reads need nothing but HTTP. Deposits and withdrawals need a wallet that can sign EIP-712 typed data, and the cross-chain route also needs an EVM RPC endpoint.

{% content-ref url="deposit.md" %}
[deposit.md](deposit.md)
{% endcontent-ref %}

{% content-ref url="yield.md" %}
[yield.md](yield.md)
{% endcontent-ref %}

{% content-ref url="withdraw.md" %}
[withdraw.md](withdraw.md)
{% endcontent-ref %}

{% content-ref url="reference.md" %}
[reference.md](reference.md)
{% endcontent-ref %}

## The model

Money moves in one direction at a time, and each move has exactly one thing to poll.

|             | Deposit                                                   | Withdraw                                       |
| ----------- | --------------------------------------------------------- | ---------------------------------------------- |
| You send    | A vault `deposit()` on an EVM chain, or a Core `transfer` | A signed `createWithdrawal` request            |
| It lands in | `earn_balance`                                             | Your Native Core balance                       |
| You poll    | `deposits`                                                 | `withdrawals`                                  |
| Correlate on | `src_tx_hash` on the EVM route, your `cloid` on the Core route | `operation_id`                             |

A deposit cannot be cancelled. A withdrawal can, but only a scheduled one, and only before it becomes claimable.

## Endpoints

Native Core Pool is served on mainnet at `https://api-ui.native.org`. There is no API key; requests are metered by source IP.

```bash
POOL_API_URL=https://api-ui.native.org
```

Every Pool operation is `POST /api/v3/earn` with a `type` field naming it. Two supporting reads sit elsewhere: the token registry at `GET /api/v3/core/registry`, and the deposit fee quote at `POST /api/v3/accounting`.

**The HTTP status is always 200.** Branch on the `code` field in the body instead.

```json
{ "code": 0, "data": { }, "message": "success" }
```

`code: 0` is success. Anything else is a failure, and a failure body has no `data` key at all:

```json
{ "code": 131004, "message": "limit exceeds max 200" }
```

{% hint style="warning" %}
Read `code` before you read `data`. A rate-limited or failed request that you treat as a 200 with an empty body reads as "no deposits yet" and never recovers. The full list is in [Error codes](reference.md#error-codes).
{% endhint %}

Every response carries a `trace_id` header. Include it when you report a problem.

## Rate limits

Requests are metered per source IP, and **each `type` gets its own budget**, so a page that reads `config`, `account` and `deposits` at once does not compete with itself.

Poll at no more than **one request per second per `type`** and leave the rest of the budget for retries. Signed writes are metered more tightly than reads; one withdrawal action at a time is well inside the limit.

Over the limit, the response is `code: 201005` with an empty body. Wait for the next second and retry.

```json
{ "code": 201005, "message": "rate reach limit" }
```

## Amounts and time

Amounts are **integer atom strings**, never numbers and never decimals.

| Where                              | Precision                                                              |
| ---------------------------------- | ---------------------------------------------------------------------- |
| Pool balances and amounts          | The asset's `balance_decimals` from `config`, currently `8` for every asset |
| Cross-chain deposit call           | The source ERC20's own `decimals` from the registry                     |

The two differ, so a cross-chain deposit needs a conversion: 1 USDT is `1000000` to the ERC20 on Ethereum (6 decimals) and `100000000` once credited to the Pool (8 decimals).

Sizes exceed JavaScript's safe integer range on 18-decimal tokens. Parse every amount with `BigInt`, not `Number`.

Every `*_unix_ms` field is a millisecond timestamp.

## Discovery

Read the routing table at runtime. Assets are listed and delisted, and limits are changed by operations, so treat every address, id, and limit below as a read rather than a constant.

| What you need                                                     | Where it comes from                                            |
| ----------------------------------------------------------------- | -------------------------------------------------------------- |
| `asset_id`, `balance_decimals`, fees, withdrawal limits            | [`config`](reference.md#config)                                 |
| Vault address and the EIP-712 domain                               | [`config`](reference.md#config)                                 |
| Depositable ERC20s per chain, and their minimums                   | [`GET /api/v3/core/registry`](reference.md#get-apiv3coreregistry) |
| The gas-token fee a first deposit must carry                       | [`depositInitFee`](reference.md#depositinitfee)                 |

`config` is the authority on which assets the Pool accepts. The registry lists every token Native Core supports, which is a much larger set — a token in the registry with no matching `asset_id` in `config` cannot be deposited into the Pool.

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
    "vault_address": "0x4017d1520D734C8fa6dd0fDEba150C3E0fEA65Ea",
    "fee_wallet_address": "0x546EC2c2f548F64AE3D21f1B408A8023D116fFa3",
    "snapshot_time_utc": "00:00:00.000",
    "withdrawal_paused": false,
    "eip712_domain": {
      "name": "Native Core Earn",
      "version": "1",
      "verifyingContract": "0x4017d1520D734C8fa6dd0fDEba150C3E0fEA65Ea"
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

Three values in that response decide how your integration behaves:

* `vault_address` is both the recipient of a Core-internal deposit and the EIP-712 `verifyingContract`.
* `withdraw_pending_seconds` is the wait before a scheduled withdrawal can be claimed. It is also the entire window in which the withdrawal can be cancelled.
* `min_withdraw_amount` and `max_single_withdraw_amount` use `"0"` to mean **no limit**, not a limit of zero. Only positive values are enforced.

Field definitions are in [Reference](reference.md#config).

## Accounts

A Pool balance belongs to an address, and that address needs a Native Core account. The account is created by its owner's first deposit, which carries a one-time activation fee — see [Deposit](deposit.md#2-size-the-activation-fee).

One address can have **one withdrawal in flight at a time**, across all assets. `active_withdrawal` in [`account`](reference.md#account) is the check; a second request is rejected while it is non-null.

Credit accounts cannot use this path. See [Account Types](../native-core/account-types.md).
