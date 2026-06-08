# POST Cancel settlement

{% openapi src="../../../.gitbook/assets/nativeLend.json" path="/v1/lend/cancel-settlement" method="post" %}
[nativeLend.json](../../../.gitbook/assets/nativeLend.json)
{% endopenapi %}

#### Example response

```json
{
    "status": "success",
    "id": "b9292014-1e85-43e8-a635-8a4f1c5423b9"
}
```

#### Error responses

```json
{
    "statusCode": 400,
    "message": "Settlement request not found",
    "error": "Bad Request"
}
```
