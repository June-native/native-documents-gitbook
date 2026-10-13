# Blacklisting

To protect LP, malicious arbitrage that puts Native liquidity at risk will be profiled, and the user/swapper address will be added to the blacklist.

To avoid producing additional errors during the /firm-quote stage, any quotes submitted with `from_address` or `beneficiary_address` being blacklisted will have a deduction applied to `amountOut` due to increased risk (default 50bps; this is configurable per deployment and may differ for your API key). `from_address` is checked first, then `beneficiary_address`. Addresses on the gray list are quoted with separate, time-based slippage settings rather than this deduction.

We recommend that all upstreams query and cache a copy of the latest blacklisted addresses and apply pre-processing and filtering.

{% hint style="info" %}
`/v1/blacklist` returns only the **latest 140,000 records**. To validate addresses outside that window, use `/v1/is-black-address` or maintain a local cache.
{% endhint %}

#### /blacklist Endpoint

```
## Request
curl "https://v2.api.native.org/swap-api-v2/v1/blacklist?page_size=35000&page_index=1" \
     -H "apiKey: $NATIVE_KEY"
```

Param:

* page\_size: entries per page. Values below 1 or above 35000 are clamped to 35000
* page\_index: page index (values below 1 are treated as 1)

The list is cached and rebuilt every few minutes, so recently added addresses may take a short while to appear. Whitelisted and gray-listed addresses are excluded.

Normal return:

```
{
  "black_list": [ // latest first
    {
      "id": "466221",
      "address": "0x000000000000000000000000000000000000dead", // address
      "chainId": 1,
      "createTime": 1775026096 // unix timestamp
    }
  ],
  "total_count": 140000 // capped at 140000 (latest records only)
}
```

Upstream must contact the Native team through the existing communication channel to request permission for this endpoint.

No permission return:

```
{
  "code": 201009,
  "message": "no auth to call"
}
```

#### /is-black-address Endpoint

Check whether a single address is on the blacklist (gray-listed addresses return `false`). Like `/blacklist`, this endpoint must be enabled for your API key, otherwise it returns `201009`. Use this for addresses that may fall outside the latest 140,000 records returned by `/blacklist`.

```
## Request
curl "https://v2.api.native.org/swap-api-v2/v1/is-black-address?address=0x000000000000000000000000000000000000dead" \
     -H "apiKey: $NATIVE_KEY"
```

Param:

* address: the address to check
* chain\_id: accepted but currently ignored; the lookup is by address only

Normal return:

```
{
  "is_black_address": true
}
```

#### Blacklist status on /firm-quote

Pass optional `show_blacklist_status=true` on `/v1/firm-quote` to include `is_black_address` in the response. The field is not populated on wrap/unwrap or AMM-fallback responses, where it will read `false`. Omit the parameter or pass `false` to keep the default response (field omitted).

```
## Request
curl "https://v2.api.native.org/swap-api-v2/v1/firm-quote?src_chain=ethereum&dst_chain=ethereum&token_in=TOKEN_IN&token_out=TOKEN_OUT&amount=1000&from_address=ADDRESS&version=6&expiry_time=60&show_blacklist_status=true" \
     -H "apiKey: $NATIVE_KEY"
```

Return when `show_blacklist_status=true`:

```
{
  "success": true,
  "orders": [...],
  "is_black_address": false
}
```

Return when omitted or `false` (field not included):

```
{
  "success": true,
  "orders": [...]
}
```
