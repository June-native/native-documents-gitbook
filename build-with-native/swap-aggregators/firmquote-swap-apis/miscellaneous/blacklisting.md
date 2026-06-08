# Blacklisting

To protect LP, malicious arbitrage that puts Native liquidity at risk will be profiled, and the user/swapper address will be added to the blacklist.

To avoid producing additional errors during the /firm-quote stage, any quotes submitted with `from_address` or `beneficiary_address` being blacklisted will be subject to a 50bps added spread due to increased risk.

We recommend that all upstreams query and cache a copy of the latest blacklisted addresses and apply pre-processing and filtering.

#### /blacklist Endpoint

```
## Request
curl "https://v2.api.native.org/swap-api-v2/v1/blacklist?page_size=200&page_index=1" \
     -H "apiKey: $NATIVE_KEY"
```

Param:

* page\_size: entries per page, max 500
* page\_index: page index

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
  "total_count": 44233 // total entires count
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
