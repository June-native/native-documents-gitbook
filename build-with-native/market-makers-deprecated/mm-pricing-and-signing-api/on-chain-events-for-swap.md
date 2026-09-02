# On-chain events for swap

{% hint style="danger" %}
This doc is deprecated, for market makers integrations, please refer to:

* [native-core](../../native-core/ "mention")
* [native-core](../../../modules/native-core/ "mention")
{% endhint %}

* Market makers should monitor the pool address for the following event to detect successful trades on-chain.
* Match the event by the `quoteId`

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
