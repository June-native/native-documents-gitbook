---
description: One keyless read for the deposit routing table — the vault on each chain, and each token's address, decimals, asset_id, pause state and minimum deposit.
---

# Registry

`GET /api/v3/core/registry` returns, in one call, the vault on each EVM chain and every token it accepts that Native credits.

```bash
curl -sS https://api-ui.native.org/api/v3/core/registry
```

Keyless, no parameters, mainnet. The response uses the `{code, message, data}` envelope and always returns `code: 0`.

```json
{
  "code": 0,
  "message": "success",
  "data": {
    "updated_at": 1792496688145,
    "chains": [
      {
        "evmChainId": 56,
        "address": "0xd7AFd8FbEcC1AE7a4Ce5b93eb59D76F967D0Dea7",
        "enabled": true,
        "underlyings": [
          {
            "address": "0x55d398326f99059fF775485246999027B3197955",
            "decimals": 18,
            "assetId": 2,
            "nativeSymbol": "USDT",
            "enabled": true,
            "minDepositDecimal": 8,
            "minDepositAmountWei": "10000000000000000000"
          }
        ]
      }
    ]
  }
}
```

The response carries more fields than shown. Build only on the ones below.

| Field                 | Meaning                                                                         |
| --------------------- | ------------------------------------------------------------------------------- |
| chain `address`       | The vault: `approve` spender and `deposit()` target                             |
| token `address`       | The `token` argument to `deposit()`                                             |
| `decimals`            | The token's own decimals. `deposit()` amounts are in these units                |
| `assetId`             | The Native `asset_id` the deposit credits                                       |
| `enabled`             | `false` when deposits of this token are paused                                  |
| `minDepositDecimal`   | Floor the amount to this many decimal places                                    |
| `minDepositAmountWei` | Minimum deposit in the token's base units, as a string. `"0"` means no minimum  |

* The table refreshes about once an hour, so `enabled` can lag a pause. Read `isDepositPaused(token)` and `emergencyPaused()` on-chain before each deposit.
* An empty `chains` array means the table is temporarily unavailable. Retry later.
* [`accountingDepositContracts`](../native-core/post-info.md#accountingdepositcontracts) is the authoritative vault list and can include chains this table does not.
