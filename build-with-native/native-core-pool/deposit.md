---
description: Move funds into a Native Core Pool earning balance, from an EVM chain or from an existing Native Core balance.
---

# Deposit

Two routes lead into the same `earn_balance`. Pick by where the money already is.

|                | From an EVM chain                              | From a Native Core balance             |
| -------------- | ---------------------------------------------- | -------------------------------------- |
| You send       | `deposit()` on the vault contract               | A signed `transfer` action             |
| Signed with    | An EVM transaction                              | EIP-712 typed data                     |
| Settles in     | Minutes                                         | About one Core block                   |
| `deposit_type` | `bridge_deposit`                                | `direct_transfer`                      |

Neither route can be cancelled or reversed once the first transaction lands. There is no minimum on the Pool side; the EVM route has a per-token minimum enforced by the bridge.

Read [Native Core Pool](README.md) first for the base URL, the response envelope, and discovery.

## Which assets you can deposit

`config` is the authority. An asset accepts deposits when it appears in `assets[]` with `deposit_enabled: true`.

```bash
curl -sS "$POOL_API_URL/api/v3/earn" \
  -H 'content-type: application/json' \
  -d '{"type":"config"}'
```

The token registry lists everything Native Core supports, which is a much larger set. A registry token whose `assetId` has no matching entry in `config` cannot reach the Pool: on the EVM route the deposit is recorded and then rejected, and on the Core route it lands in the vault with no record at all.

## From an EVM chain

### 1. Resolve the token and its minimum

```bash
curl -sS "$POOL_API_URL/api/v3/core/registry"
```

```json
{
  "code": 0,
  "data": {
    "chains": [
      {
        "chainKey": "ethereum",
        "evmChainId": 1,
        "address": "0xc91807C59B354437eaE0dE32F153c06665cD2270",
        "enabled": true,
        "underlyings": [
          {
            "symbol": "USDT",
            "address": "0xdAC17F958D2ee523a2206206994597C13D831ec7",
            "decimals": 6,
            "assetId": 2,
            "minDepositDecimal": 8,
            "minDepositAmountWei": "10000000",
            "enabled": true
          }
        ]
      }
    ]
  },
  "message": "success"
}
```

`chains[].address` is the vault to call on that chain. `underlyings[].address` is the ERC20 to deposit, in its own `decimals`.

`minDepositAmountWei` is the enforced floor, in that ERC20's smallest unit. It is a **string**, and on 18-decimal tokens it exceeds JavaScript's safe integer range, so compare with `BigInt`. A deposit below the floor is accepted on-chain and then stalls off-chain, with nothing to poll and no refund.

### 2. Size the activation fee

A Native Core account is created by its owner's first deposit, and that deposit carries a one-time activation fee as `msg.value`, in the source chain's gas token.

