---
description: Move funds from an EVM chain into a Native Core balance — the vault call, the activation fee, and how to confirm the credit landed.
---

# Deposit

Five steps: check whether the account exists, size the activation fee it may owe, approve and deposit, read the nonce off the receipt, then wait for the credit on Native Core.

You initiate the EVM transaction. Native's settlement pipeline observes it and credits the balance — there is nothing to submit to Native, and closing your process after the transaction lands does not affect the credit.

<figure><img src="../../.gitbook/assets/native-deposit-flow.svg" alt=""><figcaption></figcaption></figure>

Read [Deposit & Withdraw](README.md) first for the shared endpoints, discovery queries, and rate limits.

## 1. Check whether the account exists

An address that has never held a Native Core balance has no account yet, and its first deposit must pay a one-time activation fee. [`accountStatus`](../native-core/post-info.md#accountstatus) is the authoritative signal.

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

## 2. Size the activation fee

The fee rides the deposit transaction as `msg.value`, in the source chain's gas token. No token allowance or second transaction is involved.

* The fee is worth **1 USD** in the chain's gas token, priced at the time the deposit is validated.
* Validation accepts a shortfall of up to **30%**, which absorbs the price movement between your quote and settlement.
* Overpayment is accepted and never refunded. Quote at your own spot price with a small buffer.
* Send `msg.value: 0` for every deposit into an account that already exists.

Price it from [`markPrices`](../native-core/post-info.md#markprices) — the same oracle the deposit is validated against, so your quote cannot drift out of tolerance the way an external feed can. The gas token is 18-decimal on all four chains.

| Source chain             | Gas token | `asset_id` |
| ------------------------ | --------- | ---------- |
| Ethereum, Base, Arbitrum | ETH       | `3`        |
| BNB Smart Chain          | BNB       | `4`        |

```bash
curl -sS -X POST "$API_URL/info" \
  -H 'content-type: application/json' \
  -d '{"type":"markPrices"}'
```

```json
{ "mark_prices": [ { "asset_id": 3, "usd_atoms": 188483000000, "source_ts_ms": 1785222009197 } ] }
```

`usd_atoms` is a USD price in 8-decimal atoms, so one dollar of an 18-decimal gas token is `10**26 / usd_atoms` wei:

```ts
const usdAtoms = BigInt(markPrices.find(p => p.asset_id === gasAssetId).usd_atoms)
const activationFeeWei = 10n ** 26n / usdAtoms
```

{% hint style="info" %}
A first deposit that underpays past the tolerance is a **permanent** validation failure. The tokens sit in the vault, the balance is never credited, and the order is not retried — recovering it takes manual intervention from Native. Read `accountStatus` immediately before you build the transaction, and never guess the fee to zero.
{% endhint %}

## 3. Approve and deposit

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

const requested = parseUnits('100', 18)                // BSC USDT is 18-decimal
const amount = requested - (requested % 10n ** 10n)    // floor to the 8-decimal grid
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

{% hint style="info" %}
**Floor the amount to 8 decimal places yourself — the vault does not.** Native credits at 8 decimals (`balance_decimals`) and settlement refuses any deposit whose amount does not convert to 8 decimals exactly. The contract carries no matching check: an over-precise amount produces a **successful** transaction, and the credit is then held for manual review. Deposits are forward-only, so nothing is returned.

```ts
const scale = 10n ** BigInt(Math.max(0, decimals - 8))   // 10^10 for an 18-decimal token
const amount = requested - (requested % scale)
```

Tokens with 8 decimals or fewer — WBTC, cbBTC, and USDC/USDT on Ethereum, Base and Arbitrum — cannot carry excess precision and need no adjustment. `minDepositDecimalByUnderlying(token)` returns the grid (`8` on every listed token) if you would rather read it than assume it.
{% endhint %}

**A deposit must also be worth at least 10 USD.** Settlement enforces that floor and the vault does not, so an amount below it produces a successful transaction that is never credited, never appears in `deposits`, and is never retried. Recovering it takes manual intervention from Native.

The floor is a fixed amount of the deposited token, set per chain and token, rather than a conversion from the current price. On a stablecoin it is 10 tokens. On a volatile token, quote above 10 USD rather than exactly at it: a rise in that token's price lifts the floor above what 10 USD buys.

The one condition the vault does gate is **pause**: `isDepositPaused(token)` covers a single token, `emergencyPaused()` the whole vault. Read both before you build the transaction so the user sees a reason instead of a revert.

To fund an address other than the caller — a wallet or aggregator depositing on a user's behalf — use `depositFor`, which credits `user` instead of `msg.sender`:

```solidity
function depositFor(address token, uint256 amount, address user, uint256 actionFlag)
    external payable returns (uint256 wNLPAmount);
```

The activation fee follows the credited address, not the caller: read `accountStatus` for `user`.

## 4. Read the deposit nonce from the receipt

The vault assigns each deposit a sequential `nonce` and emits it in the `Deposit` event. That nonce is how Native Core identifies the deposit, and the only handle that survives into the credit record.

```ts
import { decodeEventLog } from 'viem'

const log = receipt.logs.find(l => l.address.toLowerCase() === vault.toLowerCase())
const { args } = decodeEventLog({ abi: vaultAbi, data: log.data, topics: log.topics })
const depositNonce = args.nonce          // e.g. 31n
```

## 5. Wait for the credit

Native's settlement pipeline waits for the source transaction to be safely confirmed before it credits the balance. The credit lands **about 5 minutes** after the deposit transaction — a typical latency, not a guarantee, so poll for the record instead of sleeping on a fixed timer.

Poll [`deposits`](../native-core/post-info.md#deposits) and match on the tuple `(src_chain_id, src_contract, deposit_nonce)`.

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

A matching record means the balance is credited and tradable. `asset_id` tells you which Native asset the ERC20 mapped to, so you never need a token-to-asset table of your own. `amount_atoms` is the credited amount in that asset's `balance_decimals` (8), rescaled from the token's own decimals — the on-chain `70507040000000000` at 18 decimals arrives as `7050704` at 8.

[`userBalances`](../native-core/post-info.md#userbalances) is the settled-state read once the deposit is done.

`tx_hash` and `block_height` refer to Native Core, not the source chain. Open the credit on the [Native explorer](https://app.native.org/explorer) to inspect it by hand:

```
https://app.native.org/explorer/tx/<tx_hash>
https://app.native.org/explorer/address/<user>
```

The address page lists the account's deposits and withdrawals, which is the quickest way to check a user's funding history without polling.

## What can go wrong

| Symptom                                        | Cause                                                                                                                       | What to do                                                              |
| ---------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------- |
| `deposit()` reverts `UnsupportedUnderlying`    | Token is not listed on this vault                                                                                            | Read `getSupportedUnderlyings()` before submitting                       |
| `deposit()` reverts with no data to decode     | Allowance or balance too low — most ERC20s revert here without a reason string, so nothing decodes                            | Re-check the allowance you set in step 3                                |
| Mined, still no credit well past 5 minutes     | Amount was below the 10 USD minimum, was not on the 8-decimal grid, or a first deposit underpaid the activation fee — read `getDepositRecord(nonce)` and check `amount` and `msgValue` | None of the three is self-recoverable — contact Native with the source tx hash |
| Mined, no credit yet, account already existed  | Still inside the normal settlement delay                                                                                     | Keep polling; report a credit that has not landed well past 5 minutes    |
| `429` with `RateLimited`                       | More than 1 `/info` request per second from one IP                                                                           | Wait the `retry after` interval in the error, and share one budget across loops |

## Next steps

{% content-ref url="withdraw.md" %}
[withdraw.md](withdraw.md)
{% endcontent-ref %}

{% content-ref url="vault-contract.md" %}
[vault-contract.md](vault-contract.md)
{% endcontent-ref %}
