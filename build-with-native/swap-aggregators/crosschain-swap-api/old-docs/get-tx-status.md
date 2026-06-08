# GET /tx-status

This endpoint returns the status of the bridge transaction, identified by `quoteId` that is returned from the Native firm quote endpoint.

**Bridge statuses**

* `USER_REQUESTED`: The user requested a firm quote but did not submit the initiate intent transaction.
* `USER_COMMITTED`: The user submitted the initiate intent and deposited funds into the Native bridge contract.
* `MM_FILLED`: The market maker (MM) filled the transaction on the destination chain; no further action is required from the user.
* `MM_FAILED`: The MM failed to sign the transaction within the expiry time; the user needs to perform a refund operation to retrieve their deposited funds.
* `REFUNDED`: The user performed a refund operation and withdrew their funds.
* `CLAIMED`: The MM claimed funds from the bridge contract to Native Lend, updating their credit position.

### **Headers**

| Name     | Description                           |
| -------- | ------------------------------------- |
| `apiKey` | API Key retrieved from the Native app |

### **Params**

| Name         | Description                                                              |
| ------------ | ------------------------------------------------------------------------ |
| `quoteId` \* | The unique order ID that is returned from the Native firm-quote endpoint |

### **Example Request**

```json
https://newapi.native.org/v1/bridge/tx-status?quoteId=0xeed84ac291ed96013ddeee89d2646cb0>
```

### **Example Response**

```json
{
    "quoteId": "0xeed84ac291ed96013ddeee89d2646cb0",
    "status": "MM_FILLED"
}
```
