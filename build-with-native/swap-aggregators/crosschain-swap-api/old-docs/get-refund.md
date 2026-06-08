# GET /refund

This endpoint returns the signed transaction allowing the user to perform a refund operation if the market maker (MM) does not fill the transaction within the expiry time.

Please note that refunds are only allowed after a specific interval following the user's initiation of the intent transaction. This measure is in place to protect both the user and the market maker, as block finality times vary across different chains.

### **Headers**

| Name     | Description                           |
| -------- | ------------------------------------- |
| `apiKey` | API Key retrieved from the Native app |

### **Params**

| Name         | Description                                                      |
| ------------ | ---------------------------------------------------------------- |
| `quoteId` \* | The unique order ID returned from the Native firm-quote endpoint |

### **Example Request**

```json
https://newapi.native.org/v1/bridge/refund?quoteId=0x8883078d82ae42ff81d7da62159400fa>
```

### **Example Response**

```json
{
    "token": "0x5300000000000000000000000000000000000004",
    "amount": "10000000000000",
    "swapper": "0x5ed8277cb583e16d0fE9899F7977d5A55cbd0Ea8",
    "contract": {
        "quoteId": "0x64597d172031592b1119d96c7270f4b8",
        "refundSignature": "0x1f3950485a70cbf0dd5a4184290b2dc951a34ee887eb251c666772787d521ae1077b8f0ae0b8a3426f972cccf735a063c91e0c6a1567eccc1caf67fc76f45d451b"
    },
    "txRequest": {
        "target": "0xe7b39EefDe6561952fF7A5E44596Dfcb35c07833",
        "calldata": "0xf9b068d764597d172031592b1119d96c7270f4b800000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000004000000000000000000000000000000000000000000000000000000000000000411f3950485a70cbf0dd5a4184290b2dc951a34ee887eb251c666772787d521ae1077b8f0ae0b8a3426f972cccf735a063c91e0c6a1567eccc1caf67fc76f45d451b00000000000000000000000000000000000000000000000000000000000000",
        "value": "0"
    }
}
```
