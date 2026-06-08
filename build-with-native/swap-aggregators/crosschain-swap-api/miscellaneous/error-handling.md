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
Please note that for endpoints that typically return an array, when they encounter an error, the response will be in object format.
{% endhint %}

## Error Codes

The list of most common error codes returned by CrossChain Swap APIs. For other unknown errors, please reach out to Native team members via an existing channel.

<table data-header-hidden><thead><tr><th width="96.18927001953125">Code</th><th width="229.49609375">Message</th><th>Reason</th><th>Mitigation</th></tr></thead><tbody><tr><td>351015</td><td>requested amount smaller than token out minimum wei [xxx]</td><td>The buying (bridge to) token amount is too small.</td><td>Check <a data-mention href="minimum-bridge-to-amount.md">minimum-bridge-to-amount.md</a> for the minimum amount and avoid bridging orders that are too small.</td></tr><tr><td>351003</td><td>unable to bridge: insufficient bridging liquidity</td><td>The quote request exceeds the maximum available bridging liquidity at the moment.</td><td>This is expected to be common (non-error) for upstreams that quote optimistically.<br><br>Simply try again later, and ensure this will not trick circuit breakers.</td></tr><tr><td>131003</td><td>failed to parse parameters</td><td>Issues with input parameters. Usually due to incorrect formats.</td><td>Check input format. Avoid passing floats in unsupported params like <code>amount_wei</code> or <code>timeout_millis</code></td></tr><tr><td>131004</td><td>invalid parameter</td><td>Issues with input parameters. Usually due to input conflicts or missing params.</td><td>Check the related API reference to see if any param is duplicated or missing.</td></tr><tr><td>201001</td><td>auth get api key is invalid</td><td>Auth error.</td><td>Use correct api key.</td></tr><tr><td>201005</td><td>rate reach limit</td><td>Rate limit is hit.</td><td>Reduce traffic.</td></tr></tbody></table>
