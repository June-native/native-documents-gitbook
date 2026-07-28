---
description: Fund and defund a Native Core account from your own application — the vault call, the signed withdrawal, and how to confirm each one landed.
---

# Deposit & withdraw

Funding is the one part of Native Core that spans two chains. A deposit starts on an EVM chain and finishes on Native Core; a withdrawal starts on Native Core and finishes on an EVM chain. Each direction gives you exactly one transaction to send and one thing to poll.

An EVM RPC endpoint, `POST /info`, and `POST /trade` are the whole surface — no other Native-hosted service is involved.

{% hint style="warning" %}
`/info` and `/trade` are server-to-server APIs. They do not return `Access-Control-Allow-Origin` for third-party origins, so a browser cannot call them cross-origin. Run these calls from your backend.
{% endhint %}

## The model

|                      | Deposit                                   | Withdraw                                |
| -------------------- | ----------------------------------------- | --------------------------------------- |
| You send              | `deposit()` on the vault contract         | A signed `withdraw` action to `/trade`  |
| Native settles the other side | credits the balance after EVM finality | releases tokens from the vault          |
| You confirm with      | `POST /info` `deposits`                   | vault `usedNonces(...)`                 |
| Correlation key       | The `Deposit` event's `nonce`             | The `withdraw_nonce` you chose          |

Neither direction can be cancelled or reversed once the first transaction lands. A deposit that fails validation is held for manual review, not returned; a withdrawal debits your Native balance the moment the action executes, before anything moves on EVM.

```bash
API_URL=https://api.native.org
```

## Before you start

Resolve the routing table at runtime. Vaults are redeployed and asset listings change, so treat every address and limit below as a read, never a constant.

