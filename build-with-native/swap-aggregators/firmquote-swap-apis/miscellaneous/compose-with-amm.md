# Compose with AMM

### Flexible Input Amount

Since AMM swaps’ output amount is only fixed upon onchain execution, Native RFQ is designed to handle flexible input (+10% or -50% of the original quoted input amount) to felicitate multi-hop routing with AMMs or other liquidity sources.

You may also use this feature to perform: over-quote then under-fill in some solving cases.

To utilise this feature, there are two `uint256` in the final `tradeRFQT` interface:

* `actualSellerAmount`
* `actualMinOutputAmount`

By default they are `0`. Meaning the Native Router will execute the order with the exact input amount quoted. If the balance in your caller address is lower, the call will revert due to insufficient balance. This is common when the previous hop is AMM and the output amount is impacted by slippage.

To avoid this, simply override the `actualSellerAmount` in the contract logic when executing the call.

You may choose to leave `actualMinOutputAmount` as 0. Native Router should automatically adapt based on the original quoted slippage control parameter.

{% hint style="warning" %}
Please note that in some rare cases, when the actual amount is greater than the original quoted amount, the Native Credit Pool may fail to facilitate the swap. As the actual amount is now larger than the original quote. Being unexpected by the Native quoting backend.
{% endhint %}

### Locate the Parameters

To locate the parameters in calldata you may decode the call using the Native Router ABI (all are verified on explorers.) Or, using two fields in `/firm-quote` return:

* `amountInOffset`
* `amountOutMinimumOffset`

They indicate the offset position (in bytes) of the `actualSellerAmount` and `actualMinOutputAmount` parameter within the returned calldata.

### Examples

#### Swap Tx Examples

*   Native Swap without actual amount override:

    [Simulated Transaction | Tenderly](https://www.tdly.co/shared/simulation/4f93cb05-e1f6-487f-930b-b6ef2e28c319)
*   The same Native Swap but with actual amount override:

    [Simulated Transaction | Tenderly](https://www.tdly.co/shared/simulation/92009070-2de3-4b27-b27e-21b7bf205da4)

    `4629988946865908500000` → `4167453051074000000000` (-9.99% underfill)

#### Parameters Locating Example

From this decoded swap tx: [https://www.tdly.co/shared/simulation/a378d5e6-f3d4-439d-a203-798c94c05136](https://www.tdly.co/shared/simulation/a378d5e6-f3d4-439d-a203-798c94c05136)

We see the override value being `73684281000000000000`.

From `/firm-quote` the `amountInOffset` is `36`.

According to the offset value we may extract the encoded data from location 72 to 136 (64 characters), excluding the leading `0x`.

The extracted value is `000000000000000000000000000000000000000000000003fe93272c67109000` which is representing the `actualSellerAmount` value `73684281000000000000`
