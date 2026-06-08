# GET /firm-quote

This endpoint returns the final `amount_out` based on the source chain, source token, destination chain and destination token.

Currently, Native only supports ERC20 <> ERC20, and ERC20 <> Native token bridging. If you wish to bridge a native token to another token, you must first wrap it before calling the Native endpoint.

For native token convention, Native uses `0xEeeeeEeeeEeEeeEeEeEeeEEEeeeeEeeeeeeeEEeE` to represent the token address of the native token.

### **Headers**

| Name     | Description                           |
| -------- | ------------------------------------- |
| `apiKey` | API Key retrieved from the Native app |

### **Params**

| Name             | Description                                                                                                                                            |
| ---------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `src_chain` \*   | The blockchain name belonging to the token\_in address - Refer to the GET /chains API [here](/broken/pages/KIpzhb0fqQRUmnU6x1uU) for more information  |
| `dst_chain`\*    | The blockchain name belonging to the token\_out address - Refer to the GET /chains API [here](/broken/pages/KIpzhb0fqQRUmnU6x1uU) for more information |
| `token_in`\*     | Address of the token to be sold                                                                                                                        |
| `token_out`\*    | Address of the token to be bought                                                                                                                      |
| `amount_wei` \*  | Amount of token to be sold, in **wei** unit                                                                                                            |
| `from_address`\* | Address of the user that sells the `token_in`                                                                                                          |
| `to_address`     | Address of the user that receives the `token_out`. If empty, this address will be the same as `from_address`                                           |

### **Example Request**

```json
https://newapi.native.org/v1/bridge/firm-quote?token_in=0x80137510979822322193FC997d400D5A6C747bf7&token_out=0xEeeeeEeeeEeEeeEeEeEeeEEEeeeeEeeeeeeeEEeE&amount=0.00001&from_address=0x42d4e9ee3f725c84b7934e4fda64f2be0f803130&src_chain=scroll&dst_chain=ethereum>
```

### **Example Response**

```json
{
    "sellerToken": "0x80137510979822322193FC997d400D5A6C747bf7",
    "buyerToken": "0x0000000000000000000000000000000000000000",
    "srcChainId": 534352,
    "destChainId": 1,
    "amountIn": "10000000000000",
    "amountOut": "10211737661049",
    "amountOutEther": 0.000010211737661049,
    "estimateDuration": 43,
    "source": "p1.4",
    "quoteId": "0xeb7cb5fe8014d685b3ca338e30ae7976",
    "signer": "0x04f57b77cd9102a3774570de93008e9c57b9f146",
    "initiateDeadline": 1721789016,
    "fillDeadline": 1721789056,
    "refundBufferTime": 1800,
    "contract": {
        "swapper": "0x42d4e9ee3f725c84b7934e4fda64f2be0f803130",
        "order": {
            "pool": "0x9e2505ff3565d7c83a9cbcfd260c4a545780b402",
            "signer": "0x04f57b77cd9102a3774570de93008e9c57b9f146",
            "recipient": "0x90E3e403e0C4531471abFdc1FCCF402c1064F209",
            "sellerToken": "0x80137510979822322193FC997d400D5A6C747bf7",
            "buyerToken": "0x0000000000000000000000000000000000000000",
            "effectiveSellerTokenAmount": "10000000000000",
            "sellerTokenAmount": "10000000000000",
            "buyerTokenAmount": "10211737661049",
            "deadlineTimestamp": "1721789056",
            "nonce": "5182055928219179",
            "quoteId": "0xeb7cb5fe8014d685b3ca338e30ae7976",
            "multiHop": false,
            "signature": "0x",
            "widgetFee": {
                "signer": "0x0000000000000000000000000000000000000000",
                "feeRecipient": "0x0000000000000000000000000000000000000000",
                "feeRate": 0
            },
            "widgetFeeSignature": "0x",
            "externalSwapCalldata": "0x",
            "amountOutMinimum": "0"
        },
        "orderInput": {
            "token": "0x80137510979822322193FC997d400D5A6C747bf7",
            "initiateDeadline": "1721789016",
            "amount": "10000000000000"
        },
        "orderOutput": {
            "token": "0x0000000000000000000000000000000000000000",
            "amount": "10211737661049",
            "recipient": "0x42d4e9ee3f725c84b7934e4fda64f2be0f803130",
            "chainId": 1
        },
        "fallbackData": {
            "bridgeAddress": "0x0000000000000000000000000000000000000000",
            "bridgeCalldata": "0x"
        }
    },
    "txRequest": {
        "target": "0xe7b39EefDe6561952fF7A5E44596Dfcb35c07833",
        "calldata": "0xb1cad22900000000000000000000000042d4e9ee3f725c84b7934e4fda64f2be0f8031300000000000000000000000000000000000000000000000000000000066a06a5800000000000000000000000080137510979822322193fc997d400d5a6c747bf7000000000000000000000000000000000000000000000000000009184e72a0000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000009499afeee7900000000000000000000000042d4e9ee3f725c84b7934e4fda64f2be0f8031300000000000000000000000000000000000000000000000000000000000000001000000000000000000000000000000000000000000000000000000000000014000000000000000000000000000000000000000000000000000000000000004000000000000000000000000009e2505ff3565d7c83a9cbcfd260c4a545780b40200000000000000000000000004f57b77cd9102a3774570de93008e9c57b9f14600000000000000000000000090e3e403e0c4531471abfdc1fccf402c1064f20900000000000000000000000080137510979822322193fc997d400d5a6c747bf70000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000009184e72a000000000000000000000000000000000000000000000000000000009184e72a000000000000000000000000000000000000000000000000000000009499afeee790000000000000000000000000000000000000000000000000000000066a06a800000000000000000000000000000000000000000000000000012690d6acec62beb7cb5fe8014d685b3ca338e30ae79760000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000260000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000028000000000000000000000000000000000000000000000000000000000000002a00000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000400000000000000000000000000000000000000000000000000000000000000000",
        "value": "0"
    }
}
```
