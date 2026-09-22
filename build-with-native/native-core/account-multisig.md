---
description: Configure an account-level multisig quorum over your account's owner-signed actions, and submit quorum-signed requests.
---

# Account Multisig

An **account multisig** puts a signer quorum in front of your account's owner-signed actions. Once it is active, moving value or changing who can trade for you takes `threshold`-of-`signers` approvals instead of one owner signature.

It does **not** cover trading. Trading actions continue to be signed by a single [API wallet](nonces-and-api-wallets.md#api-wallets) key.

{% hint style="danger" %}
**Read this before enabling a multisig.** Two things change, and neither can be undone:

1. **The owner key stops being a trading key.** While your account holds an **Active** configuration, every `order`, `cancel`, `cancelAll`, `modify`, and `batch` must be signed by an API wallet; a direct owner-signed order is rejected with `AccountMultisigAgentRequired`. **Approve an API wallet first**, or you will be unable to trade until you do.
2. **A configuration cannot be removed.** You can replace the signer set and threshold at any time, but the account cannot be returned to plain single-owner control, and the owner key cannot be restored as a direct trading key. The quorum also **must not** include your own account key — see [Signer-set restriction](#signer-set-restriction).
{% endhint %}

## What a quorum can sign

| | |
| --- | --- |
| **Covered** — needs the quorum | `transfer` · `withdraw` · `settle` · `repay` · `approveAgent` · `revokeAgent` · `activateFor` · `setAccountMultisig` |
| **Not covered** — API wallet, single signature | `order` · `cancel` · `cancelAll` · `modify` · `batch` |

Trading actions can never be quorum-signed. A request that carries `auth_account` together with a trading action is rejected with `AccountAuthNotAllowedForAction`.

## Setup order

Enabling a multisig locks the owner key out of trading, so the sequence matters:

1. **Approve an API wallet** with an owner-signed `approveAgent`, and confirm it can place and cancel an order. Do this *before* step 3 — afterwards the owner key can no longer sign trading actions, and `approveAgent` itself will need the quorum.
2. **Fund the account** with the creation fee — 1,000 USDC or 1,000 USDT — unless a sponsor is paying it.
3. **Submit `setAccountMultisig`**, owner-signed, with `creation_fee`.
4. **Read `accountMultisig`** and record `policy_epoch`. Every quorum-signed request from here on carries it, and it advances on each configuration change, so re-read it after any `setAccountMultisig`.

## Calling the API

`setAccountMultisig` is submitted to [`POST /trade`](post-trade.md) like any other write, and the configuration is read from [`POST /info`](post-info.md). Examples below use mainnet, `API_URL=https://api.native.org`; for testnet, substitute `https://api-test.native.org` and sign with the testnet chain id given on [Transaction Signing](transaction-signing.md).

```sh
curl -sS -X POST "$API_URL/trade" \
  -H 'content-type: application/json' \
  -H 'x-trace-id: client-trace-001' \
  -d '{ "action": { "type": "setAccountMultisig", ... }, "nonce": "...", "auth_scheme": "eip712", "signature": "0x..." }'
```

`POST /trade` is **synchronous** and answers with the standard envelope — `submission_status` of `accepted`, `rejected`, or `timeout`, plus `tx_hash` and an `error.code`. The full contract, including what `timeout` means for whether a transaction can still land, is on [POST /trade](post-trade.md). Every code in [Errors](#errors) below arrives as that `error.code`.

`nonce` is a Unix millisecond timestamp and is validated per authority; see [Nonces & API Wallets](nonces-and-api-wallets.md). `expires_after_ms` is optional on every request here, except a [sponsored](#creation-fee) creation, which requires it.

## Reading your configuration

```json
{ "type": "accountMultisig", "user": "0xYourOwnerAddress" }
```

```json
{
  "query_height": 198543355,
  "app_hash": "0x…",
  "owner": "0xYourOwnerAddress",
  "found": true,
  "account_index": 93,
  "enabled": true,
  "role": null,
  "threshold": 2,
  "signers": ["0x…", "0x…", "0x…"],
  "policy_epoch": "7"
}
```

`found` reports whether the **account** exists, not whether it has a multisig row — read `enabled` to decide that. `role` is non-null only for protocol-operated accounts; for your own account it is always `null`. `policy_epoch` is the value every quorum-signed request must carry; it advances on each configuration change.

## `setAccountMultisig`

Creates the account's first configuration, or replaces the current one. The action is the same, but the envelope is not: the **first creation is owner-signed** with `signature`, as below; once a configuration is Active, a replacement is a covered action like any other and must be [quorum-signed](#submitting-a-quorum-signed-request) — an owner-signed replacement is rejected with `AccountMultisigRequired`. A first-ever creation additionally carries `creation_fee`.

```json
{
  "action": {
    "type": "setAccountMultisig",
    "threshold": 2,
    "signers": ["0x1111…", "0x2222…", "0x3333…"],
    "creation_fee": {
      "asset_id": "1",
      "amount": "100000000000",
      "payment": { "type": "selfPaid" }
    },
    "cloid": "0x000102030405060708090a0b0c0d0e0f"
  },
  "nonce": "1757203200000",
  "auth_scheme": "eip712",
  "signature": "0x…"
}
```

| Field | Type | Notes |
| --- | --- | --- |
| `threshold` | number | Approvals required. `1` … `signers.length`. `0` is rejected. |
| `signers` | string\[] | 1–**32** addresses. Must be **unique, non-zero, and sorted ascending** — an unsorted array is rejected even if the set is correct. |
| `creation_fee` | object | **Only** on a first-ever creation. Omit it when replacing an existing configuration. |
| `cloid` | string | **Required.** 16-byte hex. Used only for `txStatusByCloid`; it is not an idempotency key. |

Field names are snake\_case and the payload is **strict** — an unrecognised field fails the request rather than being ignored. `asset_id` and `amount` are **decimal strings**, not JSON numbers, and `amount` is in raw atoms with no display decimals.

### Creation fee

Charged once, on the first configuration only.

| `asset_id` | Asset | `amount` (atoms) | Value |
| --- | --- | --- | --- |
| `"1"` | USDC | `"100000000000"` | 1,000 USDC |
| `"2"` | USDT | `"100000000000"` | 1,000 USDT |

The amount must match one of the rows above exactly — a different value is rejected with `InvalidAccountMultisigCreationFee`, not rounded or topped up.

**`selfPaid`** — the account pays its own fee from its available balance:

```jsonc
"payment": { "type": "selfPaid" }
```

**`sponsored`** — another account pays it. The sponsor signs a separate EIP-712 `AccountMultisigSponsor` message; its 65-byte signature rides inside the action:

```jsonc
"payment": {
  "type": "sponsored",
  "payer": "0xSponsorOwnerAddress",
  "payer_policy_epoch": "3",
  "signature": "0x…"
}
```

The sponsor signs its own EIP-712 message, under the quorum-signed domain (`version: "2"` — see [EIP-712 typed data](#eip-712-typed-data)):

```
AccountMultisigSponsor(uint256 nativeChainId,address targetAccount,uint256 payerPolicyEpoch,uint256 feeAssetId,uint256 feeAmountAtoms,uint256 expiresAfterMs)
```

`targetAccount` is the account being configured. `expiresAfterMs` is the value the request's `expires_after_ms` field carries — which is why that field is **required** on a sponsored creation; the sponsor's authorization expires with it. The signature is verified by recovering the signer and requiring it to equal `payer`. It is **not** part of the outer `SetAccountMultisig` digest — signing the outer message does not cover it, and it must be produced separately by the sponsor.

### Signer-set restriction

A configuration you submit **must not list your own account's key** among `signers`. This applies to both a first-ever creation and any replacement set. Violations are rejected with `InvalidAccountMultisigConfig` before the nonce is consumed.

## Submitting a quorum-signed request

Once a configuration is Active, the covered actions are submitted with an **account-auth** request: drop `signature`, and send the collected approvals in `signatures` along with the two account-auth fields.

```json
{
  "action": {
    "type": "withdraw",
    "asset_id": "1",
    "amount": "100000000",
    "dst_chain_id": "1",
    "dst_address": "0xDestinationAddress",
    "withdraw_nonce": "1",
    "cloid": "0x000102030405060708090a0b0c0d0e0f"
  },
  "nonce": "1757203200000",
  "auth_scheme": "eip712",
  "auth_account": "0xYourOwnerAddress",
  "policy_epoch": "7",
  "signatures": ["0x…", "0x…"]
}
```

| Field | Notes |
| --- | --- |
| `auth_account` | The account the quorum acts for. Must be sent **with** `policy_epoch`. |
| `policy_epoch` | From the `accountMultisig` read. A stale value is rejected with `AccountMultisigEpochMismatch`. |
| `signatures` | Array of 65-byte hex signatures, at least `threshold` of them, each from a distinct configured signer. **Order matters** — see below. |
| `agent_epoch` | **Must be absent.** Account-auth and agent-auth are mutually exclusive. |

Every signature covers the same [quorum-signed digest](#eip-712-typed-data), so the signers can sign independently and in any sequence — but the **array you submit must be ordered so that the recovered addresses ascend**. Collect the approvals in any order, then sort them by recovered signer address before submitting. An out-of-order array is rejected with `DecodeMultisigProofSignaturesNotSorted` even when every signature is individually valid, and the same signer appearing twice is rejected with `DecodeMultisigProofDuplicateRecoveredSigner` rather than counting as two approvals.

## EIP-712 typed data

Which scheme you sign follows the envelope, not the action.

**Owner-signed (the first creation)** — the standard **v4** scheme: the domain, the six common fields, and the encoding rules are exactly as specified on [Transaction Signing](transaction-signing.md#eip-712-signing-auth_scheme-eip712), with `authKind: 1` and `authScope: 0`.

**Quorum-signed (every account-auth request)** — a v5 variant of that scheme. Every approval in a `signatures` array must be computed this way, and it differs from the v4 scheme in exactly three places:

* The domain `version` is **`"2"`**. Everything else about the domain — the name, the zero `verifyingContract`, the absence of `chainId` — is unchanged.
* The six common fields become **seven**: `authScope` is replaced by `address authAccount, uint256 policyEpoch`, signed with the same values the envelope's `auth_account` and `policy_epoch` fields carry:

  ```
  uint256 nativeChainId,uint256 authKind,address authAccount,uint256 policyEpoch,uint256 nonce,bool expiresAfterMsPresent,uint256 expiresAfterMs
  ```

* `authKind` is **`2`** — including an array carrying a single approval.

The action tail after the common fields is identical under both schemes — for this action and for every other covered action. Written out in full, the **v4** primary type for `setAccountMultisig` is below; for the **v5** form, replace the six common fields at its head with the seven above and leave everything from `uint256 threshold` onward unchanged.

```
SetAccountMultisig(uint256 nativeChainId,uint256 authKind,uint256 authScope,uint256 nonce,bool expiresAfterMsPresent,uint256 expiresAfterMs,uint256 threshold,address[] signers,uint256 feeMode,uint256 feeAssetId,uint256 feeAmountAtoms,address feePayer,uint256 feePayerPolicyEpoch,bytes16 cloid)
```

Within that tail the fee is **flattened**, not nested. `feeMode` is the discriminator — `0` no fee (a replacement), `1` `selfPaid`, `2` `sponsored` — and every unused fee field is zero-valued: address zero for `feePayer`, `0` for the three numerics.

## Errors

Every code below is a fixed `CamelCase` string returned verbatim as `error.code`. Match on the exact string; the codes are grouped here by how far the request got, because that determines what you do next.

**Request-level** — the request is malformed and never reaches execution. `submission_status` is `rejected` and no `tx_hash` is returned:

| Error | Meaning |
| --- | --- |
| `AuthAccountRequiresPolicyEpoch` | `auth_account` sent without `policy_epoch`. |
| `PolicyEpochRequiresAuthAccount` | `policy_epoch` sent without `auth_account`. |
| `AgentEpochNotAllowedWithAuthAccount` | Both agent-auth and account-auth fields present. |
| `SignaturesNotAllowedForAction` | `signatures` sent without `auth_account`. This is the usual shape of a forgotten `auth_account`. |
| `AccountAuthRequiresEip712` | `auth_account` sent without `auth_scheme: "eip712"`. |
| `AccountAuthNotAllowedForAction` | Account-auth used with an action a quorum cannot sign — a trading action, most often. |
| `AccountAuthNotSupported` | Account multisig is not available on the network you addressed. |
| `InvalidSigner` / `InvalidPayer` / `InvalidSponsorSignature` | Malformed hex, or a wrong length, in `signers`, `payer`, or the sponsor signature. |
| `DecodeMultisigProofSignaturesNotSorted` | `signatures` is not ordered by ascending recovered signer address. |
| `DecodeMultisigProofDuplicateRecoveredSigner` | The same signer appears twice in `signatures`. |
| `InvalidSignaturesLen` | `signatures` is empty, or holds more than 32 entries. |
| `DecodeMultisigProofRecoveryFailed` | A signature in the array does not recover to any address. |
| `LegacySignatureNotAccepted` | Submitted without `auth_scheme:"eip712"` — this action accepts no other scheme. |
| `InvalidAuthAccount` | `auth_account` is not a well-formed address. |
| `AmbiguousAuthFields` | Both `signature` and `signatures` were sent, or neither — a quorum request carries only `signatures`. |

**Execution-level** — the request was accepted and then rejected on chain. These split at the **nonce boundary**, and the side decides whether you can retry with the same nonce.

*Before the nonce is consumed* — the nonce is untouched, so correct the request and resubmit it unchanged:

| Error | Meaning |
| --- | --- |
| `AccountMultisigAgentRequired` | The owner key submitted a trading action while a configuration is Active. Use an API wallet. |
| `AccountMultisigEpochMismatch` | `policy_epoch` is stale. Re-read `accountMultisig` and re-sign. |
| `AccountMultisigNotConfigured` | The account has no configuration. |
| `AccountMultisigNotActive` | A configuration exists but is not Active. |
| `AccountMultisigRequired` | The account's lifecycle requires a quorum, but a single owner signature was sent. |
| `UnauthorizedMultisig` | Fewer than `threshold` valid signatures, or a signature from an address outside the set. |
| `InvalidAccountMultisigConfig` | Structural problem — empty, unsorted, duplicated, zero address, threshold out of range, or the set includes the account's own key. |
| `FeatureDisabled` | The network is below the account-multisig activation height, so account-auth frames are not yet accepted. |

*After the nonce is consumed* — the nonce is spent. Fix the cause and resubmit with a **fresh** nonce:

| Error | Meaning |
| --- | --- |
| `AccountFrozen` | The account is frozen and cannot reconfigure until an admin unfreezes it. A frozen sponsor is rejected the same way. |
| `AccountMultisigCreationFeeRequired` | First-ever creation submitted without `creation_fee`. |
| `UnexpectedAccountMultisigCreationFee` | `creation_fee` sent on a **replacement**. Only the first-ever creation carries one. |
| `InvalidAccountMultisigCreationFee` | `asset_id` or `amount` does not exactly match a creation-fee row. |
| `InvalidAccountMultisigFeePayer` | The sponsor is ineligible, frozen, or `payer_policy_epoch` is stale. |
| `AccountNotFound` | The account does not exist. Deposit to create it first. |
| `AssetNotFound` | The `asset_id` in `creation_fee` is not a known asset. |
| `AccountMultisigActiveLimitExceeded` | The network's cap of 10,000 active configurations is reached. |
| `AccountMultisigLifecycleLimitExceeded` | The network's cap of 100,000 lifecycle entries is reached. |
| `AccountMultisigEpochOverflow` | The global policy epoch counter overflowed. Not expected in normal operation. |
