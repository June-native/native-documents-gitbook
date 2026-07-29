# POST Request settlement

{% openapi src="../../../.gitbook/assets/nativeLend.json" path="/v1/lend/request-settlement" method="post" %}
[nativeLend.json](../../../.gitbook/assets/nativeLend.json)
{% endopenapi %}

#### Example response

```json
{
    "id": "d49c39ed-ff63-41c0-bf3a-75f6d56f7346",
    "nonce": "6168514449590345",
    "recipient": "0x9B85B4A413Efe69684290816a3E814B4aA1EFf63",
    "chain": "base",
    "positionUpdates": [
        {
            "tokenAddress": "0x4200000000000000000000000000000000000006",
            "amount": "-5000000000"
        }
    ],
    "creditChangeUsd": -0.00001168615,
    "trader": "0x9B85B4A413Efe69684290816a3E814B4aA1EFf63",
    "requestAt": "2024-01-31T08:09:22.645Z",
    "enableAt": "2024-01-31T08:14:22.645Z",
    "expiresAt": "2024-01-31T08:15:22.645Z",
    "type": "settlement"
}
```

#### Error responses

```json
// Try to withdraw more than long position amount
{
    "statusCode": 400,
    "message": "Cannot make long position to become short position",
    "error": "Bad Request"
}

// Try to deposit into long position
{
    "statusCode": 400,
    "message": "Cannot increasing existing long position",
    "error": "Bad Request"
}

// Try to deposit more than short position amount
{
    "statusCode": 400,
    "message": "Cannot make short position to become long position",
    "error": "Bad Request"
}

// Try to withdraw from short position
{
    "statusCode": 400,
    "message": "Cannot decreasing existing short position",
    "error": "Bad Request"
}

// Invalid token address
{
    "statusCode": 400,
    "message": "Cannot settle unexist position",
    "error": "Bad Request"
}

// Invalid mm
{
    "statusCode": 404,
    "message": "MM Settings not found",
    "error": "Not Found"
}
```
