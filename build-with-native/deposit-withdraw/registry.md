---
description: One keyless read for the deposit routing table — the vault on each chain, and for each token its address, decimals, Native asset_id, pause state and minimum deposit.
---

# Registry

`GET /api/v3/core/registry` returns the deposit routing table in one call. For each EVM chain it gives the vault address. For each token that vault accepts, it gives the ERC20 address, its decimals, the Native `asset_id` the deposit credits, whether deposits are open, and the minimum deposit.

It replaces a loop of per-token vault reads and a join against `POST /info` [`assets`](../native-core/post-info.md#assets). It does not replace the pause checks in [Deposit](deposit.md#4-approve-and-deposit): read those on-chain immediately before you build the transaction.

## Request

```bash
curl -sS https://api-ui.native.org/api/v3/core/registry
```

The endpoint is keyless, takes no parameters, and serves mainnet. It answers with the same `{code, message, data}` envelope as [the withdrawable ceiling](withdraw.md#the-withdrawable-ceiling), and always returns `code: 0`.

## Response

Trimmed to one chain and two tokens:

```json
{
  "code": 0,
  "data": {
    "updated_at": 1792496688145,
    "network": "mainnet",
    "global": {
      "coreChainId": 696969,
      "depositExpiresAfterMaxMs": 300000,
      "withdrawDeadlineMaxMs": 300000,
      "depositRollingWindowUsdCap": "1000000",
      "withdrawRollingWindowUsdCap": "200000"
    },
    "chains": [
      {
        "chainKey": "bsc",
        "label": "BNB Smart Chain",
        "evmChainId": 56,
        "version": "v2-nlp",
        "address": "0xd7AFd8FbEcC1AE7a4Ce5b93eb59D76F967D0Dea7",
        "statusMode": "contract",
        "enabled": true,
        "underlyings": [
          {
            "symbol": "WBNB",
            "nativeSymbol": "BNB",
            "address": "0xbb4CdB9CBd36B01bD1cBaEBF2De08d9173bc095c",
            "decimals": 18,
            "assetId": 4,
            "iconSlug": "bnb",
            "wrappedNLP": "0x85F08a266758eA0C98B15DEae71CA6eE84392AFA",
            "minDepositDecimal": 8,
            "enabled": true,
            "minDepositAmountWei": "16000000000000000"
          },
          {
            "symbol": "USDT",
            "nativeSymbol": "USDT",
            "address": "0x55d398326f99059fF775485246999027B3197955",
            "decimals": 18,
            "assetId": 2,
            "iconSlug": "usdt",
            "wrappedNLP": "0xEA5FF211eF700DccC521a1e6501C9fe1B95D8EE7",
            "minDepositDecimal": 8,
            "enabled": true,
            "minDepositAmountWei": "10000000000000000000"
          }
        ]
      }
    ]
  },
  "message": "success"
}
```

### Chain

| Field         | Type    | Meaning                                                                                  |
| ------------- | ------- | ---------------------------------------------------------------------------------------- |
| `evmChainId`  | number  | EVM chain id                                                                             |
| `label`       | string  | Display name                                                                             |
| `address`     | string  | `DepositWithdrawVault` on this chain: the `approve` spender and the `deposit()` target   |
| `enabled`     | boolean | `true` when at least one token in `underlyings` is open for deposit                      |
| `underlyings` | array   | Tokens this vault accepts and Native credits                                             |

### Underlying

| Field                 | Type    | Meaning                                                                                                                       |
| --------------------- | ------- | ----------------------------------------------------------------------------------------------------------------------------- |
| `address`             | string  | ERC20 contract: the `token` argument to `deposit()`                                                                           |
| `decimals`            | number  | The ERC20's own decimals. `deposit()` amounts are in these units                                                              |
| `assetId`             | number  | The Native `asset_id` the deposit credits. It is the same id as in [`assets`](../native-core/post-info.md#assets) and in the [`deposits`](../native-core/post-info.md#deposits) credit record |
| `symbol`              | string  | The token's on-chain `symbol()`. Empty when the token does not return a readable string                                       |
| `nativeSymbol`        | string  | The asset's symbol on Native. It can differ from `symbol`: a WBNB deposit credits `BNB`, a WETH deposit credits `ETH`          |
| `enabled`             | boolean | `false` when deposits of this token are paused, or when the pause state could not be read                                    |
| `minDepositAmountWei` | string  | Minimum deposit, in the token's own base units. `"0"` means no minimum                                                        |
| `minDepositDecimal`   | number  | The decimal grid Native credits on. Floor the amount to this many decimal places                                             |
| `wrappedNLP`          | string  | The wrapped-NLP token the vault mints against the deposit                                                                     |

`minDepositAmountWei` is a string because an 18-decimal minimum is larger than a JavaScript number can hold exactly. Parse it as a `BigInt`.

`minDepositDecimal` and `wrappedNLP` are absent when their vault read failed. Read them from the vault instead: [`minDepositDecimalByUnderlying`](vault-contract.md#reading) and [`wrappedNLPByUnderlying`](vault-contract.md#reading).

`chainKey`, `version`, `statusMode` and `iconSlug` serve the Native web app. Do not build on them.

### Global

`coreChainId` is the Native chain id: the `nativeChainId` you sign a [withdrawal](withdraw.md#2-sign-and-submit) with. The other `global` fields are operator settings. Do not build on them.

`global` is absent when its read failed.

## Freshness and coverage

* The table is rebuilt about **once an hour**. `updated_at` is the Unix millisecond time it was built. Cache it on your side.
* `minDepositAmountWei` is the exception: it is current to within 30 seconds.
* `enabled` can lag a pause by up to an hour. Before each deposit, read `isDepositPaused(token)` and `emergencyPaused()` on-chain. The chain-level `enabled` does not reflect `emergencyPaused()`.
* An empty `chains` array means the table is not available right now, not that deposits are closed. Retry later.
* A token appears only when Native credits it. A token the vault accepts but Native has not listed is left out.
* The table can list fewer chains than [`accountingDepositContracts`](../native-core/post-info.md#accountingdepositcontracts), which stays the authoritative set of vaults. For a chain that is not listed here, take the vault from `accountingDepositContracts` and read its tokens from the vault.

## Use it for a deposit

Each [Deposit](deposit.md) step reads one part of a token's entry.

| Deposit step                                                  | Fields                                         |
| ------------------------------------------------------------- | ---------------------------------------------- |
| [3. Validate the amount](deposit.md#3-validate-the-amount)    | `decimals`, `minDepositDecimal`, `minDepositAmountWei` |
| [4. Approve and deposit](deposit.md#4-approve-and-deposit)    | chain `address`, token `address`, `enabled`    |
| [6. Wait for the credit](deposit.md#6-wait-for-the-credit)    | `assetId`, to match the credit's `asset_id`    |

```ts
const res = await fetch('https://api-ui.native.org/api/v3/core/registry').then(r => r.json())
const chain = res.data.chains.find(c => c.evmChainId === 56)
const token = chain.underlyings.find(u => u.assetId === 2)        // USDT on BNB Smart Chain

const scale = 10n ** BigInt(Math.max(0, token.decimals - (token.minDepositDecimal ?? 8)))
const amount = requested - (requested % scale)                    // floor to the grid
if (amount < BigInt(token.minDepositAmountWei)) throw new Error('below the minimum deposit')
if (!token.enabled) throw new Error('deposits of this token are paused')
```
