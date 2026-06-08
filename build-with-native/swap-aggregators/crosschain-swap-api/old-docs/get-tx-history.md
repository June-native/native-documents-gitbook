# GET /tx-history

This endpoint returns the transaction history initiated by a specific user.

#### **Headers**

| Name     | Description                           |
| -------- | ------------------------------------- |
| `apiKey` | API Key retrieved from the Native app |

#### **Params**

| Name             | Description                                                                          |
| ---------------- | ------------------------------------------------------------------------------------ |
| `userAddress` \* | The address of the trader who initiated the bridge transaction.                      |
| `limit`          | The number of transactions to be returned, with a minimum of 1 and a maximum of 5000 |

### **Example Request**

```json
https://newapi.native.org/v1/bridge/tx-history?userAddress=0x5ed8277cb583e16d0fE9899F7977d5A55cbd0Ea8&limit=100>
```

### **Example Response**

```json
{
  "data": [
    {
      "quoteId": "0xeed84ac291ed96013ddeee89d2646cb0",
      "sourceChain": "scroll",
      "sourceTxHash": "0xf28d4b638e7fdb9ea3321aaf879120a48957f8762d7873e0b65864c99f3ccc4d",
      "destChain": "ethereum",
      "destTxHash": "0xefc8ca8e1eafe77d22fa0ef6dc804f4443dd5f82b47ee1c1dc20ed50552a12fe",
      "sourceTokenIn": "0x5300000000000000000000000000000000000004",
      "destTokenOut": "0xc02aaa39b223fe8d0a0e5c4f27ead9083c756cc2",
      "sourceAmountIn": 0.0001,
      "destAmountOut": 0.000099979466177244,
      "sourceTokenInUsdValue": 0.347364,
      "destTokenOutUsdValue": 0.34729267289192184,
      "status": "MM_FILLED",
      "intentCreatedAt": "2024-07-22T07:10:05.847Z"
    },
  ],
  "hasNextPage": false,
  "nextCursor": ""
}
```
