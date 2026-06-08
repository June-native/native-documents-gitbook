# On-chain events for swap

* Market makers can monitor the pool address for the following event to detect successful trades on-chain.
* Match the event by the `quoteId.`

```
event RfqTrade(
    address recipient,
    address sellerToken,
    address buyerToken,
    uint256 sellerTokenAmount,
    uint256 buyerTokenAmount,
    bytes16 quoteId,
    address signer
);
```