| What you need                         | Where it comes from                                                  |
| ------------------------------------- | -------------------------------------------------------------------- |
| Vault address per chain               | `POST /info` [`accountingDepositContracts`](post-info.md#accountingdepositcontracts) |
| Depositable tokens on a chain         | vault `getSupportedUnderlyings()`                                     |
| `asset_id`, `balance_decimals`, withdrawal fee | `POST /info` [`assets`](post-info.md#assets)                 |
| Withdrawable `(chain, asset)` and minimums | `POST /info` [`accountingWithdrawTokens`](post-info.md#accountingwithdrawtokens) |

```bash
curl -sS -X POST "$API_URL/info" \
  -H 'content-type: application/json' \
  -d '{"type":"accountingDepositContracts"}'
```

```json
{
  "accounting_deposit_contracts": [
    { "src_chain_id": 1,     "src_contract": "0xc91807c59b354437eae0de32f153c06665cd2270" },
    { "src_chain_id": 56,    "src_contract": "0xd7afd8fbecc1ae7a4ce5b93eb59d76f967d0dea7" },
    { "src_chain_id": 8453,  "src_contract": "0x79292d171531673ff97035315fda568189c3c8a5" },
    { "src_chain_id": 42161, "src_contract": "0x5d4c35e9c9a06061ba80e34b531bba88bf9952bc" }
  ],
  "query_height": 126780651,
  "app_hash": "0x…"
}
```

The contract surface these calls hit — function signatures, events, and revert reasons — is in [Vault Contract](vault-contract.md).

{% hint style="info" %}
`/info` is rate limited to **1 request per second per IP**. Every polling loop on this page needs an interval at or above that, and a shared budget if you run several at once.
{% endhint %}

## Deposit

Five steps: check whether the account exists, size the activation fee it may owe, approve and deposit, read the nonce off the receipt, then wait for the credit on Native Core.

### 1. Check whether the account exists

An address that has never held a Native Core balance has no account yet, and its first deposit must pay a one-time activation fee. [`accountStatus`](post-info.md#accountstatus) is the authoritative signal.

```bash
curl -sS -X POST "$API_URL/info" \
  -H 'content-type: application/json' \
  -d '{"type":"accountStatus","user":"0xbf381e1cbfdb0d02f3800010e490130d3dc73118"}'
```

```json
{ "found": true, "account_index": 6, "status": "active", "query_height": 126781073, "app_hash": "0x…" }
```

`found: true` means the account exists and the deposit carries no fee. `found: false` means this deposit will create it, and must carry the fee described next.

Treat an unreadable `accountStatus` as blocking. The vault does not enforce the fee, so a deposit sent while the answer is unknown can land on-chain and then fail validation.

### 2. Size the activation fee

The fee rides the deposit transaction as `msg.value`, in the source chain's gas token. No token allowance or second transaction is involved.

* The fee is worth **1 USD** in the chain's gas token, priced at the time the deposit is validated.
* Validation accepts a shortfall of up to **30%**, which absorbs the price movement between your quote and settlement.
* Overpayment is accepted and never refunded. Quote at your own spot price with a small buffer.
* Send `msg.value: 0` for every deposit into an account that already exists.

{% hint style="danger" %}
A first deposit that underpays past the tolerance is a **permanent** validation failure. The tokens sit in the vault, the balance is never credited, and the order is not retried — recovering it takes manual intervention from Native. Read `accountStatus` immediately before you build the transaction, and never guess the fee to zero.
{% endhint %}

### 3. Approve and deposit

`deposit()` pulls the ERC20 from the caller, so it needs an allowance first. Amounts are in the token's own decimals.

```solidity
function deposit(address token, uint256 amount, uint256 actionFlag)
    external payable returns (uint256 wNLPAmount);
```

Pass `actionFlag: 0` — the standard deposit path.

```ts
import { createPublicClient, createWalletClient, http, erc20Abi, parseUnits } from 'viem'
import { bsc } from 'viem/chains'
import { vaultAbi } from './vaultAbi'

const publicClient = createPublicClient({ chain: bsc, transport: http(RPC_URL) })
const wallet = createWalletClient({ chain: bsc, transport: http(RPC_URL), account })

const amount = parseUnits('100', 18)          // BSC USDT is 18-decimal
const feeWei = accountExists ? 0n : activationFeeWei

const [vaultPaused, tokenPaused] = await Promise.all([
  publicClient.readContract({ address: vault, abi: vaultAbi, functionName: 'emergencyPaused' }),
  publicClient.readContract({ address: vault, abi: vaultAbi, functionName: 'isDepositPaused', args: [token] }),
])
if (vaultPaused || tokenPaused) throw new Error('deposits are paused')

const approveHash = await wallet.writeContract({
  address: token, abi: erc20Abi, functionName: 'approve', args: [vault, amount],
})
await publicClient.waitForTransactionReceipt({ hash: approveHash })

const depositHash = await wallet.writeContract({
  address: vault, abi: vaultAbi, functionName: 'deposit',
  args: [token, amount, 0n], value: feeWei,
})
const receipt = await publicClient.waitForTransactionReceipt({ hash: depositHash })
```

Two on-chain rules reject a deposit before it reaches Native:

* **Precision.** `minDepositDecimalByUnderlying(token)` returns `8` for every listed token, matching Native's `balance_decimals`. An amount carrying more than 8 decimal places reverts `InvalidAmount` — for an 18-decimal token, `amount` must be a multiple of `10^10`.
* **Pause.** `isDepositPaused(token)` gates a single token; `emergencyPaused()` gates the whole vault. Read both before you build the transaction so the user sees a reason instead of a revert.

To fund an address other than the caller — a wallet or aggregator depositing on a user's behalf — use `depositFor`, which credits `user` instead of `msg.sender`:

```solidity
function depositFor(address token, uint256 amount, address user, uint256 actionFlag)
    external payable returns (uint256 wNLPAmount);
```

The activation fee follows the credited address, not the caller: read `accountStatus` for `user`.

### 4. Read the deposit nonce from the receipt

The vault assigns each deposit a sequential `nonce` and emits it in the `Deposit` event. That nonce is how Native Core identifies the deposit, and the only handle that survives into the credit record.

```ts
import { decodeEventLog } from 'viem'

const log = receipt.logs.find(l => l.address.toLowerCase() === vault.toLowerCase())
const { args } = decodeEventLog({ abi: vaultAbi, data: log.data, topics: log.topics })
const depositNonce = args.nonce          // e.g. 31n
```

### 5. Wait for the credit

Native Core credits the balance only after the source transaction is irreversible, judged by the chain's own `finalized` block tag rather than a confirmation count. Budget for it:

| Chain    | Time to `finalized` |
| -------- | ------------------- |
| BSC      | ~1 second           |
| Ethereum | ~13–15 minutes      |
| Arbitrum | ~15 minutes         |
| Base     | ~15–20 minutes      |

Then poll [`deposits`](post-info.md#deposits) and match on the tuple `(src_chain_id, src_contract, deposit_nonce)`.

```bash
curl -sS -X POST "$API_URL/info" \
  -H 'content-type: application/json' \
  -d '{"type":"deposits","user":"0xbf381e1cbfdb0d02f3800010e490130d3dc73118"}'
```

```json
{
  "deposits": [
    {
      "tx_hash": "0x2dd46662ecc762d70291e9475e50890ffb4c79c85e61c539f1e138a8b71b82a7",
      "block_height": 125866793,
      "block_timestamp_ms": 1785157736296,
      "asset_id": 44,
      "amount_atoms": "7050704",
      "src_chain_id": 56,
      "src_contract": "0xd7afd8fbecc1ae7a4ce5b93eb59d76f967d0dea7",
      "deposit_nonce": "30"
    }
  ],
  "user": "0xbf381e1cbfdb0d02f3800010e490130d3dc73118"
}
```

A matching record means the balance is credited and tradable. `asset_id` tells you which Native asset the ERC20 mapped to, so you never need a token-to-asset table of your own. `amount_atoms` is the credited amount in that asset's `balance_decimals` (8), rescaled from the token's own decimals — the on-chain `70507040000000000` at 18 decimals arrives as `7050704` at 8. `tx_hash` and `block_height` refer to Native Core, not the source chain.

[`userBalances`](post-info.md#userbalances) is the settled-state read once the deposit is done.

## Withdraw

Four steps: validate the amount against the route's limits, sign and submit the action, confirm Native Core executed it, then watch the destination chain for the release.

### 1. Validate the amount

Read the route's constraints first. Both come from `/info` and are enforced at admission.

* **Minimum** — `min_withdraw_atoms` from the [`accountingWithdrawTokens`](post-info.md#accountingwithdrawtokens) row whose `chain_id` and `asset_id` match your destination chain and asset.
* **Fee** — `withdraw_fee_atoms` for the asset from [`assets`](post-info.md#assets). It is recorded, not deducted: you sign the **gross** amount, Native debits the gross, and the destination chain releases `amount − fee`. `amount` must be strictly greater than the fee.

{% hint style="danger" %}
**Cap withdrawal amounts at 6 decimal places.** Native balances are 8-decimal, and the release rescales the amount into the destination token's decimals. USDT and USDC are 6-decimal on Ethereum, Arbitrum and Base, so an amount that is not a whole number of atoms at 6 decimals cannot be rescaled exactly. Native Core accepts and executes the action — the balance is debited — but the release is never constructed and the withdrawal stalls. A 6-decimal cap divides evenly into every destination token currently listed.
{% endhint %}

### 2. Sign and submit

`withdraw` is an owner action: sign it with your main wallet under `auth_scheme: "eip712"`, never with an API wallet. The field reference is [`withdraw`](post-trade.md#withdraw); the scheme is [EIP-712 signing](transaction-signing.md#eip-712-signing-auth_scheme-eip712).

Use the current Unix millisecond timestamp for both `withdraw_nonce` and the envelope `nonce`, incrementing locally if you send more than one withdrawal for the same account in the same millisecond. `withdraw_nonce` is the handle you use for the rest of this flow, on both chains — persist it before you sign.

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
  nativeChainId: 696969n,        // 969696 on testnet
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

The domain carries **no `chainId`**, so a wallet signs it while connected to any EVM chain; `nativeChainId` inside the message binds the signature to an environment. Post the wallet's raw 65-byte `r‖s‖v` signature verbatim.

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

Branch on `submission_status`, never on the HTTP status — `/trade` returns the same body shape for 200, 400, 429, 503 and 504. `accepted` means the action executed and the balance is debited. `rejected` carries an `error.code` to fix before you sign a fresh action. `timeout` is **not** a rejection: the withdrawal may still commit, so reconcile it by `cloid` with `txStatusByCloid` instead of re-signing — that is what the required `cloid` is for. The decision playbook is [Handle outcomes & timeouts](handle-timeouts.md); the code catalog is [Error Responses](error-responses.md).

The authority is the recovered signer, so the debited account is whoever signed. `dst_address` is only the EVM payout target — watch it on the destination chain.

### 3. Confirm the Native debit

[`withdraws`](post-info.md#withdraws) returns the executed withdrawal records for an address. Find yours by `withdraw_nonce`.

```bash
curl -sS -X POST "$API_URL/info" \
  -H 'content-type: application/json' \
  -d '{"type":"withdraws","user":"0xbf381e1cbfdb0d02f3800010e490130d3dc73118"}'
```

This read is a **3-day window**. It confirms the debit; it is not durable history, so record the withdrawal on your side at submit time.

### 4. Watch the destination chain

Native's signer network co-signs the release and submits it to the vault. You do not call `withdraw()` on the contract — the vault rejects anything but the operator's multi-signed call.

The release is observable without any indexer. `usedNonces` is the vault's permanent replay guard, keyed by the payout address and your `withdraw_nonce`. It flips to `true` in the same transaction that transfers the tokens:

```solidity
function usedNonces(address user, uint256 nonce) external view returns (bool);
```

```ts
const released = await publicClient.readContract({
  address: vault, abi: vaultAbi, functionName: 'usedNonces',
  args: [dstAddress, withdrawNonce],
})
```

`true` is terminal — poll on an interval that suits the destination chain and stop on the first `true`. For the exact credited amount, read the destination token's `Transfer` event from the vault to `dstAddress` in that block, or take `amount − withdraw_fee_atoms` from the values you already hold.

A withdrawal debited on Native that still shows `usedNonces == false` after a long wait is stalled, not lost. The release pipeline retries and never drops a debited withdrawal; the amount-precision rule in step 1 is the most common cause.

## What can go wrong

| Symptom                                              | Cause                                                                                 | What to do                                                            |
| ---------------------------------------------------- | ------------------------------------------------------------------------------------- | --------------------------------------------------------------------- |
| `deposit()` reverts `InvalidAmount`                  | Amount carries more than 8 decimal places                                             | Round down to the `minDepositDecimalByUnderlying` grid                |
| `deposit()` reverts `DepositPaused` / `UnsupportedUnderlying` | Token paused, or not listed on this vault                                    | Read `isDepositPaused` and `getSupportedUnderlyings` before submitting |
| Deposit mined, no credit after the finality window   | First deposit underpaid the activation fee — confirm with `getDepositRecord(nonce).msgValue` | Not self-recoverable — contact Native with the source tx hash   |
| Deposit mined, no credit, account already existed    | Still inside the finality window                                                      | Wait out the window for that chain, then re-poll                       |
| `/trade` returns `WithdrawAmountBelowMinimum`        | Below `min_withdraw_atoms` for the route                                              | Re-read `accountingWithdrawTokens`                                     |
| `/trade` returns `WithdrawAmountNotAboveFee`         | Amount is not strictly greater than the asset's withdraw fee                          | Re-read `withdraw_fee_atoms` in `assets`                               |
| `/trade` returns `WithdrawDuplicateNonce`            | `withdraw_nonce` reused, or below the 3-day pruned floor                              | Use a fresh millisecond timestamp                                      |
| `/trade` returns `ActionNotAllowedForSpotCreditAccount` | The signer is a credit account                                                     | Credit accounts cannot use this path — see [Account Types](account-types.md) |
| Withdraw accepted, `usedNonces` stays `false`        | Amount is not exactly representable in the destination token's decimals               | Cap amounts at 6 decimal places; contact Native for a stalled one      |
| `429` with `RateLimited`                             | More than 1 `/info` request per second from one IP                                    | Widen the polling interval; share one budget across loops              |

## Next steps

{% content-ref url="vault-contract.md" %}
[vault-contract.md](vault-contract.md)
{% endcontent-ref %}

{% content-ref url="post-info.md" %}
[post-info.md](post-info.md)
{% endcontent-ref %}

{% content-ref url="decimals-units.md" %}
[decimals-units.md](decimals-units.md)
{% endcontent-ref %}
