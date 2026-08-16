---
description: Move funds into a Native Core Pool earning balance, from an EVM chain or from an existing Native Core balance.
---

# Deposit

Two routes lead into the same `earn_balance`. Choose the route by where the funds are now.

|                | From an EVM chain                              | From a Native Core balance             |
| -------------- | ---------------------------------------------- | -------------------------------------- |
| You send       | `deposit()` on the vault contract               | A signed `transfer` action             |
| Signed with    | An EVM transaction                              | EIP-712 typed data                     |
| Settles in     | Minutes                                         | About one Core block                   |
| `deposit_type` | `bridge_deposit`                                | `direct_transfer`                      |

Neither route can be cancelled or reversed once the first transaction lands.

Read [Native Core Pool](README.md) first for the base URL, the response envelope, and discovery.

## Which assets you can deposit

An asset accepts deposits when it appears in `config.assets[]` with `deposit_enabled: true`.

```bash
curl -sS "$POOL_API_URL/api/v3/earn" \
  -H 'content-type: application/json' \
  -d '{"type":"config"}'
```

## From an EVM chain

**This is the Native Core deposit flow, with one argument changed.** Follow [Deposit](../deposit-withdraw/deposit.md): check whether the account exists, size the activation fee, approve and call the vault, then read the deposit nonce from the receipt. Every rule on that page applies here.

`actionFlag` is that argument, and it also decides where the deposit lands and where you confirm it:

|                 | Native Core deposit                 | Native Core Pool deposit           |
| --------------- | ----------------------------------- | ---------------------------------- |
| `actionFlag`    | `0`                                 | **`1`**                            |
| Credited to     | The Native Core trading balance     | `earn_balance`                     |
| Confirmed with  | `POST /info` `deposits`             | Pool `deposits`, below             |

```solidity
function deposit(address token, uint256 amount, uint256 actionFlag)
    external payable returns (uint256 wNLPAmount);
```

{% hint style="warning" %}
Passing `actionFlag: 0` produces a transaction that succeeds and credits the Native Core trading balance instead. The funds are not lost, but they are not earning, and no Pool record explains why.
{% endhint %}

`depositFor` carries the same argument, so a wallet or custodian depositing on a user's behalf passes `actionFlag: 1` the same way.

### Confirm the credit

Poll the Pool's `deposits` and match your source transaction hash.

```bash
curl -sS "$POOL_API_URL/api/v3/earn" \
  -H 'content-type: application/json' \
  -d '{"type":"deposits","user_address":"0x5555…5555","limit":20}'
```

```json
{
  "code": 0,
  "data": {
    "items": [
      {
        "id": 9,
        "operation_id": "deposit:1:0x3333…3333:31",
        "deposit_type": "bridge_deposit",
        "asset_id": 2,
        "amount": "10000000000",
        "status": "credited",
        "core_height": 153570000,
        "core_event_timestamp_ms": 1786620000000,
        "src_chain_id": 1,
        "src_tx_hash": "0xaaaa…aaaa",
        "core_tx_hash": "0xcccc…cccc"
      }
    ],
    "next_before_id": null
  },
  "message": "success"
}
```

`status: "credited"` means the amount is in `earn_balance` and earning. `status: "rejected"` means the amount is not credited and the funds require manual recovery from Native.

`amount` is in the asset's 8-decimal `balance_decimals`, rescaled from the token's own decimals: `100000000` sent as 6-decimal USDT arrives as `10000000000`.

The deposit becomes visible when it is credited, and not before. Hold your own record between submitting the transaction and seeing it appear, because the record does not exist until it reaches `credited` or `rejected`. A deposit that has not appeared long after the source transaction confirmed should be reported with its source transaction hash rather than retried.

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
  cloid: '0x6666…6666',
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
      "to": "0x1111…1111",
      "asset_id": "2",
      "amount": "100000000",
      "cloid": "0x6666…6666"
    },
    "nonce": "1786607374550",
    "auth_scheme": "eip712",
    "signature": "0x…"
  }'
```

A typed-data struct that does not match the definition above recovers a different signer, so a mistake there surfaces as an unknown account rather than as a signature error.

The `transfer` action goes to Native Core's `/trade`, which returns the Native Core envelope rather than the Pool envelope. Branch on `submission_status` and treat `timeout` as unresolved rather than failed. See [POST /trade](../native-core/post-trade.md), [Transaction Signing](../native-core/transaction-signing.md), and [Handle outcomes & timeouts](../native-core/handle-timeouts.md).

### 3. Wait for the credit

An accepted transfer debits the Core balance immediately. The Pool credit lands about one Core block later. Treat that as typical latency rather than a guarantee, and poll for the row.

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
| Deposit mined, never appears, `actionFlag` was `1`  | The amount or the activation fee broke a rule on the Native Core deposit page | See [What can go wrong](../deposit-withdraw/deposit.md#what-can-go-wrong) there |
| `status: "rejected"`                                | The asset is not listed for the Pool, or has `deposit_enabled: false` | Read `config` before building the transaction                |
| Transfer returns `missing_cloid` or `invalid_cloid` | `cloid` omitted, or not 16 bytes                                     | Send a 16-byte `cloid` and sign with `cloidPresent: true`     |
| Transfer rejected for its signing scheme            | The binary scheme was used; `transfer` accepts only `auth_scheme: "eip712"` | Sign the typed data above and post the 65-byte signature |
| Transfer returns a parse error                      | The action carried a field not listed above                          | Send only `type`, `to`, `asset_id`, `amount` and `cloid`      |
| Transfer accepted, no `direct_transfer` row         | The asset is not deposit-enabled for the Pool                        | Nothing will appear; report it with the Core transaction hash |
| `code: 131004` `limit exceeds max 200`              | `limit` above the cap                                                | The request failed entirely; resend with `limit` at most 200  |

## Next steps

{% content-ref url="yield.md" %}
[yield.md](yield.md)
{% endcontent-ref %}

{% content-ref url="withdraw.md" %}
[withdraw.md](withdraw.md)
{% endcontent-ref %}
