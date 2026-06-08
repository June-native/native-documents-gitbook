# POST /trade

Submits one signed action.

```sh
curl -sS -X POST "$API_URL/trade" \
  -H 'content-type: application/json' \
  -d '{
    "action": {
      "type": "order",
      "market_id": "0",
      "side": "bid",
      "order_type": "limit",
      "tif": "gtc",
      "price": "1000.00",
      "quantity": "0.0025",
      "cloid": "0x11111111111111111111111111111111"
    },
    "nonce": "1760000000000",
    "expires_after_ms": "1760000005000",
    "signature": "0x..."
  }'
```

Request envelope fields:

| Field              | Required | Description                                                                                                                                                                                                                                          |
| ------------------ | -------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `action`           | yes      | Public action object. Supported top-level action types are `order`, `cancel`, `cancelAll`, `modify`, and `batch`.                                                                                                                                    |
| `nonce`            | yes      | Decimal string `u64` Unix millisecond timestamp nonce. Use current `Date.now()`/Unix ms; if sending multiple requests in the same millisecond for the same signer, increment locally so each signed nonce is unique and monotonically nondecreasing. |
| `agent_epoch`      | no       | Decimal string `u64`; required only for agent-signed requests. Omit for owner-signed requests.                                                                                                                                                       |
| `expires_after_ms` | no       | Decimal string `u64` Unix milliseconds. Expired transactions are rejected during execution.                                                                                                                                                          |
| `signature`        | yes      | `0x`-prefixed 65-byte recoverable secp256k1 signature over the canonical unsigned payload.                                                                                                                                                           |

Nonce validation is signer-scoped. Execution accepts nonces within the committed block timestamp window (`block_timestamp_ms - 2 days` through `block_timestamp_ms + 1 day`), rejects duplicates, and retains the latest 100 consumed nonces per signer. When the retained window is full, a new nonce must be greater than the current minimum retained nonce.

### order

Places one order.

| Field        | Required | Values                                                                                                                                          |
| ------------ | -------- | ----------------------------------------------------------------------------------------------------------------------------------------------- |
| `type`       | yes      | `"order"`                                                                                                                                       |
| `market_id`  | yes      | Decimal string market id.                                                                                                                       |
| `side`       | yes      | `"bid"` or `"ask"`; `"buy"` and `"sell"` are also accepted aliases.                                                                             |
| `order_type` | yes      | `"limit"` or `"market"`.                                                                                                                        |
| `tif`        | yes      | `"gtc"`, `"ioc"`, `"fok"`, or `"alo"`.                                                                                                          |
| `price`      | yes      | Human decimal string display price. For limit orders this is the limit price. For market orders this is the protection price used by execution. |
| `quantity`   | yes      | Human decimal string base-asset quantity.                                                                                                       |
| `cloid`      | no       | `0x`-prefixed 16-byte client order id.                                                                                                          |

Limit order example:

```json
{
  "type": "order",
  "market_id": "2",
  "side": "bid",
  "order_type": "limit",
  "tif": "alo",
  "price": "3500.00",
  "quantity": "1.0000",
  "cloid": "0x11111111111111111111111111111111"
}
```

Protected market order example:

```json
{
  "type": "order",
  "market_id": "2",
  "side": "ask",
  "order_type": "market",
  "tif": "ioc",
  "price": "3490.00",
  "quantity": "1.0000"
}
```

### cancel

Cancels one order by exchange order id or client order id. Provide `oid` or `cloid`; if both are present, `oid` is used.

| Field       | Required    | Values                                                                   |
| ----------- | ----------- | ------------------------------------------------------------------------ |
| `type`      | yes         | `"cancel"`                                                               |
| `market_id` | yes         | Decimal string market id.                                                |
| `oid`       | conditional | Decimal string exchange order id. Required unless `cloid` is present.    |
| `cloid`     | conditional | `0x`-prefixed 16-byte client order id. Required unless `oid` is present. |

```json
{
  "type": "cancel",
  "market_id": "2",
  "oid": "773094113280001"
}
```

```json
{
  "type": "cancel",
  "market_id": "2",
  "cloid": "0x11111111111111111111111111111111"
}
```

### cancelAll

