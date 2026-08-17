---
description: The Native Core Pool contract — every request type, its parameters, its response fields, and the error codes.
---

# Reference

Every operation is `POST /api/v3/earn` on `https://api-ui.native.org`, with a `type` field naming it.

```bash
POOL_API_URL=https://api-ui.native.org
```

The HTTP status is always 200. Success is `code: 0`; a failure carries `code` and `message` and no `data` key. See [Error codes](#error-codes).

{% openapi src="../../.gitbook/assets/native-core-pool-api.json" path="/api/v3/earn" method="post" %}
[native-core-pool-api.json](../../.gitbook/assets/native-core-pool-api.json)
{% endopenapi %}

Pick an operation from the request examples above to send it against mainnet. Every operation shares one path and one method, so the `type` field is what selects it.

| `type`             | Purpose                                       | Signed |
| ------------------ | --------------------------------------------- | ------ |
| `config`           | Global settings and the asset list             | No     |
| `account`          | Balances, nonce, and the active withdrawal     | No     |
| `deposits`         | Credited and rejected deposits                 | No     |
| `yieldHistory`     | Applied yield distributions                    | No     |
| `withdrawals`      | Withdrawal records, with payout hashes         | No     |
| `withdrawal`       | One withdrawal by `operation_id`               | No     |
| `createWithdrawal` | Create a withdrawal                            | Yes    |
| `claimWithdrawal`  | Claim a matured scheduled withdrawal           | Yes    |
| `cancelWithdrawal` | Cancel a scheduled withdrawal                  | Yes    |

## config

No parameters.

```bash
curl -sS "$POOL_API_URL/api/v3/earn" -H 'content-type: application/json' -d '{"type":"config"}'
```

| Field                | Type    | Meaning                                                             |
| -------------------- | ------- | ------------------------------------------------------------------- |
| `native_chain_id`    | number  | The Native chain id to sign with, `696969` on mainnet                |
| `vault_address`      | string  | Recipient of a Core-internal deposit, and the EIP-712 `verifyingContract` |
| `fee_wallet_address` | string  | Where instant-withdrawal fees are paid                               |
| `snapshot_time_utc`  | string  | Daily snapshot time, `HH:MM:SS.mmm`                                  |
| `withdrawal_paused`  | boolean | Blocks creating and claiming; cancelling is unaffected               |
| `eip712_domain`      | object  | `name`, `version`, `verifyingContract`. Pass through unchanged        |
| `assets`             | array   | One entry per asset the Pool supports                                |

Each `assets[]` entry:

| Field                        | Type    | Meaning                                                   |
| ---------------------------- | ------- | --------------------------------------------------------- |
| `asset_id`                   | number  | The id used by every other request                         |
| `symbol`                     | string  | Display symbol                                             |
| `balance_decimals`           | number  | Precision of every Pool amount for this asset, `8`         |
| `deposit_enabled`            | boolean | Whether deposits are accepted                              |
| `scheduled_withdraw_enabled` | boolean | Whether scheduled withdrawals are accepted                 |
| `instant_withdraw_enabled`   | boolean | Whether instant withdrawals are accepted                   |
| `withdraw_pending_seconds`   | number  | Scheduled wait before claiming, and the cancel window      |
| `instant_fee_bps`            | number  | Instant fee in basis points, applied to the gross          |
| `min_withdraw_amount`        | string  | Minimum gross. `"0"` means no minimum                      |
| `max_single_withdraw_amount` | string  | Maximum gross. `"0"` means no maximum                      |
| `projected_apy`              | string  | Annualized ratio, `"0.12"` is 12%. See [Yield](yield.md#reported-rates) |
| `realtime_tvl_amount`        | string  | Total earning balance for the asset, in its atoms          |

`projected_apy` and `realtime_tvl_amount` are omitted when the backend has no value for them.

## account

| Parameter      | Required | Notes                     |
| -------------- | -------- | ------------------------- |
| `user_address` | Yes      | `0x` plus 40 hex characters |

| Field                          | Type          | Meaning                                                    |
| ------------------------------ | ------------- | ---------------------------------------------------------- |
| `user_address`                 | string        | The queried address                                         |
| `next_user_nonce`              | number        | The `user_nonce` to sign the next `createWithdrawal` with   |
| `active_withdraw_operation_id` | string        | The in-flight withdrawal, `""` when there is none           |
| `active_withdrawal`            | object / null | The same withdrawal in full, when it can be loaded          |
| `balances`                     | array         | One entry per asset held                                    |

Use `active_withdraw_operation_id` as the gate. `active_withdrawal` is a convenience view and can be `null` while a withdrawal is active.

Each `balances[]` entry:

| Field                     | Meaning                                                   |
| ------------------------- | --------------------------------------------------------- |
| `asset_id`                | The asset                                                  |
| `earn_balance`            | Available and earning                                      |
| `in_queue_balance`        | Gross amount of a pending scheduled withdrawal             |
| `withdraw_locked_balance` | Gross amount of an instant withdrawal in flight            |
| `lifetime_deposit`        | Total ever deposited                                       |
| `lifetime_yield`          | Total yield ever credited, for this asset only             |
| `lifetime_withdraw`       | Total ever withdrawn                                       |

The three balance buckets are disjoint. Only `earn_balance` earns.

## deposits

| Parameter      | Required | Default | Notes                                       |
| -------------- | -------- | ------- | ------------------------------------------- |
| `user_address` | Yes      | —       |                                             |
| `asset_id`     | No       | all     |                                             |
| `before_id`    | No       | —       | Cursor                                      |
| `limit`        | No       | `50`    | Over `200` the request fails                |

| Field                    | Meaning                                                             |
| ------------------------ | ------------------------------------------------------------------- |
| `id`                     | Row id, also the pagination cursor                                   |
| `operation_id`           | `deposit:<src_chain_id>:<vault>:<nonce>` or `direct-transfer:<native_chain_id>:<height>:<tx_index>:<event_index>` |
| `deposit_type`           | `bridge_deposit` or `direct_transfer`                                |
| `asset_id`               | The credited asset                                                   |
| `amount`                 | Credited amount, in `balance_decimals`                               |
| `status`                 | `credited` or `rejected`                                             |
| `core_height`            | Core block height of the credit                                      |
| `core_event_timestamp_ms`| Core event time                                                      |
| `src_chain_id`           | Source EVM chain. **Key absent** on `direct_transfer`                |
| `src_tx_hash`            | Source EVM transaction. **Key absent** on `direct_transfer`          |
| `core_tx_hash`           | Core transaction. **Key absent** on `direct_transfer`                |

Nothing in flight is listed. See [Deposit](deposit.md#reading-the-deposit-list).

## yieldHistory

Parameters match `deposits`.

| Field                | Meaning                                                    |
| -------------------- | ---------------------------------------------------------- |
| `id`                 | Row id, also the pagination cursor                          |
| `distribution_id`    | Opaque grouping key. Contains no date                       |
| `asset_id`           | The asset                                                   |
| `snapshot_balance`   | The balance the share was computed from                     |
| `yield_amount`       | The amount credited                                         |
| `applied_at_unix_ms` | When it was credited                                        |

Pass `asset_id` to sum that asset's `yield_amount` rows and reconcile the total against its `lifetime_yield`.

## withdrawals

| Parameter      | Required | Default | Notes                                                |
| -------------- | -------- | ------- | ---------------------------------------------------- |
| `user_address` | Yes      | —       |                                                      |
| `asset_id`     | No       | all     |                                                      |
| `status`       | No       | all     | One of the nine statuses below; any other value fails |
| `before_id`    | No       | —       | Cursor                                               |
| `limit`        | No       | `50`    | Over `200` the request fails                         |

`withdrawals` has **no `operation_id` parameter**. Sending one is ignored without an error, and the response is an unfiltered page.

| Field                      | Meaning                                                            |
| -------------------------- | ------------------------------------------------------------------ |
| `id`                       | Row id, also the pagination cursor                                  |
| `operation_id`             | `withdraw:<user_address>:<user_nonce>`                              |
| `user_address`             | Owner                                                               |
| `user_nonce`               | The nonce signed at creation                                        |
| `withdraw_type`            | `scheduled` or `instant`                                            |
| `asset_id`                 | The asset                                                           |
| `gross_amount`             | The signed amount, before fee                                       |
| `user_amount`              | What the user receives, `gross_amount − fee_amount`                 |
| `fee_amount`               | Fee charged, `0` for scheduled                                      |
| `fee_bps_snapshot`         | The rate applied, recorded at creation                              |
| `request_deadline_unix_ms` | The deadline signed at creation                                     |
| `claimable_at_unix_ms`     | When claiming opens. `null` for instant                             |
| `status`                   | See below                                                           |
| `created_at_unix_ms`       | Creation time                                                       |
| `core_tx_hash`             | Payout transfer to the user. **Key absent** until submitted         |
| `fee_core_tx_hash`         | Payout transfer to the fee wallet. **Key absent** unless a fee was charged |
| `failure_code`             | Payout failure. **Key absent** when there is none                   |
| `failure_message`          | Payout failure detail. **Key absent** when there is none            |
| `claimed_at_unix_ms`       | Always present, `null` until claimed                                |
| `cancelled_at_unix_ms`     | Always present, `null` until cancelled                              |
| `completed_at_unix_ms`     | Always present, `null` until paid                                   |

The nine statuses:

| Status          | Terminal | Meaning                                            |
| --------------- | -------- | -------------------------------------------------- |
| `created`       | No       | An instant withdrawal was accepted                  |
| `queued`        | No       | A scheduled withdrawal is waiting                   |
| `authorizing`   | No       | Authorization in progress                           |
| `authorized`    | No       | Authorized, payout not yet submitted                |
| `transferring`  | No       | Payout transfer submitted                           |
| `claimed`       | Yes      | Scheduled withdrawal claimed and paid               |
| `completed`     | Yes      | Instant withdrawal paid                             |
| `cancelled`     | Yes      | Scheduled withdrawal cancelled                      |
| `manual_review` | No       | Payout failed and is held for manual intervention   |

`manual_review` does not resolve on its own. Neither does a non-terminal record carrying `failure_code`.

## withdrawal

| Parameter      | Required | Notes                                    |
| -------------- | -------- | ---------------------------------------- |
| `user_address` | Yes      | Checked against the record's owner        |
| `operation_id` | Yes      |                                          |

Returns one record in the shape above, **without** `core_tx_hash` or `fee_core_tx_hash`. Use `withdrawals` when you need the payout hashes.

A record belonging to another address returns the same response as one that does not exist.

## createWithdrawal

| Parameter          | Type   | Notes                                                    |
| ------------------ | ------ | -------------------------------------------------------- |
| `user_address`     | string | Must match the recovered signer                           |
| `user_nonce`       | number | `next_user_nonce` from `account`                          |
| `withdraw_type`    | string | `"scheduled"` or `"instant"`. Signed as `1` or `2`        |
| `asset_id`         | number |                                                           |
| `amount`           | string | Gross, in 8-decimal atoms, digits only                    |
| `deadline_unix_ms` | number | Must be in the future when the request arrives            |
| `user_signature`   | string | `0x` plus 130 hex characters                              |

Sign this action as `CreateWithdrawal`. Typed data and the encoding differences are in [Withdraw](withdraw.md#3-sign-and-submit).

Returns the created withdrawal record.

## claimWithdrawal and cancelWithdrawal

Both take the same four fields.

| Parameter          | Type   | Notes                                                   |
| ------------------ | ------ | ------------------------------------------------------- |
| `user_address`     | string | Must match the recovered signer                          |
| `operation_id`     | string | The plain string. Signed as its keccak256 hash           |
| `deadline_unix_ms` | number | Must be in the future                                    |
| `user_signature`   | string | `0x` plus 130 hex characters                             |

Sign a claim as `ClaimWithdrawal` and a cancel as `CancelWithdrawal`. The first claim attempt binds its digest, so resend the identical bytes on every retry and never sign a new claim for the same `operation_id`. See [Withdraw](withdraw.md#5-claim-a-scheduled-withdrawal).

## Pagination

`deposits`, `yieldHistory` and `withdrawals` share one cursor scheme.

```json
{ "type": "deposits", "user_address": "0x…", "before_id": 9, "limit": 50 }
```

`limit` defaults to `50`. **Above `200` the request fails** with `limit exceeds max 200`; it is not clamped down.

`next_before_id` is `null` on the last page. It is set whenever a page comes back exactly `limit` long, so the final full page still carries a cursor and one extra request is needed to confirm the end.

## Error codes

| `code`   | Meaning                                | Handling                                        |
| -------- | -------------------------------------- | ----------------------------------------------- |
| `0`      | Success                                 |                                                 |
| `131002` | Backend unavailable                     | Retry. Never read it as an empty result          |
| `131004` | Invalid parameter or state not satisfied | Read `message`                                  |
| `131006` | Upstream timeout                        | Retry                                            |
| `201005` | Rate limit exceeded                     | Wait for the next second and retry                |

These five are the complete set for `/api/v3/earn`. The Core-internal deposit route uses Native Core's `/trade`, which has its own status codes and error shape; see [Error Responses](../native-core/error-responses.md).

### Common `131004` messages

| Message                                             | Cause                                                        |
| --------------------------------------------------- | ------------------------------------------------------------ |
| `unknown type: <value>`                             | Unrecognized `type`                                           |
| `user_address is required`                          | Missing parameter                                             |
| `user_address must be 0x + 40 hex chars`            | Malformed address                                             |
| `limit exceeds max 200`                             | `limit` above the cap                                         |
| `user_signature must be 0x + 130 hex chars (65 bytes)` | Wrong signature length                                     |
| `deadline_unix_ms must be in the future`            | The deadline has passed                                       |
| `withdrawal signer does not match user`             | Typed data mismatch; a different address was recovered        |
| `invalid signature recovery id`                     | The signature's `v` byte is not `0`, `1`, `27` or `28`        |
| `user already has an active withdrawal`             | One is already in flight for this address                     |
| `insufficient earn balance`                         | Gross amount exceeds `earn_balance`                           |
| `withdrawal amount is below minimum`                | Below `min_withdraw_amount`                                   |
| `withdrawal amount exceeds maximum`                 | Above `max_single_withdraw_amount`                            |
| `withdrawal is not claimable`                       | Before `claimable_at_unix_ms`, or no longer `queued`          |
| `withdrawal is not cancelable`                      | Past `claimable_at_unix_ms`, or already claimed               |
| `withdrawals are paused`                            | `withdrawal_paused` is true                                   |
| `withdrawal type is disabled for asset`             | That type is turned off for the asset                         |
| `withdrawal not found`                              | No such record, or it belongs to another address              |

Every response carries a `trace_id` header. Include it when reporting a problem.
