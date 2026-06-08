# Firm-Quote

Based on the order book you provide, our router will determine the optimal liquidity source to offer the best price. If your pricing is selected, Native will request a firm quote from you.

The firm quote endpoint must adhere to the following format:

### **Endpoint**

```
GET <market_maker_base_url>/firm-quote
```

### **Params**

<table><thead><tr><th width="291">Name</th><th>Description</th></tr></thead><tbody><tr><td><code>sellerTokenAmount</code></td><td>The token input amount of the order, in wei</td></tr><tr><td><code>chainId</code></td><td>Chain ID of the network (e.g., 1 for Ethereum, 56 for BSC)</td></tr><tr><td><code>sellerToken</code></td><td>The ERC20 token address that will be sent to the market maker</td></tr><tr><td><code>buyerToken</code></td><td>The ERC20 token address that will be received from the market maker</td></tr><tr><td><code>pool</code></td><td>Native pool contract address that will execute this order.</td></tr><tr><td><code>nativePool</code></td><td>The Native Rfq Pool address that execute the trade.</td></tr><tr><td><code>quoteId</code></td><td>Unique ID for this order request</td></tr><tr><td><code>deadlineTimestamp</code></td><td>Deadline UNIX timestamp specified by the user. You may double-check if the number is acceptable. And choose to reject any quotes with invalid numbers.<br><br>However, if a Market Maker wishes to set a maximum allowed expiry, please include that in the orderbook endpoint.</td></tr><tr><td><code>feeBps</code></td><td>Fee that Native will charge market makers, factored into their quote.<br><br>Native collaborates with market makers to track and pay fees off-chain. Contracts with market makers will specify how fees are set and paid, and data sources will be provided for monitoring fee obligations.</td></tr><tr><td><code>availableBorrowBalance</code></td><td>The amount of buyer token Market Makers can borrow from Native Inventory at the moment, in wei</td></tr></tbody></table>

#### **Example**

```plaintext
<market_maker_base_url>/firm-quote?sellerTokenAmount=10000000000000000000000&chainId=97&sellerToken=0x8ac76a51cc950d9822d68b83fe1ad97b32cd580d&buyerToken=0x55d398326f99059ff775485246999027b3197955&nativePool=0xaaE854bdd940cf402d79e8051DC7E3390e32A3ac&quoteId=62716-206e-41cb-8559-013f1ed1a65a&deadlineTimestamp=1671086729&feeBps=12&availableBorrowBalance=10000000000000000000000
```

In this example, a user sells **10,000 USDC** to the market maker in exchange for **USDT** on **Binance Smart Chain (BSC)**.

### **Response**

You must provide a response in the following format:

<table><thead><tr><th width="225">Name</th><th>Description</th></tr></thead><tbody><tr><td><code>success</code></td><td>Indicates whether you can provide a firm quote for this order flow (true/false)</td></tr><tr><td><code>buyerTokenAmount</code></td><td>The amount of tokens you are exchanging with the seller</td></tr><tr><td><code>deadlineTimestamp</code></td><td>The timestamp when this order will expire, typically 20-30 seconds for execution</td></tr><tr><td><code>errorMessage</code></td><td>Include the reason for the error if <code>success</code> is false.</td></tr></tbody></table>

#### **Example**

```json
{
  "success": true,
  "buyerTokenAmount": "10100000000000000000000",
  "deadlineTimestamp": 1671086729, // Will be passed to sign-quote
  "errorMessage": ""
}
```

In the response above, the market maker returns **10,100 USDT** for the order sent in the example.

{% hint style="info" %}
The Native server is hosted in the AWS Tokyo region, and all market makers are required to respond to all endpoints in 100ms (end-to-end).
{% endhint %}
