---
description: Move funds from a Native Core balance out to an EVM chain — the signed action, the amount rules, and how to confirm the payout landed.
---

# Withdraw

<figure><img src="../../.gitbook/assets/native-withdraw-flow.svg" alt=""><figcaption></figcaption></figure>

Read [Deposit & Withdraw](README.md) first for the shared endpoints, discovery queries, and rate limits.

## 1. Validate the amount

Three rules govern the amount. The first two come from `/info` and are enforced at admission. The third is enforced nowhere: break it and Native Core accepts the action, debits the balance, and the release is never constructed.

* **Minimum** — `min_withdraw_atoms` from the [`accountingWithdrawTokens`](../native-core/post-info.md#accountingwithdrawtokens) entry whose `chain_id` and `asset_id` match your destination chain and asset.
* **Fee** — `withdraw_fee_atoms` for the asset from [`assets`](../native-core/post-info.md#assets). It is recorded, not deducted: you sign the **gross** amount, Native debits the gross, and the destination chain releases `amount − fee`. `amount` must be strictly greater than the fee.
* **Decimal cap** — cap the amount at 6 decimal places. Native balances are 8-decimal and the release rescales into the destination token's decimals. USDT and USDC are 6-decimal on Ethereum, Arbitrum and Base, so a 6-decimal cap divides evenly into every destination token currently listed.

## 2. Sign and submit

`withdraw` is an owner action: sign it with your main wallet under `auth_scheme: "eip712"`, never with an API wallet. The field reference is [`withdraw`](../native-core/post-trade.md#withdraw); the scheme is [EIP-712 signing](../native-core/transaction-signing.md#eip-712-signing-auth_scheme-eip712).

Use the current Unix millisecond timestamp for both `withdraw_nonce` and the envelope `nonce`, incrementing locally if two withdrawals for the same account land in the same millisecond.

Persist `withdraw_nonce` before you sign. It is the handle for the rest of this flow, on both chains.

The typed data is the common EIP-712 prefix followed by the action's own fields:

```ts
const domain = { name: 'Native Core', version: '1', verifyingContract: '0x0000000000000000000000000000000000000000' }

const types = {
  Withdraw: [
    { name: 'nativeChainId',         type: 'uint256' },
    { name: 'authKind',              type: 'uint256' },
    { name: 'authScope',             type: 'uint256' },
    { name: 'nonce',                 type: 'uint256' },
    { name: 'expiresAfterMsPresent', type: 'bool'    },
    { name: 'expiresAfterMs',        type: 'uint256' },
    { name: 'assetId',               type: 'uint256' },
    { name: 'amount',                type: 'uint256' },
    { name: 'dstChainId',            type: 'uint256' },
    { name: 'dstAddress',            type: 'address' },
    { name: 'withdrawNonce',         type: 'uint256' },
    { name: 'cloidPresent',          type: 'bool'    },
    { name: 'cloid',                 type: 'bytes16' },
  ],
}

const withdrawNonce = BigInt(Date.now())
const message = {
  nativeChainId: 696969n,        // the Native chain id, not an EVM chain id
  authKind: 1n,
  authScope: 0n,
  nonce: withdrawNonce,
  expiresAfterMsPresent: false,
  expiresAfterMs: 0n,
  assetId: 2n,
  amount: 40000000000n,          // gross, 8-decimal atoms
  dstChainId: 56n,
  dstAddress: '0xbf381e1cbfdb0d02f3800010e490130d3dc73118',
  withdrawNonce,
  cloidPresent: true,
  cloid: '0x22222222222222222222222222222222',
}

const signature = await wallet.signTypedData({ domain, types, primaryType: 'Withdraw', message })
```

The domain carries **no `chainId`**, so a wallet signs it while connected to any EVM chain. Post the wallet's raw 65-byte `r‖s‖v` signature verbatim.

{% hint style="info" %}
Assert the recovery locally while you build the integration. Native derives the account from the recovered signer, so typed data that does not match this definition recovers a *different* address — one with no account — and the withdrawal comes back as `OwnerDoesNotExist`, never as a signature error. A wrong struct, a stray `chainId` in the domain, and a field in the wrong order all fail this way.

```ts
import { recoverTypedDataAddress } from 'viem'

const signer = await recoverTypedDataAddress({ domain, types, primaryType: 'Withdraw', message, signature })
if (signer.toLowerCase() !== wallet.account.address.toLowerCase()) throw new Error('typed data mismatch')
```

The digest is pinned and identical across implementations, so you can also assert it directly:

```ts
hashTypedData({
  domain, types, primaryType: 'Withdraw',
  message: {
    nativeChainId: 969696n, authKind: 1n, authScope: 0n, nonce: 99n,
    expiresAfterMsPresent: true, expiresAfterMs: 1700000000000n,
    assetId: 7n, amount: 1234567n, dstChainId: 56n,
    dstAddress: '0x1111111111111111111111111111111111111111',
    withdrawNonce: 42n, cloidPresent: true, cloid: '0x22222222222222222222222222222222',
  },
})
// 0xc657633ef3dcfd2726efbabfc56d400b307a15e8c5a88d571c39367c7c527fc3
```
{% endhint %}

```bash
curl -sS -X POST "$API_URL/trade" \
  -H 'content-type: application/json' \
  -d '{
    "action": {
      "type": "withdraw",
      "asset_id": "2",
      "amount": "40000000000",
      "dst_chain_id": "56",
      "dst_address": "0xbf381e1cbfdb0d02f3800010e490130d3dc73118",
      "withdraw_nonce": "1785162532020",
      "cloid": "0x22222222222222222222222222222222"
    },
    "nonce": "1785162532020",
    "auth_scheme": "eip712",
    "signature": "0x…"
  }'
```

Branch on `submission_status`, never on the HTTP status — `/trade` returns the same body shape for 200, 400, 429, 503 and 504.

* `accepted` — the action executed and the balance is debited.
* `rejected` — fix the `error.code` before you sign a fresh action.
* `timeout` — **not** a rejection. The withdrawal may still commit, so reconcile by `cloid` with `txStatusByCloid` instead of re-signing.

The decision playbook is [Handle outcomes & timeouts](../native-core/handle-timeouts.md); the code catalog is [Error Responses](../native-core/error-responses.md).

The authority is the recovered signer, so the debited account is whoever signed. `dst_address` is only the EVM payout target — watch it on the destination chain.

## 3. Confirm the Native debit

[`withdraws`](../native-core/post-info.md#withdraws) returns the executed withdrawal records for an address. Find yours by `withdraw_nonce`.

```bash
curl -sS -X POST "$API_URL/info" \
  -H 'content-type: application/json' \
  -d '{"type":"withdraws","user":"0xbf381e1cbfdb0d02f3800010e490130d3dc73118"}'
```

This read is a **3-day window**. It confirms the debit; it is not durable history, so record the withdrawal on your side at submit time.

Each record's `tx_hash` is the Native debit. Open it on the [Native explorer](https://app.native.org/explorer) to inspect the withdrawal by hand:

```
https://app.native.org/explorer/tx/<tx_hash>
https://app.native.org/explorer/address/<user>
```

## 4. Watch the destination chain

Poll `usedNonces(dstAddress, withdrawNonce)` on the destination chain until it returns `true`, on an interval that suits that chain. You never call `withdraw()` yourself — the vault rejects anything but the operator's multi-signed call.

`usedNonces` is the vault's permanent replay guard, keyed by the payout address and your `withdraw_nonce`. It flips to `true` in the same transaction that transfers the tokens, so `true` is terminal.

```solidity
function usedNonces(address user, uint256 nonce) external view returns (bool);
```

```ts
const released = await publicClient.readContract({
  address: vault, abi: vaultAbi, functionName: 'usedNonces',
  args: [dstAddress, withdrawNonce],
})
```

Compute the credited amount yourself: `amount − withdraw_fee_atoms`, from values you already hold.

To watch the token's `Transfer` event instead, pin the destination ERC20 addresses for the routes you support at integration time. `/info` does not map `(asset_id, chain_id)` onto an address, and symbol matching breaks on the wrapped-native routes — `ETH` is `WETH`, `BNB` is `WBNB` — and on Arbitrum `USDT`, which is `USD₮0`.

A withdrawal debited on Native that still shows `usedNonces == false` after a long wait is stalled, not lost. The release pipeline retries and never drops a debited withdrawal; the decimal cap in step 1 is the most common cause.

## What can go wrong

| Symptom                                                 | Cause                                                                   | What to do                                                                    |
| ------------------------------------------------------- | ------------------------------------------------------------------------ | ------------------------------------------------------------------------------ |
| `WithdrawAmountBelowMinimum`                            | Below `min_withdraw_atoms` for the route                                 | Re-read `accountingWithdrawTokens`                                             |
| `WithdrawAmountNotAboveFee`                             | Amount is not strictly greater than the asset's withdraw fee             | Re-read `withdraw_fee_atoms` in `assets`                                       |
| `WithdrawInsufficientBalance`                           | Available balance does not cover the gross amount                        | Re-read `userBalances`; locked balance does not count                          |
| `WithdrawDuplicateNonce`                                | `withdraw_nonce` reused, or below the 3-day pruned floor                 | Use a fresh millisecond timestamp                                              |
| `ActionNotAllowedForSpotCreditAccount`                  | The signer is a credit account                                           | Credit accounts cannot use this path — see [Account Types](../native-core/account-types.md) |
| `OwnerDoesNotExist` for an account you know exists      | The typed data does not match, so the recovered signer is a different address | Recover your own signature locally and compare it to the signing address |
| `submission_status: "timeout"`                          | Outcome not observed in the wait budget — it may still commit            | Reconcile by `cloid`; never re-sign under a new nonce                          |
| Accepted, `usedNonces` stays `false`                    | Amount is not exactly representable in the destination token's decimals  | Cap amounts at 6 decimal places; contact Native for a stalled one              |

## Next steps

{% content-ref url="vault-contract.md" %}
[vault-contract.md](vault-contract.md)
{% endcontent-ref %}

{% content-ref url="deposit.md" %}
[deposit.md](deposit.md)
{% endcontent-ref %}
