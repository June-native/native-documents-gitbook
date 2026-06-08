---
hidden: true
---

# POST Liquidation

{% openapi src="../../../.gitbook/assets/nativeLend.json" path="/v1/lend/liquidation" method="post" %}
[nativeLend.json](../../../.gitbook/assets/nativeLend.json)
{% endopenapi %}

#### During time delay

```json
{
    "status": "pending",
    "readyAt": "2024-01-31T09:39:15.776Z",
    "expiresAt": "2024-01-31T09:40:15.776Z"
}
```

* **Note that for position updates:** A positive amount means you want to close a short position by depositing money in; a negative amount means you want to close a long position by withdrawing the money out.
* The claim collateral amount is always positive, indicating how much collateral you want to claim.
