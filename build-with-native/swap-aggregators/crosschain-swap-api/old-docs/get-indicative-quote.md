# GET /indicative-quote

This endpoint returns an estimated `amount_out` based on the source chain, source token, destination chain and destination token.

Current, Native only supports ERC20 <> ERC20, and ERC20 <> Native token bridging. If you wish to bridge a native token to another token, you must first wrap it before calling the Native endpoint.

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
https://newapi.native.org/v1/bridge/indicative-quote?token_in=0x80137510979822322193FC997d400D5A6C747bf7&token_out=0xEeeeeEeeeEeEeeEeEeEeeEEEeeeeEeeeeeeeEEeE&amount=0.00001&from_address=0x42d4e9ee3f725c84b7934e4fda64f2be0f803130&src_chain=scroll&dst_chain=ethereum>
```

### **Example Response**

```json
{
    "sellerToken": "0x651810cad876b5959a57b22a2caf62d895857a74",
    "buyerToken": "0x79cdb5deef69b66a3a6ffc8656ba39182c1c069f",
    "srcChainId": 97,
    "destChainId": 11155111,
    "amountIn": "1000000",
    "amountOut": "999668",
    "amountOutEther": 0.999668,
    "estimateDuration": 60,
    "bridgeFeeUsd": 0,
    "source": "p1.4"
}
```
