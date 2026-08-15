---
description: Take funds out of Native Core Pool — the two withdrawal types, the signed create, claim and cancel actions, and the status lifecycle.
---

# Withdraw

A withdrawal moves funds from `earn_balance` back to the address's Native Core balance. Every step is a signed request; nothing is submitted on-chain by you.

Choose a type at creation. The choice is permanent.

|              | `scheduled`                   | `instant`                        |
| ------------ | ----------------------------- | -------------------------------- |
| Fee          | None                          | `instant_fee_bps` on the gross   |
| Wait         | `withdraw_pending_seconds`    | None                             |
| Claim step   | Required                      | None                             |
| Cancellable  | During the wait only          | Never                            |
| Ends at      | `claimed`                     | `completed`                      |

**One address can have one withdrawal in flight at a time**, across all assets. The next one cannot be created until the current one reaches a terminal state.

Read [Native Core Pool](README.md) first for the base URL, the response envelope, and discovery.

## 1. Check the account

```bash
curl -sS "$POOL_API_URL/api/v3/earn" \
  -H 'content-type: application/json' \
  -d '{"type":"account","user_address":"0xbf381e…"}'
```

```json
{
  "code": 0,
  "data": {
    "user_address": "0xbf381e…",
    "next_user_nonce": 3,
    "active_withdraw_operation_id": "",
    "active_withdrawal": null,
    "balances": [
      {
        "asset_id": 2,
        "earn_balance": "4000000000",
        "in_queue_balance": "0",
        "withdraw_locked_balance": "0",
        "lifetime_deposit": "6200000000",
        "lifetime_yield": "0",
        "lifetime_withdraw": "1200000000"
      }
    ]
  },
  "message": "success"
}
```

* `next_user_nonce` is required to create a withdrawal. Read it fresh each time.
* **`active_withdraw_operation_id` is the gate.** An empty string means nothing is in flight. Use this rather than `active_withdrawal`, which is a convenience view that can be `null` even while a withdrawal is active. Creating against a stale reading costs the user a signature and returns `user already has an active withdrawal`.

## 2. Size the amount

You sign and submit the **gross** amount. The fee comes out of it.

```
fee_amount  = gross_amount × instant_fee_bps ÷ 10000   (instant only, truncated to whole atoms)
user_amount = gross_amount − fee_amount
```

A user asking to withdraw 100 receives 99.95 at 5 bps. Deriving `gross` from a target payout is your side of the calculation.

