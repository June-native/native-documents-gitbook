# Sign-Quote

After validating the firm quote, Native will request you to sign the quote, authorizing the swap for the approved amount. The sign quote endpoint must adhere to the following format:

### Endpoint <a href="#sign-quote-endpoint" id="sign-quote-endpoint"></a>

```
POST <base-url>/sign-quote
```

### **Params** <a href="#params" id="params"></a>

<table><thead><tr><th width="238">Name</th><th>Description</th></tr></thead><tbody><tr><td><code>nonce</code></td><td>ID to prevent the transaction from being executed more than once. Incremental for each transaction by <code>txOrigin</code> and <code>pool</code>.</td></tr><tr><td><code>signer</code></td><td>Public address of the signer who will sign this order</td></tr><tr><td><code>buyerToken</code></td><td>The ERC20 token address that will be received from the market maker</td></tr><tr><td><code>sellerToken</code></td><td>The ERC20 token address that will be sent to the market maker</td></tr><tr><td><code>buyerTokenAmount</code></td><td>The token output amount of the order, in wei. Can be modified by the signer to determine the final output amount.</td></tr><tr><td><code>sellerTokenAmount</code></td><td>The token input amount of the order, in wei</td></tr><tr><td><code>deadlineTimestamp</code></td><td>Deadline UNIX timestamp specified by the user. You may double-check if the number is acceptable. And choose to reject any quotes with invalid numbers.<br><br>However, if a Market Maker wishes to set a maximum allowed expiry, please include that in the orderbook endpoint.</td></tr><tr><td><code>quoteId</code></td><td>Unique ID for this order request</td></tr><tr><td><code>chainId</code></td><td>Chain ID of the network (e.g., 1 for Ethereum, 56 for BSC)</td></tr><tr><td><code>nativePool</code></td><td>The Native Rfq Pool address that execute the trade.</td></tr><tr><td><code>availableBorrowBalance</code></td><td>The amount of buyer token Market Makers can borrow from Native Inventory at the moment, in wei</td></tr></tbody></table>

#### **Example**

```json
{
    "nonce": 0,
    "signer": "0xf39Fd6e51aad88F6F4ce6aB8827279cffFb92266",
    "buyerToken": "0x55d398326f99059ff775485246999027b3197955", 
    "sellerToken": "0x8ac76a51cc950d9822d68b83fe1ad97b32cd580d", 
    "buyerTokenAmount": "10100000000000000000000", 
    "sellerTokenAmount": "10000000000000000000000",
    "deadlineTimestamp": "1671086729", 
    "quoteId": "62716-206e-41cb-8559-013f1ed1a65a",
    "chainId": 56,
    "nativePool": "0x9aF2F3c0cD35283e13f7087e2b34b1444b57A44C"
}
```

In this example, a user requests to sign an order of 10,000 USDC for 10,100 USDT on Binance Smart Chain (BSC) to be transacted on the market maker's Native pool (`0xaaE854bdd940cf402d79e8051DC7E3390e32A3ac`). The order must be signed by the signer (`0xf39Fd6e51aad88F6F4ce6aB8827279cffFb92266`) for successful on-chain execution.

### Order signing

Native uses EIP-712 signatures to ensure the order is endorsed only by the assigned signer. The inputs required to sign the order correspond to the data sent in the `POST sign-quote` request. The code snippet below generates bytes encoded in hexadecimal to be returned in response to the `POST /sign-quote` request.

```javascript
async function signOrderNative(data) {
    const wallet = new ethers.Wallet(privateKey, provider);
    const order = {
        nonce: data.nonce,
        signer: data.signer,
        buyerToken: data.buyerToken,
        sellerToken: data.sellerToken,
        buyerTokenAmount: data.buyerTokenAmount,
        sellerTokenAmount: data.sellerTokenAmount,
        deadlineTimestamp: data.deadlineTimestamp,
        quoteId: data.quoteId
    };
    const signature = await generateSignature(order, wallet, data.nativePool, data.chainId);
    return {
        success: true,
        signature: signature,
        order: order
    };
}

async function generateSignature(data, wallet, nativePool, chainId) {
    const signatureData = {
        types: {
            Order: [
                { name: "nonce", type: "uint256" },
                { name: "signer", type: "address" },
                { name: "buyerToken", type: "address" },
                { name: "sellerToken", type: "address" },
                { name: "buyerTokenAmount", type: "uint256" },
                { name: "sellerTokenAmount", type: "uint256" },
                { name: "deadlineTimestamp", type: "uint256" },
                { name: "decayStartTime", type: "uint256" },
                { name: "decayExponent", type: "uint256" },
                { name: "maxSlippageBps", type: "uint256" },
                { name: "quoteId", type: "bytes16" }
            ],
        },
        primaryType: 'Order',
        domain: {
            name: "Native RFQ Pool",
            version: "1",
            verifyingContract: nativePool,
            chainId
        },
        message: {
            nonce: data.nonce,
            signer: data.signer,
            buyerToken: data.buyerToken,
            sellerToken: data.sellerToken,
            buyerTokenAmount: data.buyerTokenAmount,
            sellerTokenAmount: data.sellerTokenAmount,
            deadlineTimestamp: data.deadlineTimestamp,
            decayStartTime: 0,
            decayExponent: 0,
            maxSlippageBps: 0,
            quoteId: Buffer.from(data.quoteId.replace(/-/g, ""), "hex")
        },
    };

    const signature = await wallet._signTypedData(signatureData.domain, signatureData.types, signatureData.message);
    return signature;
}
```

### **Response**

The response must be in the following format:

<table><thead><tr><th width="179">Name</th><th>Description</th></tr></thead><tbody><tr><td><code>success</code></td><td>Indicates whether you can sign the firm quote (true/false)</td></tr><tr><td><code>signature</code></td><td>The signature from signing the EIP-712 order object</td></tr><tr><td><code>order</code></td><td>The order object that contains the order details</td></tr><tr><td><code>errorMessage</code></td><td>Include the reason for the error if <code>success</code> is false.</td></tr></tbody></table>

#### **Example**

```json
{
  "success": true,
  "signature": "0x4547e11611094f4295c4175a3266ebd98028edfb30d29a1dc0c42b8bf71401ef4e61c41a4f7aa5e536a6118273d22ed9efe5139005a750745994de6771e273181c",
  "order": {
    "nonce": 1, // Same as the nonce 
    "signer": "0xf39Fd6e51aad88F6F4ce6aB8827279cffFb92266",
    "buyerToken": "0x55d398326f99059ff775485246999027b3197955",
    "sellerToken": "0x8ac76a51cc950d9822d68b83fe1ad97b32cd580d",
    "buyerTokenAmount": "10100000000000000000000",
    "sellerTokenAmount": "10000000000000000000000",
    "deadlineTimestamp": "1671086729",
    "quoteId": "62716-206e-41cb-8559-013f1ed1a65a"
  },
  "errorMessage": ""
}
```

In the response above, the market maker returns the generated signature and the corresponding order object to Native. With the signature, the customer can submit the transaction to execute the swap.

{% hint style="info" %}
**Note**

* The signature must start with `0x` and have a total of 132 characters.
{% endhint %}

{% hint style="info" %}
The Native server is hosted in the AWS Tokyo region, and all market makers are required to respond to all endpoints in 100ms (end-to-end).
{% endhint %}
