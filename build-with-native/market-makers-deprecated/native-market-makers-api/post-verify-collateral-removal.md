---
hidden: true
---

# POST Verify collateral removal

{% openapi src="../../../.gitbook/assets/nativeLend.json" path="/v1/lend/verify-collateral-removal" method="post" %}
[nativeLend.json](../../../.gitbook/assets/nativeLend.json)
{% endopenapi %}

#### Success response

```json
{
    "id": "673a69b0-06db-4309-b0f1-6582435b80ca",
    "nonce": "6885024828728157",
    "recipient": "0x9B85B4A413Efe69684290816a3E814B4aA1EFf63",
    "chain": "base",
    "tokens": [
        {
            "tokenAddress": "0x5d55432c6aAeDB4D34523B2744e959F03aEffFE3",
            "amount": "10000000000000000000"
        }
    ],
    "totalCollateralRemovedUsd": 9.778953854000001e-12,
    "trader": "0x9B85B4A413Efe69684290816a3E814B4aA1EFf63",
    "requestAt": "2024-01-31T09:17:41.847Z",
    "enableAt": "2024-01-31T09:22:41.847Z",
    "expiresAt": "2024-01-31T09:23:41.847Z",
    "type": "collateralRemoval"
}
```

#### Error responses

```json
// Invalid token
{
    "statusCode": 400,
    "message": "Collateral does not exist for token 0x12345",
    "error": "Bad Request"
}

// Insufficient collateral
{
    "statusCode": 400,
    "message": "Collateral amount is not enough for token 0x5d55432c6aAeDB4D34523B2744e959F03aEffFE3",
    "error": "Bad Request"
}

// Invalid mm
{
    "statusCode": 404,
    "message": "MM Settings not found",
    "error": "Not Found"
}
```
