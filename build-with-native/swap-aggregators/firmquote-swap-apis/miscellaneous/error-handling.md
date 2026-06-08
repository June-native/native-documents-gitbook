# Error Handling

## Error Handling

Requests with application or business logic errors will return with an HTTP code of 200, but with additional fields for error code and message. For example:

```json
{
  "code": 171008,
  "message": "Internal Server Error"
}

{
  "code": 171018,
  "message": "expire time exceeds global limit 120s"
}
```

To verify the response status, please check `success` bool, or if `code` is present in the response.

{% hint style="info" %}
Please note that for endpoints that typically return an array, when they encounter an error, the response will be in object format, for example /orderbook.
{% endhint %}

## Error Codes

The list of most common error codes returned by Swap APIs. For other unknown errors, please reach out to Native team members via an existing channel.

<table data-header-hidden><thead><tr><th width="96.18927001953125">Code</th><th width="229.49609375">Message</th><th>Reason</th><th>Mitigation</th></tr></thead><tbody><tr><td>301016<br>405030</td><td>quote invalid, risk management checks failed</td><td>The quote provided by the underlying private market maker is rejected by Native's risk managment system to protect swapper/taker.</td><td>This is usually caused by sudden, violent market movements and should never be persistent.<br><br>Simply try again after a few seconds of cooldown.</td></tr><tr><td>101010</td><td>quoted amount exceeds maximum available</td><td>The quoting amount exceeds the maximum available liquidity/depth represented by levels in the <code>/orderbook</code> endpoint.</td><td>Please always refer to the up-to-date <code>/orderbook</code> results while quoting. And quote within the available depth.</td></tr><tr><td>171037</td><td>orderbook empty for quoted trade</td><td>The quoting trade currently has 0 available liquidity/depth, and is removed from the <code>/orderbook</code> endpoint.<br>Please always refer to the latest <code>/orderbook</code> results for quoting.</td><td>Please always refer to the up-to-date <code>/orderbook</code> results while quoting. And only quote existing orderbooks.</td></tr><tr><td>171011</td><td>requested pair not found</td><td>The requested pair is not supported by Native and can’t be found.</td><td>Refer to <code>/orderbook</code> results while quoting. And only quote existing orderbooks.<br><br>Or contact Native to enable AMM fallback.</td></tr><tr><td>171015</td><td>quoted token not available</td><td>The requested quote/buying token is not available.</td><td>Refer to <code>/orderbook</code> results while quoting. And only quote existing orderbooks.<br><br>Or contact Native to enable AMM fallback.</td></tr><tr><td>101007</td><td>quoted pair not available, empty orderbook</td><td>The quoting trade currently has 0 available liquidity/depth, and is removed from the <code>/orderbook</code> endpoint.<br>Please always refer to the latest <code>/orderbook</code> results for quoting.</td><td>Please always refer to the up-to-date <code>/orderbook</code> results while quoting. And only quote existing orderbooks.</td></tr><tr><td>131003</td><td>failed to parse parameters</td><td>Issues with input parameters. Usually due to incorrect formats.</td><td>Check input format. Avoid passing floats in unsupported params like <code>amount_wei</code> or <code>timeout_millis</code></td></tr><tr><td>131004</td><td>invalid parameter</td><td>Issues with input parameters. Usually due to input conflicts.</td><td>Try avoid using duplicated params together like <code>amount</code> and <code>amount_wei</code></td></tr><tr><td>131011</td><td>front key version required</td><td><code>version</code> is required when calling Native’s public UI apiKey. Note that version must be ≥ 3.</td><td>Use <code>version=3</code> or above while quoting.</td></tr><tr><td>171018</td><td>expire time exceeds global limit 120s</td><td>The requested <code>expiry</code> should not exceed 120s.</td><td>Remove or pass <code>expiry_time</code> that is smaller than 120, recommend using 60 globally.</td></tr><tr><td>171053</td><td>fee bps exceeds limit 300</td><td>The commission fee should not exceed 300 bps.</td><td>Adjust <code>fee</code> to be 300 or lower.</td></tr><tr><td>131005</td><td>chain parameters invalid</td><td>Missing <code>chain</code> input.</td><td>Refer to <a data-mention href="../../../../resources/networks.md">networks.md</a>for supported networks. And use small-caps, standardised chain names.</td></tr><tr><td>201001</td><td>auth get api key is invalid</td><td>Auth error.</td><td>Use correct api key.</td></tr><tr><td>201005</td><td>rate reach limit</td><td>Rate limit is hit.</td><td>Reduce traffic.</td></tr></tbody></table>