Cancels every open resting order for the effective owner (recovered signer or agent-resolved principal) in one market. The market must exist; an unknown market is rejected by execution as `MarketNotFound`. A market that exists but has no open orders for this owner is a successful no-op.

Effects to confirm via reads:

* The owner's `openOrders` for this `market_id` becomes empty.
* Each previously-open `oid` reaches `orderStatus = "cancelled"` with `remaining_qty = 0`. Orders remain queryable by their existing `oid` and `cloid` (if any).
* Open orders in other markets and orders owned by other signers are unaffected.

| Field       | Required | Values                    |
| ----------- | -------- | ------------------------- |
| `type`      | yes      | `"cancelAll"`             |
| `market_id` | yes      | Decimal string market id. |

`cancelAll` carries no `oid` and no `cloid`. The `missing_oid_or_cloid` parse error does not apply to it. Agent signatures are accepted (same allowlist as `cancel`). Submit precheck classifies it (and any pure-`cancelAll` or `cancel`/`cancelAll`-only batch) as a pure cancel: oracle freshness, frozen SpotCreditAccount, mark coverage, quote-min-notional, and duplicate cloid checks are skipped at admission.

```json
{
  "type": "cancelAll",
  "market_id": "2"
}
```

### modify

Replaces one open order using action-atomic cancel-plus-place semantics. Provide `oid` or `cloid`; if both are present, `oid` is used. `replacement` has the same shape as an order without `market_id`.

| Field                    | Required    | Values                                                                                                                                          |
| ------------------------ | ----------- | ----------------------------------------------------------------------------------------------------------------------------------------------- |
| `type`                   | yes         | `"modify"`                                                                                                                                      |
| `market_id`              | yes         | Decimal string market id.                                                                                                                       |
| `oid`                    | conditional | Decimal string exchange order id. Required unless `cloid` is present.                                                                           |
| `cloid`                  | conditional | `0x`-prefixed 16-byte client order id. Required unless `oid` is present.                                                                        |
| `replacement.side`       | yes         | `"bid"` or `"ask"`; `"buy"` and `"sell"` are also accepted aliases.                                                                             |
| `replacement.order_type` | yes         | `"limit"` or `"market"`.                                                                                                                        |
| `replacement.tif`        | yes         | `"gtc"`, `"ioc"`, `"fok"`, or `"alo"`.                                                                                                          |
| `replacement.price`      | yes         | Human decimal string display price. For limit orders this is the limit price. For market orders this is the protection price used by execution. |
| `replacement.quantity`   | yes         | Human decimal string base-asset quantity.                                                                                                       |
| `replacement.cloid`      | no          | `0x`-prefixed 16-byte replacement client order id, or `null` to omit one.                                                                       |

```json
{
  "type": "modify",
  "market_id": "2",
  "cloid": "0x11111111111111111111111111111111",
  "replacement": {
    "side": "bid",
    "order_type": "limit",
    "tif": "alo",
    "price": "3490.00",
    "quantity": "1.0000",
    "cloid": "0x33333333333333333333333333333333"
  }
}
```

### batch

`batch` is the only multi-item write action. Its `items` array may mix these item types in payload order under one envelope nonce:

* `order`
* `cancel`
* `cancelAll`
* `modify`

`cancel` and `modify` items use the same target rules as top-level actions: provide `oid` or `cloid`; if both are present, `oid` is used. A `cancelAll` item carries only `market_id` and behaves the same as a top-level `cancelAll`. A replacement has the same order shape as an order without `market_id`.

```json
{
  "type": "batch",
  "items": [
    {
      "type": "order",
      "market_id": "2",
      "side": "bid",
      "order_type": "limit",
      "tif": "alo",
      "price": "3500.00",
      "quantity": "1.0000",
      "cloid": "0x22222222222222222222222222222222"
    },
    {
      "type": "modify",
      "market_id": "2",
      "cloid": "0x22222222222222222222222222222222",
      "replacement": {
        "side": "bid",
        "order_type": "limit",
        "tif": "alo",
        "price": "3490.00",
        "quantity": "1.0000",
        "cloid": "0x33333333333333333333333333333333"
      }
    }
  ]
}
```

Batch constraints:

* `items` must contain `1..=10` items.
* Items execute in array order.
* The batch has one envelope nonce. Individual items may succeed or fail inside the batch execution result.
