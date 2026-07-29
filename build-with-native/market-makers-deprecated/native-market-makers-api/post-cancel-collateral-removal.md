# POST Cancel collateral removal

{% openapi src="../../../.gitbook/assets/nativeLend.json" path="/v1/lend/cancel-collateral-removal" method="post" %}
[nativeLend.json](../../../.gitbook/assets/nativeLend.json)
{% endopenapi %}

#### Success response

```json
{
    "success": true
}
```

#### Error responses

```json
{
    "statusCode": 400,
    "message": "Collateral removal request not found",
    "error": "Bad Request"
}
```