`min_withdraw_amount` and `max_single_withdraw_amount` from [`config`](reference.md#config) are checked against the **gross** amount too. Both use `"0"` to mean no limit, and only positive values are enforced.

Amounts are 8-decimal atom strings, digits only. A zero-padded or signed value is rejected before it reaches the signature check.

## 3. Sign and submit

The domain is the `eip712_domain` object from `config`, passed through unchanged.

{% hint style="danger" %}
**Do not add `chainId` to the domain.** Native Core is not an EVM chain, so a wallet cannot switch to it, and a domain carrying `chainId` makes the signature request impossible to complete.
{% endhint %}

```ts
const { eip712_domain: domain } = config

const typedData = {
  domain,
  primaryType: 'CreateWithdrawal',
  types: {
    EIP712Domain: [
      { name: 'name',              type: 'string'  },
      { name: 'version',           type: 'string'  },
      { name: 'verifyingContract', type: 'address' },
    ],
    CreateWithdrawal: [
      { name: 'user',         type: 'address' },
      { name: 'userNonce',    type: 'uint256' },
      { name: 'withdrawType', type: 'uint8'   },
      { name: 'assetId',      type: 'uint256' },
      { name: 'amount',       type: 'uint256' },
      { name: 'deadline',     type: 'uint256' },
    ],
  },
  message: {
    user: userAddress,
    userNonce: String(nextUserNonce),
    withdrawType: '1',                 // scheduled = 1, instant = 2
    assetId: '2',
    amount: '200000000',               // gross, 8-decimal atoms
    deadline: String(deadlineUnixMs),
  },
}
```

Three values are encoded differently in the request body and in the signature. Derive both from one object, because a mismatch reports only `withdrawal signer does not match user`, never which field is wrong.

| Field           | Request body                | Signed message           |
| --------------- | --------------------------- | ------------------------ |
| `withdraw_type` | `"scheduled"` / `"instant"` | `"1"` / `"2"` as `uint8` |
| `amount`        | Decimal atom string         | Same value as `uint256`  |
| `deadline`      | Milliseconds                | The same milliseconds    |

```bash
curl -sS "$POOL_API_URL/api/v3/earn" \
  -H 'content-type: application/json' \
  -d '{
    "type": "createWithdrawal",
    "user_address": "0xbf381e…",
    "user_nonce": 3,
    "withdraw_type": "scheduled",
    "asset_id": 2,
    "amount": "200000000",
    "deadline_unix_ms": 1786700000000,
    "user_signature": "0xa1bc51d1…52e9a0561c"
  }'
```

`user_signature` is the wallet's raw 65-byte signature, `0x` plus 130 hex characters.

`deadline_unix_ms` must be in the future when the request arrives. A request carrying a past deadline returns `deadline_unix_ms must be in the future`. Set it five minutes ahead.

A successful response returns the created withdrawal record, in the shape described below.

## 4. Follow the status

```bash
curl -sS "$POOL_API_URL/api/v3/earn" \
  -H 'content-type: application/json' \
  -d '{"type":"withdrawals","user_address":"0xbf381e…","limit":20}'
```

```json
{
  "code": 0,
  "data": {
    "items": [
      {
        "id": 3,
        "operation_id": "withdraw:0xbf381e…:1",
        "user_address": "0xbf381e…",
        "user_nonce": 1,
        "withdraw_type": "scheduled",
        "asset_id": 2,
        "gross_amount": "200000000",
        "user_amount": "200000000",
        "fee_amount": "0",
        "fee_bps_snapshot": 0,
        "request_deadline_unix_ms": 1786607858484,
        "claimable_at_unix_ms": 1786607271588,
        "status": "claimed",
        "created_at_unix_ms": 1786607261588,
        "core_tx_hash": "0xe838b87fa9b1bafabc6c6c39f0f5c0199d307e4cb86767c5e751f962d135f3fc",
        "claimed_at_unix_ms": 1786607281588,
        "cancelled_at_unix_ms": null,
        "completed_at_unix_ms": 1786607281588
      }
    ],
    "next_before_id": null
  },
  "message": "success"
}
```

`operation_id` is `withdraw:<user_address>:<user_nonce>`, and it is the handle for claim and cancel.

Each type walks its own path:

```
scheduled   queued  → authorizing → authorized → transferring → claimed
instant     created → authorizing → authorized → transferring → completed
```

| Status          | Meaning                                                    |
| --------------- | ---------------------------------------------------------- |
| `created`       | An instant withdrawal has been accepted                     |
| `queued`        | A scheduled withdrawal is waiting for its claim window      |
| `authorizing`   | Authorization in progress                                   |
| `authorized`    | Authorized, payout not yet submitted                        |
| `transferring`  | The payout transfer has been submitted                      |
| `claimed`       | Terminal. A scheduled withdrawal was claimed and paid       |
| `completed`     | Terminal. An instant withdrawal was paid                    |
| `cancelled`     | Terminal. A scheduled withdrawal was cancelled              |
| `manual_review` | The payout failed and is held for manual intervention       |

These nine are also the accepted values for the `status` filter; any other value is rejected.

{% hint style="warning" %}
**A failed payout does not get its own status.** The status stops advancing and `failure_code` with `failure_message` appear on the record. A withdrawal sitting in a non-terminal state with `failure_code` set is stuck, not in flight, and `manual_review` never resolves on its own. Treat both as terminal for your own polling and report them.
{% endhint %}

`claimed_at_unix_ms`, `cancelled_at_unix_ms` and `completed_at_unix_ms` are always present, and `null` until they happen. They are not mutually exclusive: claiming a scheduled withdrawal sets `claimed_at` and `completed_at` to the same value.

Two payout transaction hashes appear once the transfers are submitted, and are **absent as keys** until then:

| Field              | Content                       | Appears on                              |
| ------------------ | ----------------------------- | --------------------------------------- |
| `core_tx_hash`     | The transfer to the user      | Every withdrawal                        |
| `fee_core_tx_hash` | The transfer to the fee wallet | Instant withdrawals with a non-zero fee |

{% hint style="info" %}
Both hashes come from a join the Pool performs only for the list query. `{type:"withdrawals"}` and `account.active_withdrawal` carry them; the single-record `{type:"withdrawal"}` query does not.

The list query also has **no `operation_id` parameter**. Sending one does not filter and does not error — you get an unfiltered page back. Fetch the list and match on `operation_id` in your own code, narrowing with `status` or `asset_id` if the history is long.
{% endhint %}

## 5. Claim a scheduled withdrawal

Only `scheduled` withdrawals need this step, and only inside their window:

```
withdraw_type === 'scheduled'
&& status === 'queued'
&& claimable_at_unix_ms != null
&& Date.now() >= claimable_at_unix_ms
```

**Read `claimable_at_unix_ms`; do not recompute it.** The Pool sets it from server time when the withdrawal is created, so `created_at_unix_ms + withdraw_pending_seconds` does not reproduce it and fails at the boundary.

The signature is the same domain with a different primary type. `operationId` is signed as the **keccak256 hash** of the trimmed string, typed `bytes32`, while the request body carries the plain string. This is the most common mistake on this endpoint.

```ts
const typedData = {
  domain,
  primaryType: 'ClaimWithdrawal',
  types: {
    EIP712Domain: [
      { name: 'name',              type: 'string'  },
      { name: 'version',           type: 'string'  },
      { name: 'verifyingContract', type: 'address' },
    ],
    ClaimWithdrawal: [
      { name: 'user',        type: 'address' },
      { name: 'operationId', type: 'bytes32' },
      { name: 'deadline',    type: 'uint256' },
    ],
  },
  message: {
    user: userAddress,
    operationId: keccak256(toUtf8Bytes(operationId.trim())),
    deadline: String(deadlineUnixMs),
  },
}
```

```bash
curl -sS "$POOL_API_URL/api/v3/earn" \
  -H 'content-type: application/json' \
  -d '{
    "type": "claimWithdrawal",
    "user_address": "0xbf381e…",
    "operation_id": "withdraw:0xbf381e…:1",
    "deadline_unix_ms": 1786700000000,
    "user_signature": "0x…"
  }'
```

{% hint style="danger" %}
**After a network timeout, resend the identical request body. Do not re-sign.**

The first claim binds its signature digest to the withdrawal. A byte-identical resend short-circuits and returns the same result; a fresh signature with a new deadline is rejected against the bound digest. Keep the exact body you sent.

The retry window is the deadline you signed. Past it, even the identical body fails with `deadline_unix_ms must be in the future`, and the original claim may already have succeeded. Poll `withdrawals` before concluding anything.
{% endhint %}

## 6. Cancel a scheduled withdrawal

A scheduled withdrawal can be cancelled only while all of the following hold:

```
withdraw_type === 'scheduled'
&& status === 'queued'
&& claimable_at_unix_ms != null
&& Date.now() < claimable_at_unix_ms
```

At most one of claim and cancel is available at any moment. When `withdraw_pending_seconds` is `0` the withdrawal is claimable immediately and can never be cancelled.

The request is identical to claim with `primaryType` changed to `CancelWithdrawal`:

```bash
curl -sS "$POOL_API_URL/api/v3/earn" \
  -H 'content-type: application/json' \
  -d '{
    "type": "cancelWithdrawal",
    "user_address": "0xbf381e…",
    "operation_id": "withdraw:0xbf381e…:1",
    "deadline_unix_ms": 1786700000000,
    "user_signature": "0x…"
  }'
```

The status becomes `cancelled` and the gross amount returns from `in_queue_balance` to `earn_balance`, where it earns again.

## When withdrawals are paused

`withdrawal_paused` in `config` blocks **creating and claiming**. Cancelling still works within its normal window, so a scheduled withdrawal that has not yet become claimable can still be reversed during a pause.

During a pause, an instant withdrawal and any scheduled withdrawal past its cancel window stay in their current status until the pause lifts.

## What can go wrong

| Message                                    | Cause                                                                     | What to do                                              |
| ------------------------------------------ | ------------------------------------------------------------------------- | -------------------------------------------------------- |
| `user already has an active withdrawal`    | One is already in flight for this address                                  | Check `active_withdraw_operation_id` before signing      |
| `withdrawal signer does not match user`    | The typed data does not match, so a different address was recovered        | Check the `uint8` type code, the `bytes32` `operationId`, the millisecond `deadline`, and that no `chainId` reached the domain |
| `invalid signature recovery id`            | The 65-byte signature's `v` byte is not `0`, `1`, `27` or `28`             | Post the wallet's raw `r‖s‖v`; do not reorder or re-encode it |
| `deadline_unix_ms must be in the future`   | The deadline has passed, including on a delayed claim retry                | For a create, sign a new one; for a claim, poll `withdrawals` first |
| `withdrawal is not claimable`              | Before `claimable_at_unix_ms`, or the status is no longer `queued`         | Compare against `claimable_at_unix_ms`                   |
| `withdrawal is not cancelable`             | Past `claimable_at_unix_ms`, or already claimed                            | Cancelling is no longer possible; claim instead          |
| `insufficient earn balance`                | The gross amount exceeds `earn_balance`                                    | A withdrawal already in flight is not in `earn_balance`   |
| `withdrawal amount is below minimum`       | Gross below `min_withdraw_amount`                                          | Re-read `config`; `"0"` means no minimum                  |
| `withdrawal amount exceeds maximum`        | Gross above `max_single_withdraw_amount`                                   | Re-read `config`; `"0"` means no maximum                  |
| `withdrawals are paused`                   | `withdrawal_paused` is true                                                | Cancelling is still available in its window              |
| `withdrawal type is disabled for asset`    | The asset has that type turned off                                         | Check `scheduled_withdraw_enabled` and `instant_withdraw_enabled` |
| `withdrawal not found`                     | No such record, **or** it belongs to another address                       | Both cases return the same response                      |

## Next steps

{% content-ref url="reference.md" %}
[reference.md](reference.md)
{% endcontent-ref %}

{% content-ref url="yield.md" %}
[yield.md](yield.md)
{% endcontent-ref %}
