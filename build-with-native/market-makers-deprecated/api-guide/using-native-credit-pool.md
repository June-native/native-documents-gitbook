---
hidden: true
---

# Using Native Credit Pool

### Pre-requisite

This section assumes you have already integrated with Native as a market maker.

### Firm-Quote <a href="#firm-quote" id="firm-quote"></a>

When Native requests a Firm-Quote from the market maker, it will pass the `availableBorrowBalance` parameter, indicating the amount of available tokens you can borrow to fulfil the quote.

If you wish to use Native Credit Pool to fulfil the order, respond with `isBorrowing: true` in your Firm-Quote response.

### Signing the order <a href="#signing-the-order" id="signing-the-order"></a>

If you are using Native Credit Pool to fulfil the order, the `buyer` and `verifyingContract` fields when signing the order should be the Native Credit Pool address.

If you are using your own inventory, use the address of your own `RfqPool`.

### Orderbook

Market makers can publish a different orderbook for using Native Credit Pool versus their own inventory. This can be done by using different endpoints or the same endpoint with a different field in the response to distinguish between the two inventory sources.
