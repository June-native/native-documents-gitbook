# POST Request collateral removal

{% openapi src="../../../.gitbook/assets/nativeLend.json" path="/v1/lend/request-collateral-removal" method="post" %}
[nativeLend.json](../../../.gitbook/assets/nativeLend.json)
{% endopenapi %}

#### Success response

```json
{
    "success": true,
    "id": "ffa84899-3c76-4e2b-a720-10c5d9bc0da0"
}
```

#### Error responses

```json
// Pending request
{
    "statusCode": 400,
    "message": "Pending request exists, please cancel previous request!",
    "error": "Bad Request"
}
```
