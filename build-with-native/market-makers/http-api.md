---
description: POST books and GET the books Native currently holds for your maker account.
layout:
  width: wide
---

# HTTP API

## Push orderbooks — `POST /v1/orderbook`

Headers:

| Header | Value |
| --- | --- |
| `Apikey` | Your API key |
| `Content-Type` | `application/json` |

Request:

```json
{
  "nonce": 1731318400123,
  "orderbooks": [
    {
      "chain_id": 56,
      "base":  { "address": "0x698250914c54eb2f1bb7e0eb1e6a24c0f9b1c1a1", "symbol": "MEME" },
      "quote": { "address": "0x55d398326f99059ff775485246999027b3197955", "symbol": "USDT" },
      "bids": [
        { "price": "0.98", "qty": "12000" },
        { "price": "0.97", "qty": "25000" }
      ],
      "asks": [
        { "price": "1.02", "qty": "8000" },
        { "price": "1.03", "qty": "15000" }
      ],
      "expire_at_ms": 1731318406000,
      "book_id": "b-000123"
    }
  ]
}
```

| Field | Type | Required | Description |
| --- | --- | --- | --- |
| `nonce` | integer | yes | Per-maker monotonic counter (see [Nonce](README.md#nonce)) |
| `orderbooks` | array | yes | 1–20 orderbooks per request |
| `orderbooks[].chain_id` | integer | yes | EVM chain id of the pair |
| `orderbooks[].base.address` | string | yes | Base token contract, EVM hex; **must be lowercase** |
| `orderbooks[].quote.address` | string | yes | Quote token contract, EVM hex; **must be lowercase** |
| `orderbooks[].base.symbol`, `orderbooks[].quote.symbol` | string | yes | The registered symbols, **case-sensitive** — use the exact strings from the mapping (onboarding sheet / `GET /v1/orderbook`) |
| `orderbooks[].bids`, `orderbooks[].asks` | array | yes | Levels, best price first; may be empty to withdraw that side |
| `orderbooks[].expire_at_ms` | integer | yes | Absolute expiry, ms Unix; at least receive time + 5 s |
| `orderbooks[].book_id` | string | yes | Your id for this book (non-empty, ≤ 64 chars, **unique within a request**); echoed in the ack and in your statements |

A push response carries no book data and no trim data: on success it is a bare confirmation, and `results` appears only on failure — listing only the problem books, in request order, correlated by `book_id` (required, unique within a request).

```json
{
  "applied": true,
  "trace_id": "01JD8Z9W3C",
  "server_time_ms": 1731318400450
}
```

| Field | Description |
| --- | --- |
| `applied` | `true` — every book passed and was applied, and the nonce was consumed. `false` — nothing was applied and the nonce was **not** consumed |
| `server_time_ms` | Native's clock when the ack was built |
| `trace_id` | Trace id of this push |
| `results` | Present only when `applied` is `false`; lists only the problem books, in request order, correlated by `book_id` |
| `results[].status` | `rejected` — this book failed validation (see `reason`); `skipped` — this book was fine, but the request as a whole was not applied (`atomic_abort`), or the nonce gate failed (`stale_nonce`). `accepted` never appears today — an applied request returns no results; it may appear if partial acceptance is enabled later |
| `results[].reason`, `results[].detail` | On rejected/skipped results. `detail` names the offending level with its values (e.g. `level[3] qty 50 < min 100`). A rejected result repeats the pair identity (`chain_id`, `base`, `quote`) so the failure is self-contained |

When `applied` is `false`, nothing changed at all — every previous book stays quotable until it expires, and the same nonce may be reused. A rejected result repeats the pair identity and states the problem precisely; a skipped result stays minimal. Example — one bad book aborts the whole request:

```json
{
  "applied": false,
  "trace_id": "01JD8ZA112F",
  "server_time_ms": 1731318400900,
  "results": [
    {
      "book_id": "b-000123",
      "status": "skipped",
      "reason": "atomic_abort"
    },
    {
      "chain_id": 137,
      "base": "0xbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbb",
      "quote": "0xcccccccccccccccccccccccccccccccccccccccccc",
      "book_id": "b-000124",
      "status": "rejected",
      "reason": "crossed_book",
      "detail": "best bid 1.05 >= best ask 1.02"
    }
  ]
}
```

Quick test with curl:

```bash
curl -X POST "https://api.native.org/v1/orderbook" \
  -H "Apikey: YOUR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
        "nonce": 1731318400123,
        "orderbooks": [{
          "chain_id": 56,
          "base":  { "address": "0x698250914c54eb2f1bb7e0eb1e6a24c0f9b1c1a1", "symbol": "MEME" },
          "quote": { "address": "0x55d398326f99059ff775485246999027b3197955", "symbol": "USDT" },
          "bids": [{ "price": "0.98", "qty": "12000" }],
          "asks": [{ "price": "1.02", "qty": "8000" }],
          "expire_at_ms": 1731318406000,
          "book_id": "b-000123"
        }]
      }'
```

## Query your current orderbooks — `GET /v1/orderbook`

Returns every **currently effective** book of the maker of this API key — the levels exactly as last accepted, plus what is still quotable after trimming. Only live books are returned: a pair whose book is expired, withdrawn or never pushed simply does not appear. This is the only query endpoint; pair constraints, fee rates and the authoritative mapping come from your onboarding sheet. Rate limit for this endpoint: **5 requests/s per maker**, separate from the push rate.

```json
{
  "server_time_ms": 1731318401000,
  "trace_id": "01JD8ZK444T",
  "orderbooks": [
    {
      "chain_id": 56,
      "base":  { "address": "0x698250914c54eb2f1bb7e0eb1e6a24c0f9b1c1a1", "symbol": "MEME" },
      "quote": { "address": "0x55d398326f99059ff775485246999027b3197955", "symbol": "USDT" },
      "bids": [ { "price": "0.98", "qty": "12000" } ],
      "asks": [ { "price": "1.02", "qty": "8000" } ],
      "nonce": 1731318400123,
      "book_id": "b-000123",
      "recv_time_ms": 1731318400450,
      "expire_at_ms": 1731318406000,
      "quoted_bid_base_qty": "24800",
      "quoted_ask_base_qty": "20900"
    }
  ]
}
```

`bids`/`asks` are the levels the server currently holds (as accepted, before trimming); `quoted_*` is the depth still quotable now — **this endpoint is where trim feedback lives; the push response never carries depth numbers**. For every pair shown, the exact `symbol` strings (case-sensitive) and the direction are what your pushes must match; for pairs not shown, your onboarding sheet is the mapping.

## HTTP status codes

| Code | Meaning | What to do |
| --- | --- | --- |
| 200 | Request processed — check `applied`; `results` appears only on failure | — |
| 400 | Malformed JSON, or a required field missing/invalid at request level | Fix the named field |
| 401 | Missing, unknown or disabled API key | Check the `Apikey` header |
| 403 | Source IP not allowlisted | Contact ops |
| 413 | Body over 1 MiB, or over 20 orderbooks | Split the request |
| 429 | Over 10 requests/s | Back off; see `retry_after_ms` in the body |

Reject reasons and size/rate limits are on [Reject reasons & limits](errors-and-limits.md).
