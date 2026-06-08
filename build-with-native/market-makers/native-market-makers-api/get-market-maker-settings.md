---
hidden: true
---

# GET Market maker settings

{% openapi src="../../../.gitbook/assets/nativeLend.json" path="/v1/lend/mm-settings" method="get" %}
[nativeLend.json](../../../.gitbook/assets/nativeLend.json)
{% endopenapi %}

#### Example response

```json
{
    "mm_address": "0x2447f272547760c66FB9758E9bB87F0EA0D6EDD1",
    "settler": "0x3Bdd75701dc1683f1229D77d5487A469Bae561e0",
    "chain": "ethereum",
    "mm_partner": "testPmm",
    "hard_limit_usd_threshold": 10000, //additional credit limit on top of collateral
    "token_amount_limit": null,
    "is_forced_paused": false, //force paused by admin from all activities
    "is_paused": false, //pause borrowing because of time delay of settlement or collateral removal
    "is_interest_paused": false
}
```
