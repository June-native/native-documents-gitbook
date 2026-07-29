---
hidden: true
---

# GET All market maker settings

{% openapi src="../../../.gitbook/assets/nativeLend.json" path="/v1/lend/all-mm-settings" method="get" %}
[nativeLend.json](../../../.gitbook/assets/nativeLend.json)
{% endopenapi %}

#### Example response

```json
[
    {
        "mm_address": "0x54083336251a609e79c7f8ebb6180b7ef5f96402",
        "chains": [
            "base"
        ]
    },
    {
        "mm_address": "0xff8ba4d1fc3762f6154cc942ccf30049a2a0cec6",
        "chains": [
            "base"
        ]
    }
]
```
