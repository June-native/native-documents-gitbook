---
description: Get every deposit vault and the tokens it accepts in one request.
---

# Registry

`GET /api/v3/core/registry` returns the deposit vault on each EVM chain and the tokens each vault accepts.

```bash
curl -sS https://api-ui.native.org/api/v3/core/registry
```

No API key needed. No parameters. Mainnet only. The HTTP status is always `200` and `code` is always `0`.

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

The example leaves out fields that only the Native web app uses.

Each item in `chains`:

| Field         | Meaning                    |
| ------------- | -------------------------- |
| `evmChainId`  | EVM chain ID               |
| `address`     | Vault address              |
| `underlyings` | Tokens this vault accepts  |

Each item in `underlyings`:

| Field                 | Meaning                                                                       |
| --------------------- | ----------------------------------------------------------------------------- |
| `address`             | Token address                                                                 |
| `decimals`            | Token decimals                                                                |
| `assetId`             | Asset ID on Native Core                                                       |
| `nativeSymbol`        | Asset symbol on Native Core                                                   |
| `enabled`             | `true` if deposits of this token are open                                     |
| `minDepositDecimal`   | Maximum decimal places of a deposit amount                                    |
| `minDepositAmountWei` | Minimum deposit in the token's smallest unit, as a string. `"0"` means no minimum |

`updated_at` is the last update time, in Unix milliseconds.

* The data is updated about once an hour, so `enabled` can be out of date. Before each deposit, read `isDepositPaused(token)` and `emergencyPaused()` from the vault.
* If `chains` is empty, the data has not loaded yet. Try again later.
* Not every chain is listed here yet. For the full list of vaults, use [`accountingDepositContracts`](../native-core/post-info.md#accountingdepositcontracts).