* Read [`accountStatus`](../native-core/post-info.md#accountstatus) for the credited address. `found: false` means this deposit pays the fee.
* **Send `msg.value: 0` for every deposit into an account that already exists.** The fee is not checked on those, so anything you attach is kept.
* Validation accepts a shortfall of up to **30%**, which absorbs price movement between your quote and settlement.

```bash
curl -sS "$POOL_API_URL/api/v3/accounting" \
  -H 'content-type: application/json' \
  -d '{"type":"depositInitFee"}'
```

```json
{
  "code": 0,
  "data": {
    "items": [
      {
        "chain_id": 1,
        "fee_usd": 1,
        "gas_token_symbol": "ETH",
        "gas_token_price_usd": "1875.224981",
        "native_token_decimals": 18,
        "fee_amount": "0.00053326934641556",
        "fee_amount_wei": "533269346415560",
        "available": true
      }
    ]
  },
  "message": "success"
}
```

Take `fee_amount_wei` for your chain and send it as `msg.value`. It tracks the gas token's price, so quote it immediately before you build the transaction rather than caching it.

`fee_amount` is the same value in decimal form. Do not send transactions with it; the float conversion loses precision.

{% hint style="warning" %}
`available: false` means the gateway could not price the fee right now, not that the chain is closed. On that row `fee_amount_wei` is not a usable number, so parsing it will throw. Retry the query instead of blocking the deposit permanently.

`fee_usd` is set per chain by operations. It is `1` on every chain today, but read it rather than hardcoding a dollar.
{% endhint %}

### 3. Approve and deposit

```solidity
function deposit(address token, uint256 amount, uint256 actionFlag)
    external payable returns (uint256 wNLPAmount);
```

**`actionFlag` must be `1`.** That is the value that routes the deposit into the Pool. `actionFlag: 0` is an ordinary Native Core deposit: it succeeds, credits the user's trading balance, and never appears in the Pool's deposit list.

`deposit()` pulls the ERC20 from the caller, so set an allowance first. To credit an address other than the caller, use `depositFor(address token, uint256 amount, address user, uint256 actionFlag)`; the activation fee then follows `user`, not the caller.

```ts
import { createPublicClient, createWalletClient, http, erc20Abi, parseUnits } from 'viem'
import { mainnet } from 'viem/chains'
import { vaultAbi } from './vaultAbi'

const publicClient = createPublicClient({ chain: mainnet, transport: http(RPC_URL) })
const wallet = createWalletClient({ chain: mainnet, transport: http(RPC_URL), account })

const requested = parseUnits('100', 6)                 // Ethereum USDT is 6-decimal
const scale = 10n ** BigInt(Math.max(0, decimals - 8)) // 1 here, 10^10 on an 18-decimal token
const amount = requested - (requested % scale)         // floor onto the 8-decimal grid
if (amount < BigInt(minDepositAmountWei)) throw new Error('below the token minimum')

const feeWei = accountExists ? 0n : BigInt(feeAmountWei)

const approveHash = await wallet.writeContract({
  address: token, abi: erc20Abi, functionName: 'approve', args: [vault, amount],
})
await publicClient.waitForTransactionReceipt({ hash: approveHash })

const depositHash = await wallet.writeContract({
  address: vault, abi: vaultAbi, functionName: 'deposit',
  args: [token, amount, 1n], value: feeWei,
})
const receipt = await publicClient.waitForTransactionReceipt({ hash: depositHash })
```

{% hint style="danger" %}
**Floor the amount onto the 8-decimal grid before you submit.** Native credits at 8 decimals, and an amount carrying more precision than that cannot be credited. Tokens with 8 decimals or fewer, including USDT and USDC on Ethereum, Base and Arbitrum, cannot carry excess precision and need no adjustment. The 18-decimal tokens, including USDT and USDC on BNB Smart Chain, do. `minDepositDecimal` in the registry is the grid.
{% endhint %}

The contract surface, the read calls that tell you whether a deposit will succeed, and the ABI fragment are in [Vault Contract](../deposit-withdraw/vault-contract.md).

### 4. Wait for the credit

The deposit becomes visible when it is credited, and not before. Poll `deposits` and match your source transaction hash.

```bash
curl -sS "$POOL_API_URL/api/v3/earn" \
  -H 'content-type: application/json' \
  -d '{"type":"deposits","user_address":"0xbf381e…","limit":20}'
```

```json
{
  "code": 0,
  "data": {
    "items": [
      {
        "id": 9,
        "operation_id": "deposit:1:0xc91807c59b354437eae0de32f153c06665cd2270:31",
        "deposit_type": "bridge_deposit",
        "asset_id": 2,
        "amount": "10000000000",
        "status": "credited",
        "core_height": 153570000,
        "core_event_timestamp_ms": 1786620000000,
        "src_chain_id": 1,
        "src_tx_hash": "0x2dd46662ecc762d70291e9475e50890ffb4c79c85e61c539f1e138a8b71b82a7",
        "core_tx_hash": "0x04530cb273e03c64c141…"
      }
    ],
    "next_before_id": null
  },
  "message": "success"
}
```

`status: "credited"` means the amount is in `earn_balance` and earning. `status: "rejected"` means it will not be, and the funds need manual recovery.

`amount` is in the asset's 8-decimal `balance_decimals`, rescaled from the token's own decimals: `100000000` sent as 6-decimal USDT arrives as `10000000000`.

Hold your own record between submitting the transaction and seeing it appear. The source transaction, the bridge, and Core settlement each take time, and none of those stages is queryable — the record does not exist until it reaches `credited` or `rejected`. A deposit that has not appeared long after the source transaction confirmed should be reported with its source transaction hash rather than retried.

## From a Native Core balance

This route moves an existing Native Core balance into the Pool with one signed action. No EVM chain is involved and there is no minimum.

### 1. Read the vault address

`vault_address` from [`config`](reference.md#config) is the recipient. Read it at runtime; a transfer to a superseded address is not credited.

### 2. Sign and submit the transfer

`transfer` is a Native Core action, submitted to the Native Core API rather than to the Pool API. It is an owner action: sign it with the main wallet, never with an API wallet.

```ts
const domain = { name: 'Native Core', version: '1', verifyingContract: '0x0000000000000000000000000000000000000000' }

const types = {
  Transfer: [
    { name: 'nativeChainId',         type: 'uint256' },
    { name: 'authKind',              type: 'uint256' },
    { name: 'authScope',             type: 'uint256' },
    { name: 'nonce',                 type: 'uint256' },
    { name: 'expiresAfterMsPresent', type: 'bool'    },
    { name: 'expiresAfterMs',        type: 'uint256' },
    { name: 'to',                    type: 'address' },
    { name: 'assetId',               type: 'uint256' },
    { name: 'amount',                type: 'uint256' },
    { name: 'cloidPresent',          type: 'bool'    },
    { name: 'cloid',                 type: 'bytes16' },
  ],
}

const nonce = BigInt(Date.now())
const message = {
  nativeChainId: 696969n,        // the Native chain id, not an EVM chain id
  authKind: 1n,
  authScope: 0n,
  nonce,
  expiresAfterMsPresent: false,
  expiresAfterMs: 0n,
  to: vaultAddress,              // config.vault_address
  assetId: 2n,
  amount: 100000000n,            // 8-decimal atoms
  cloidPresent: true,
  cloid: '0x44c25ffe3c4f091bf28501543b63a59f',
}

const signature = await wallet.signTypedData({ domain, types, primaryType: 'Transfer', message })
```

The domain carries **no `chainId`**, so a wallet signs it while connected to any EVM chain. The chain is bound inside the message as `nativeChainId`. `authKind` is `1` and `authScope` is `0`; a transfer has no multi-signature or agent-key path.

```bash
curl -sS -X POST "https://api.native.org/trade" \
  -H 'content-type: application/json' \
  -d '{
    "action": {
      "type": "transfer",
      "to": "0x4017d1520d734c8fa6dd0fdeba150c3e0fea65ea",
      "asset_id": "2",
      "amount": "100000000",
      "cloid": "0x44c25ffe3c4f091bf28501543b63a59f"
    },
    "nonce": "1786607374550",
    "auth_scheme": "eip712",
    "signature": "0x…"
  }'
```

{% hint style="warning" %}
Three requirements have no field-level error to guide you:

* **`cloid` is required.** Omitting it returns `missing_cloid`, and a wrong length returns `invalid_cloid`. Sign with `cloidPresent: true` to match.
* **`auth_scheme: "eip712"` is required.** The binary signing scheme is rejected for `transfer` on every path.
* **The action carries no unknown fields.** An extra key is a parse error, not a silently ignored one.

A typed-data struct that does not match the definition above recovers a different signer, and the transfer fails as an unknown account rather than as a signature error.
{% endhint %}

This is Native Core's `/trade`, so it follows that endpoint's contract, not the Pool envelope: branch on `submission_status`, and treat `timeout` as unresolved rather than failed. See [POST /trade](../native-core/post-trade.md), [Transaction Signing](../native-core/transaction-signing.md), and [Handle outcomes & timeouts](../native-core/handle-timeouts.md).

### 3. Wait for the credit

An accepted transfer debits the Core balance immediately. The Pool credit follows once Core has moved past the transaction, usually within a block.

The row appears in `deposits` with `deposit_type: "direct_transfer"` and no transaction hashes — the Pool does not record one for this route. Match on the asset, the amount, and `core_event_timestamp_ms`, then use `operation_id` as the key from that point on. `operation_id` is assigned when the event is ingested and cannot be computed before you submit; your `cloid` is the handle until the row exists.

{% hint style="warning" %}
A direct transfer that cannot be credited leaves **no record at all**. Sending an asset the Pool has not listed, or one with `deposit_enabled: false`, moves the funds into the vault and produces nothing to poll: no row, no `rejected` status. Check `config` before you sign, and stop a client that is waiting for a row rather than letting it poll forever.
{% endhint %}

## Reading the deposit list

One query covers both routes and the full credited history.

| Parameter      | Default | Notes                                                       |
| -------------- | ------- | ----------------------------------------------------------- |
| `user_address` | —       | Required                                                     |
| `asset_id`     | all     | Optional filter                                              |
| `limit`        | `50`    | **Over `200` the request fails**, it is not clamped          |
| `before_id`    | —       | Cursor from the previous page's `next_before_id`             |

`status` only ever reads `credited` or `rejected`. The list is filtered by `user_address`, and ownership is only established at credit time, so nothing in flight is visible.

Three trace fields are **absent as keys** rather than null when they do not apply, so test with `if (row.src_tx_hash)`:

| Field          | Present on                          |
| -------------- | ----------------------------------- |
| `src_chain_id` | `bridge_deposit`                    |
| `src_tx_hash`  | `bridge_deposit`                    |
| `core_tx_hash` | `bridge_deposit`                    |

Paging returns `next_before_id` whenever a page comes back exactly `limit` long, so the last full page still carries a cursor. One extra request returning an empty page is what confirms the end.

## What can go wrong

| Symptom                                            | Cause                                                             | What to do                                                  |
| -------------------------------------------------- | ------------------------------------------------------------------ | ------------------------------------------------------------ |
| Deposit mined, never appears                        | `actionFlag` was not `1`, so it credited the trading balance instead | Check the balance on Native Core; the funds are not lost      |
| Deposit mined, never appears, `actionFlag` was `1`  | Below `minDepositAmountWei`, or not on the 8-decimal grid            | Neither is self-recoverable; report the source transaction hash |
| `status: "rejected"`                                | The asset is not listed for the Pool, or has `deposit_enabled: false` | Read `config` before building the transaction                |
| First deposit credited nothing                      | The activation fee underpaid past the 30% tolerance                  | Report the source transaction hash                           |
| Transfer returns `missing_cloid`                    | `cloid` omitted from the action body                                 | Send a 16-byte `cloid` and sign with `cloidPresent: true`     |
| Transfer accepted, no `direct_transfer` row         | The asset is not deposit-enabled for the Pool                        | Nothing will appear; report it with the Core transaction hash |
| `code: 131004` `limit exceeds max 200`              | `limit` above the cap                                                | The request failed entirely; resend with `limit` at most 200  |

## Next steps

{% content-ref url="yield.md" %}
[yield.md](yield.md)
{% endcontent-ref %}

{% content-ref url="withdraw.md" %}
[withdraw.md](withdraw.md)
{% endcontent-ref %}
