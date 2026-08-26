---
description: One request that redeems from Native Core Pool and pays out to an EVM wallet, authorized by two signatures.
---

# Withdraw Directly to EVM

[Withdraw](withdraw/README.md) ends at the address's Native Core balance. Getting from there to a wallet on an EVM chain is a second, separate step.

This endpoint does both in one request. The user signs twice — once for the Pool redemption, once for the Native Core withdrawal — and the two run as a single tracked operation.

|                    | Withdraw                    | Withdraw directly to EVM                        |
| ------------------ | --------------------------- | ----------------------------------------------- |
| Ends at            | Native Core balance         | The destination address on the EVM chain        |
| Signatures         | 1                           | 2                                                |
| One in flight per  | Asset                       | **Address**                                      |
| Types              | `scheduled` and `instant`   | `instant`, or a claim of a ready `scheduled`     |

Read [Withdraw](withdraw/README.md) and the Native Core [`withdraw` action](../native-core/post-trade.md#withdraw) first. This page only covers what the combined request adds.

## 1. Choose the source

`source` selects which Pool authorization you are sending. It is explicit; nothing is inferred from which fields you fill in.

| `source`          | What it does                                                             | Pool fee              |
| ----------------- | ------------------------------------------------------------------------ | --------------------- |
| `instant`         | Redeems immediately                                                      | `instant_fee_bps`     |
| `scheduled_claim` | Claims a `scheduled` withdrawal that has reached `claimable_at_unix_ms`  | None                  |

**You cannot create a `scheduled` withdrawal here.** Create it with [`createWithdrawal`](withdraw/README.md#3-sign-and-submit), wait out its window, then send `scheduled_claim` instead of `claimWithdrawal`.

## 2. Size the amount

Two fees apply, at different stages.

**The Pool fee**, charged by `instant` on the gross and rounded up:

```
earn_fee_amount = ⌈ gross_amount × instant_fee_bps ÷ 10000 ⌉
net_amount      = gross_amount − earn_fee_amount
```

At 5 bps, a gross of `1000000001` pays a fee of `500001`, not `500000`. `scheduled_claim` has no Pool fee, so `net_amount` equals the gross of the withdrawal you are claiming.

**The withdrawal fee**, a flat per-asset amount taken when the payout is released on the destination chain. It applies to both sources. Read it from `POST /api/v3/info {"type":"meta"}`, on the `assets[]` entry for your asset:

```json
{ "asset_id": 2, "symbol": "USDT", "balance_decimals": 8,
  "withdraw_fee": "1", "withdraw_fee_atoms": "100000000" }
```

USDC and USDT are 1 unit each. It is charged in atoms of the asset, so quote it to the user from `withdraw_fee`.

```
received = net_amount − withdraw_fee_atoms
```

A gross of `1000000000` USDT at 5 bps: the Pool takes `500000`, leaving a `net_amount` of `999500000`, and the payout takes `100000000`, so `899500000` — 8.995 USDT — arrives in the wallet.

{% hint style="warning" %}
**`core_withdraw.amount` must equal `net_amount` exactly — do not subtract the withdrawal fee from it.** That fee is taken downstream, when the payout is released; the authorization you are signing covers the full `net_amount` leaving the Native Core balance. Subtracting it, sending the gross, or rounding the Pool fee down all return `earn net amount or asset does not match Core withdraw authorization`.
{% endhint %}

## 3. Sign the Pool redemption

Identical to the standalone action, same domain and same typed data:

| `source`          | Sign                                                                              |
| ----------------- | --------------------------------------------------------------------------------- |
| `instant`         | [`CreateWithdrawal`](withdraw/README.md#3-sign-and-submit) with `withdrawType: '2'`         |
| `scheduled_claim` | [`ClaimWithdrawal`](withdraw/README.md#5-claim-a-scheduled-withdrawal) over the `operationId` hash |

The result goes in `user_signature`.

## 4. Sign the Core withdrawal

The Native Core `withdraw` action, signed under the [EIP-712 v4 scheme](../native-core/transaction-signing.md#eip-712-signing-auth_scheme-eip712) — domain `{name:"Native Core", version:"1", verifyingContract:0x0000…0000}`, no `chainId`, `authKind` `1`, `authScope` `0`.

Send it as `core_withdraw`:

| Field                   | Value                                                                       |
| ----------------------- | --------------------------------------------------------------------------- |
| `nonce`                 | Transaction nonce. Current Unix milliseconds                                 |
| `expires_after_unix_ms` | When the authorization stops being valid                                     |
| `asset_id`              | Same asset as the redemption                                                 |
| `amount`                | `net_amount`, in atoms                                                       |
| `dst_chain_id`          | The EVM chain to pay out on                                                  |
| `dst_address`           | The receiving address on that chain                                          |
| `withdraw_nonce`        | Business nonce. Current Unix milliseconds; increment locally within the same millisecond |
| `cloid`                 | `0x` + 32 hex, exactly 16 bytes. Required                                    |
| `signature`             | `0x` + 130 hex                                                               |

{% hint style="warning" %}
**`expires_after_unix_ms` needs at least 10 minutes left when the request arrives, not when the user signs.** A user who signs with exactly ten minutes and then pauses over the wallet prompt is rejected with `Core withdraw authorization has less than 10m0s remaining`. Sign 30 minutes out.
{% endhint %}

`withdraw_nonce` becomes the payout's operation id and carries Native Core's usual 3-day duplicate window. Reusing one returns a duplicate-nonce rejection.

## 5. Submit

```bash
curl -sS "$POOL_API_URL/api/v3/earn" \
  -H 'content-type: application/json' \
  -d '{
    "type": "createWalletWithdrawal",
    "source": "instant",
    "user_address": "0x5555…5555",
    "user_nonce": 3,
    "asset_id": 2,
    "amount": "1000000000",
    "deadline_unix_ms": 1786700000000,
    "user_signature": "0xdddd…dddd",
    "core_withdraw": {
      "nonce": 1786699000000,
      "expires_after_unix_ms": 1786700800000,
      "asset_id": 2,
      "amount": "999500000",
      "dst_chain_id": 56,
      "dst_address": "0x7777…7777",
      "withdraw_nonce": 1786699000000,
      "cloid": "0xabcdefabcdefabcdefabcdefabcdef01",
      "signature": "0xeeee…eeee"
    }
  }'
```

For `scheduled_claim`, replace `user_nonce` / `asset_id` / `amount` with the `operation_id` of the withdrawal you are claiming.

One is accepted per address at a time. The next is refused with `user already has active wallet withdrawal <id>` until the current one finishes.

## 6. Follow the status

```bash
curl -sS "$POOL_API_URL/api/v3/earn" \
  -H 'content-type: application/json' \
  -d '{"type":"walletWithdrawal","user_address":"0x5555…5555","wallet_withdrawal_id":4}'
```

```json
{
  "code": 0,
  "data": {
    "wallet_withdrawal_id": 4,
    "user_address": "0x5555…5555",
    "source": "instant",
    "asset_id": 2,
    "gross_amount": "1000000000",
    "net_amount": "999500000",
    "earn_fee_amount": "500000",
    "earn_fee_bps_snapshot": 5,
    "status": "DONE",
    "suggested_action": "NONE",
    "earn_withdraw_operation_id": "withdraw:0x5555…5555:3",
    "core_tx_hash": "0x3c59…8bd",
    "accounting_withdraw_operation_id": "withdraw:0x5555…5555:1786699000000",
    "evm_tx_hash": "0xb3fb…8c0c",
    "accepted_at_unix_ms": 1786699001000,
    "redeem_submitted_at_unix_ms": 1786699002000,
    "redeem_settled_at_unix_ms": 1786699012000,
    "withdraw_submitted_at_unix_ms": 1786699013000,
    "completed_at_unix_ms": 1786699340000
  },
  "message": "success"
}
```

`{"type":"walletWithdrawals"}` returns the same record in a keyset page, filterable by `asset_id` and `status`, paged with `before_id`.

Both reads take `user_address` and serve only that address's records. A `wallet_withdrawal_id` belonging to someone else returns the same `wallet withdrawal not found` as one that does not exist.

The redemption typically settles in seconds; the EVM leg follows destination-chain confirmation and usually takes a few minutes.

```
ACCEPTED → REDEEM_SUBMITTED → REDEEM_SETTLED → WITHDRAW_SUBMITTED → DONE
```

| Status               | Meaning                                                    |
| -------------------- | ---------------------------------------------------------- |
| `ACCEPTED`           | Both signatures verified, work not started                 |
| `REDEEM_SUBMITTED`   | Pool redemption submitted                                  |
| `REDEEM_SETTLED`     | `net_amount` is in the Native Core balance                 |
| `WITHDRAW_SUBMITTED` | The Core withdrawal is submitted, awaiting the EVM payout   |
| `DONE`               | Terminal. Paid out on the destination chain                 |
| `REDEEM_FAILED`      | Terminal. The redemption failed; nothing moved              |
| `NEEDS_RESUBMIT`     | Terminal. The redemption succeeded, the payout did not      |
| `WITHDRAW_STUCK`     | Terminal. The payout exhausted its attempts                 |

These eight are also the accepted values for the `status` filter.

{% hint style="info" %}
**Branch on `suggested_action`, not on `status`.** Two terminal states need something the status name does not imply.

| `suggested_action`       | What to tell the user                                                                    |
| ------------------------ | ---------------------------------------------------------------------------------------- |
| `WAIT`                   | Still running                                                                             |
| `NONE`                   | Finished, or failed with nothing owed                                                     |
| `RESUBMIT_CORE_WITHDRAW` | The `net_amount` is in their Native Core balance. Retrying here cannot work; withdraw it with a plain Core [`withdraw`](../native-core/post-trade.md#withdraw) |
| `CONTACT_SUPPORT`        | Held for an operator. This is the one terminal state that keeps the address's slot occupied, so no new wallet withdrawal is accepted until it is resolved |
{% endhint %}

## Fields

| Field                              | Notes                                                                     |
| ---------------------------------- | ------------------------------------------------------------------------- |
| `wallet_withdrawal_id`             | The handle for both reads                                                  |
| `source`                           | `instant` or `scheduled_claim`, as created                                 |
| `gross_amount` / `net_amount`      | Before and after the Pool fee. `net_amount` is what leaves the Native Core balance, **not** what arrives on the destination chain — the withdrawal fee comes off after it |
| `earn_fee_bps_snapshot`            | The rate at creation, so a later config change does not restate history    |
| `earn_withdraw_operation_id`       | The Pool withdrawal backing this one                                       |
| `core_tx_hash`                     | The Native Core withdrawal transaction                                     |
| `accounting_withdraw_operation_id` | Join key into `POST /api/v3/accounting {"type":"withdrawOrder"}`, where `dst_chain_id` and the payout's own state live |
| `evm_tx_hash`                      | The destination-chain payout                                               |
| `failure_code`                     | Present once an attempt has failed, on non-terminal records too            |
| `failure_message`                  | Best effort. **Branch on `failure_code`**; this field is omitted whenever the underlying text is not fit to publish |

Timestamps are `null` until reached. Hashes and ids are absent as keys until they exist.

## Rate limits

Metered per `user_address`, like every other Pool request.

| Request                                     | Budget       |
| ------------------------------------------- | ------------ |
| `createWalletWithdrawal`                    | 1 per second |
| `walletWithdrawal`, `walletWithdrawals`     | 3 per second |

## What can go wrong

| Message                                                          | Cause                                                    | What to do                                              |
| ---------------------------------------------------------------- | -------------------------------------------------------- | -------------------------------------------------------- |
| `earn net amount or asset does not match Core withdraw authorization` | `core_withdraw.amount` is not `net_amount`            | Round the fee **up**, then subtract                       |
| `Core withdraw authorization has less than 10m0s remaining`       | `expires_after_unix_ms` is too close                      | Sign 30 minutes out                                       |
| `Core withdraw signer does not match Earn user`                   | The Core typed data does not match what you sent          | Check `authKind`/`authScope`, that no `chainId` reached the domain, and that both signatures came from one object |
| `withdrawal signer does not match user`                           | The redemption's typed data does not match                 | Same checks as [Withdraw](withdraw/README.md#what-can-go-wrong)    |
| `Core withdraw cloid must be 16-byte 0x hex`                      | `cloid` is missing or the wrong length                    | `0x` + 32 hex characters                                  |
| `user already has active wallet withdrawal <id>`                  | One is already in flight for this address                 | Poll it to a terminal state first; the limit is per address, not per asset |
| `wallet withdrawal not found`                                     | No such record, **or** it belongs to another address      | Both cases return the same response                       |

## Next steps

{% content-ref url="withdraw/README.md" %}
[withdraw/README.md](withdraw/README.md)
{% endcontent-ref %}

{% content-ref url="reference.md" %}
[reference.md](reference.md)
{% endcontent-ref %}
