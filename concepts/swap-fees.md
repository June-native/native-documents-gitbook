# Swap Fees

Swap fees are calculated in basis points (bps) of the input token and are included in both the returned order books and [firm quotes](firm-quote.md).

### Integration Partners

For integrators routing through [Native Relay](../modules/native-relay.md), we collaborate with partners such as aggregators, wallets, and other taker/flow sources to establish customized fee rates based on individual requirements.&#x20;

Typically, we set the swap fee to 0 or minimum to maintain a competitive offering in the market.

### Retail

For retail users swapping via the Native UI, the swap widget collects a fixed 85bps swap fee.
