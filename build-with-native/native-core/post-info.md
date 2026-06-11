# POST /info

All public reads use one endpoint with a top-level `type` discriminator.

Unsupported or malformed query bodies return:

```json
{
  "error": {
    "code": "UnsupportedInfoType",
    "message": "unsupported public info type `...`"
  }
}
```

### assets

```sh
curl -sS -X POST "$API_URL/info" \
  -H 'content-type: application/json' \
  -H 'x-trace-id: client-trace-002' \
  -d '{"type":"assets"}'
```

Response:

See [Decimal Units](decimals-units.md) for raw/display conversion and validation rules. The gateway may serve this metadata response from an in-process cache for up to 10 seconds. The cache is refreshed by a background gateway task.

```json
{
  "query_height": 180000,
  "app_hash": "0x...",
  "assets": [
    {
      "asset_id": 0,
      "symbol": "USDC",
      "balance_decimals": 8,
      "withdraw_fee": "0",
      "withdraw_fee_atoms": "0",
      "issuer": "0x0000000000000000000000000000000000000000"
    },
    {
      "asset_id": 6,
      "symbol": "SOL",
      "balance_decimals": 8,
      "withdraw_fee": "0.01",
      "withdraw_fee_atoms": "1000000",
      "issuer": "0xabcdef0123456789abcdef0123456789abcdef01"
    }
  ]
}
```

`issuer` is the owner account bound to an asset by operator-managed metadata. Assets without an issuer binding surface as the zero address. The issuer is a per-asset binding only and confers no privileges beyond the recorded mapping. The `assets` response does not include `cloid`; client operation ids are only observable via `txStatusByCloid` while a transaction is in the recent query window. `withdraw_fee_atoms` is canonical asset metadata in asset-local atoms; `withdraw_fee` is formatted from atoms using `balance_decimals`. On a [withdraw](post-trade.md#withdraw) this fee is recorded (in the event and `/info withdraws`) but is **not** deducted from the balance.

### queryStatus

```json
{ "type": "queryStatus" }
```

Returns public query-view metadata and the retained recent-height window. This is the public way to discover the height bounds to pass to windowed reads such as [userFills](#userfills). Node role, writable/readable state, control-plane status, and other operational fields remain internal-only.

```json
{
  "query_height": 180000,
  "app_hash": "0x...",
  "oldest_available_height": 170001,
  "latest_available_height": 180000,
  "recent_query_window_blocks": 10000
}
```

When no query view is available yet, all fields are `null`.

### quoteAssets

```json
{ "type": "quoteAssets" }
```

Returns the current canonical quote-asset allowlist sorted by `asset_id`. `min_quantity` is the human-readable minimum order notional in the quote token; `min_quantity_atoms` is the raw integer atom quantity encoded as a decimal string for JavaScript safety. The gateway may serve this metadata response from an in-process cache for up to 10 seconds. The cache is refreshed by a background gateway task.

```json
{
  "query_height": 180000,
  "app_hash": "0x...",
  "quote_assets": [
    {
      "asset_id": 1,
      "symbol": "USDC",
      "balance_decimals": 8,
      "min_quantity": "10",
      "min_quantity_atoms": "1000000000"
    }
  ]
}
```

### accountingWithdrawTokens

```json
{ "type": "accountingWithdrawTokens" }
```

Returns accounting withdraw-token rows sorted by `(chain_id, asset_id)`. `min_withdraw_atoms` is the canonical raw atom amount encoded as a decimal string; `min_withdraw_amt` is formatted using the asset's `balance_decimals`.

```json
{
  "query_height": 180000,
  "app_hash": "0x...",
  "accounting_withdraw_tokens": [
    {
      "chain_id": 1,
      "asset_id": 1,
      "symbol": "USDC",
      "balance_decimals": 8,
      "min_withdraw_amt": "1.25",
      "min_withdraw_atoms": "125000000"
    }
  ]
}
```

### markets

```json
{ "type": "markets" }
```

Response field `markets` is sorted by `market_id`. `price_decimals` and `max_price_sig_figs` are market metadata, as is `base_quantity_decimals`. `quote_balance_decimals` is derived from the quote asset and included as a convenience field for raw notional conversion. See [Decimal Units](decimals-units.md). The gateway may serve this metadata response from an in-process cache for up to 10 seconds. The cache is refreshed by a background gateway task.

```json
{
  "query_height": 180000,
  "app_hash": "0x...",
  "markets": [
    {
      "market_id": 0,
      "base_asset_id": 3,
      "base_symbol": "ETH",
      "base_quantity_decimals": 4,
      "quote_asset_id": 1,
      "quote_symbol": "USDC",
      "quote_balance_decimals": 8,
      "price_decimals": 2,
      "max_price_sig_figs": 7
    }
  ]
}
```

### l2Book

```json
{
  "type": "l2Book",
  "market_id": "0",
  "depth": "20"
}
```

`depth` is optional, defaults to `20`, and is capped by the server.

```json
{
  "found": true,
  "query_height": 180000,
  "app_hash": "0x...",
  "market_id": 0,
  "requested_depth": 20,
  "depth": 20,
  "max_depth": 100,
  "bids": [
    { "price": "10", "quantity": "1.5", "order_count": 2 }
  ],
  "asks": []
}
```

### withdraws

Per-user retained withdraw records (3-day window), sorted by `(block_height, tx_index, withdraw_nonce)`. Requires `user`. Each record adds `withdraw_fee_atoms` (recorded, not deducted).

```json
{ "type": "withdraws", "user": "0x0000000000000000000000000000000000000001" }
```

```json
{
  "query_height": 180000,
  "app_hash": "0x...",
  "user": "0x0000000000000000000000000000000000000001",
  "withdraws": [
    {
      "tx_hash": "0x...",
      "block_height": 179999,
      "tx_index": 2,
      "block_timestamp_ms": 1717000000500,
      "asset_id": 1,
      "amount_atoms": "500000",
      "withdraw_fee_atoms": "1000",
      "dst_chain_id": 1,
      "dst_address": "0xbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbb",
      "withdraw_nonce": "7"
    }
  ]
}
```

### userBalances

```json
{
  "type": "userBalances",
  "user": "0x0000000000000000000000000000000000000001"
}
```

```json
{
  "query_height": 180000,
  "app_hash": "0x...",
  "owner": "0x0000000000000000000000000000000000000001",
  "balances": [
    {
      "asset_id": 0,
      "symbol": "USDC",
      "available": "1000",
      "locked": "0"
    }
  ]
}
```

### spotCreditAccount

```json
{
  "type": "spotCreditAccount",
  "user": "0x0000000000000000000000000000000000000001"
}
```

```json
{
  "query_height": 180000,
  "app_hash": "0x...",
  "owner": "0x0000000000000000000000000000000000000001",
  "authorized": true,
  "status": "active",
  "credit_usd_atoms": 100000000000,
  "available_usd_atoms": "99850000000",
  "last_known_available_usd_atoms": "99850000000",
  "oracle_status": { "status": "available" }
}
```

`status` is `"active"` or `"frozen"`. `credit_usd_atoms` and the available fields are in `usd_atoms` (`USD_SCALE = 10^8`). `available_usd_atoms` is `null` when any nonzero position asset lacks a mark at the latest query height; accounts with no exposure can report their credit without marks. `last_known_available_usd_atoms` always uses the most recently committed marks regardless of staleness and is `null` only when a position asset has never had a mark. Negative fractional USD-atom position values are rounded down conservatively, matching the execution credit gate.

### spotCreditPositions

```json
{
  "type": "spotCreditPositions",
  "user": "0x0000000000000000000000000000000000000001"
}
```

```json
{
  "query_height": 180000,
  "app_hash": "0x...",
  "owner": "0x0000000000000000000000000000000000000001",
  "spot_credit_positions": [
    {
      "asset_id": 2,
      "symbol": "ETH",
      "pending_exposure_qty": "-3",
      "actual_qty": "-2",
      "pending_exposure_display": "-0.0003",
      "actual_display": "-0.0002"
    }
  ]
}
```

Stage 6 single-leg semantics: a resting ask only debits the base asset; a resting bid only debits the quote asset. The other leg appears as `actual_qty` only when a fill produces a real settlement delta.

### oracleStatus

```json
{ "type": "oracleStatus" }
```

```json
{
  "query_height": 180000,
  "app_hash": "0x...",
  "oracle_status": { "status": "available" }
}
```

When the oracle is unavailable, `oracle_status` is `{ "status": "unavailable", "reason_code": "ProviderError" }` (or `MissingRoute`, `StalePrice`, `InvalidPrice`).

### markPrices

```json
{ "type": "markPrices", "asset_ids": [2, 4] }
```

`asset_ids` is optional; omitting it returns all known marks. The response is sorted by `asset_id`:

```json
{
  "query_height": 180000,
  "app_hash": "0x...",
  "oracle_status": { "status": "available" },
  "mark_prices": [
    {
      "asset_id": 2,
      "usd_atoms": 300000000000,
      "updated_height": 180000,
      "source_ts_ms": 1760000000000
    },
    {
      "asset_id": 4,
      "usd_atoms": 5000000000000,
      "updated_height": 180000,
      "source_ts_ms": 1760000000050
    }
  ]
}
```

`usd_atoms` is the positive mark price scaled by `USD_SCALE = 10^8`. `source_ts_ms` is the upstream provider's per-feed timestamp (Binance spot: worker wall clock at response-fully-read; OKX: per-ticker `ts`; Binance futures: per-symbol `time` from `/fapi/v2/ticker/price`; Llama: per-coin `timestamp` (seconds) × 1000; cross routes use `min(base, quote)` of the inputs).

### openOrders

```json
{
  "type": "openOrders",
  "user": "0x0000000000000000000000000000000000000001",
  "market_id": "0"
}
```

```json
{
  "query_height": 180000,
  "app_hash": "0x...",
  "owner": "0x0000000000000000000000000000000000000001",
  "market_id": 0,
  "limit": 500,
  "truncated": false,
  "open_orders": [
    {
      "status": "open",
      "owner": "0x0000000000000000000000000000000000000001",
      "market_id": 0,
      "oid": 773094113280001,
      "cloid": "0x11111111111111111111111111111111",
      "side": "bid",
      "price": "10",
      "original_qty": "1",
      "filled_qty": "0",
      "remaining_qty": "1",
      "created_tx_hash": "0x...",
      "created_height": 180000,
      "created_time_ms": 1760000000000
    }
  ]
}
```

`original_qty` is the submitted order quantity. For an order that partially fills and then rests on the book, `filled_qty` is nonzero and `remaining_qty` is the currently open quantity.

### userAgents

```json
{
  "type": "userAgents",
  "user": "0x0000000000000000000000000000000000000001"
}
```

Response field `agents` contains the active agent slots for the owner.

### userFills

```json
{
  "type": "userFills",
  "user": "0x0000000000000000000000000000000000000001",
  "from_height": "179900",
  "to_height": "180000",
  "limit": "100"
}
```

`limit` must be positive and is capped by the server. Height ranges wider than the configured recent window are rejected.

```json
{
  "query_height": 180000,
  "app_hash": "0x...",
  "owner": "0x0000000000000000000000000000000000000001",
  "fills": [
    {
      "height": 180000,
      "tx_index": 1,
      "tx_hash": "0x...",
      "market_id": 0,
      "role": "taker",
      "maker_oid": 773094113280001,
      "maker_cloid": "0x11111111111111111111111111111111",
      "maker_owner": "0x...",
      "taker_oid": 773094113280002,
      "taker_cloid": "0x22222222222222222222222222222222",
      "taker_owner": "0x...",
      "price": "10",
      "quantity": "1",
      "fee_asset_id": 1,
      "fee": "0.0001",
      "fee_mode": "credit"
    }
  ]
}
```

Each fill reports only the **querying owner's side** fee (chosen by `role`). `fee_asset_id` is the asset the fee was charged in (the asset that side received: base for a bid, quote for an ask), `fee` is the decimal amount formatted with that asset's `balance_decimals`, and `fee_mode` is `"balance"` or `"credit"`. (Internally — in canonical events and the query model — the fee `amount` is a raw atom; only this public JSON `fee` is the formatted decimal string.) Trading fees activate at a network-specific height defined by the code protocol-rules schedule. There are two distinct null cases:

* **Pre-activation / no-fee fill** (below the network's fee activation height): `"fee_asset_id": null, "fee": "0", "fee_mode": null`.
* **Fee-active fill whose fee floored to zero**: non-null `fee_asset_id` and `fee_mode` with `"fee": "0"` — so clients can tell the fee regime is live.

The response shape is unchanged, but the `fee` amount may reflect code-defined per-market and per-account fee overrides (the effective rate is the minimum of the global rate and any matching market/account override). A fully waived market appears as a fee-active fill with `"fee": "0"` (the second null case above).

### orderStatus

Query by server order id:

```json
{
  "type": "orderStatus",
  "oid": "773094113280001"
}
```

Query by client order id:

```json
{
  "type": "orderStatus",
  "user": "0x0000000000000000000000000000000000000001",
  "market_id": "0",
  "cloid": "0x11111111111111111111111111111111"
}
```

`orderStatus` first checks the live pending overlay (cloid lookup only), then the current open-order view. If neither covers the order, it falls back to the latest retained status record for that `oid` or `(user, market_id, cloid)`. Status records are action-result records, not a complete lifecycle stream: a successful incoming order writes one record even when it rests on the book, but later passive maker fills are exposed through `userFills` and open-order state changes rather than a new status record for the maker order. Explicit cancel and modify actions do write status records for the affected resting order. Outcome-stage failed order attempts also write retained status records when execution has enough order context: `IocCancel`, `FokCancel`, `BadAloPx`, `MarketOrderNoLiquidity`, and `InsufficientSpotCredit` can be queried by `oid` or `(user, market_id, cloid)` inside the recent query window.

The cloid lookup additionally surfaces:

* **Pending response (live overlay)**: when `/trade` returned `accepted` for a cloid-bearing tx but the queryView has not yet published the result, `orderStatus` by `cloid` returns a synthetic pending response (see below). The pending entry retires automatically when queryView publishes a result for the same accepted `tx_hash`; a queryView result whose `tx_hash` matches replaces the pending response with the real result on the next query.
* **Null-OID early-failure rows**: when an accepted cloid-bearing tx fails before execution can assign an `OrderId` (`BadNonce`, `ExpiredTx`, agent/owner authorization failure, missing modify target, batch-item-level pre-OID failures, etc.), the published row carries `"oid": null` and a lower-case canonical status string. The row remains visible only while the live process is up — it is not regenerated by WAL replay on restart.

### batchOrderStatus

Batch form of [orderStatus](#orderstatus), capped at 20 lookups per request. Each item supports the same lookup forms as `orderStatus`: `{ "oid": "..." }` or `{ "user": "...", "market_id": "...", "cloid": "..." }`. A parseable `oid` takes precedence for that item. Results are returned in the same order as the input array.

```json
{
  "type": "batchOrderStatus",
  "orders": [
    { "oid": "773094113280001" },
    {
      "user": "0x0000000000000000000000000000000000000001",
      "market_id": "0",
      "cloid": "0x11111111111111111111111111111111"
    }
  ]
}
```

```json
{
  "results": [
    {
      "found": true,
      "query_height": 180000,
      "app_hash": "0x...",
      "status": "open",
      "order": { "status": "open", "oid": 773094113280001 }
    },
    {
      "found": false,
      "query_height": 180000,
      "app_hash": "0x...",
      "owner": "0x0000000000000000000000000000000000000001",
      "market_id": 0,
      "cloid": "0x11111111111111111111111111111111"
    }
  ]
}
```

Malformed items do not prevent valid items from being queried; the malformed slot contains an item-local error:

```json
{
  "results": [
    {
      "error": {
        "code": "InvalidOrderStatusQuery",
        "message": "orderStatus requires either oid or user + market_id + cloid"
      }
    }
  ]
}
```

If `orders` is not an array the gateway returns HTTP 400 `InvalidOrderStatusBatch`. If more than 20 items are supplied, the gateway returns HTTP 400:

```json
{
  "error": {
    "code": "TooManyOrderStatusQueries",
    "message": "batchOrderStatus supports at most 20 orders"
  }
}
```

Transaction status by `cloid` uses the transaction authority as the `user` namespace and does not require a market id. For `withdraw`, `settle`, and `repay`, `user` is the recovered signer authority.

```json
{
  "type": "txStatusByCloid",
  "user": "0x00ee41b8f0dd58806f14b30fb11994673769b25c",
  "cloid": "0x11111111111111111111111111111111"
}
```

```json
{
  "found": true,
  "query_height": 180000,
  "app_hash": "0x...",
  "owner": "0x00ee41b8f0dd58806f14b30fb11994673769b25c",
  "cloid": "0x11111111111111111111111111111111",
  "tx_hash": "0x...",
  "tx_index": 1,
  "status": "success",
  "action_type": "settle"
}
```

Failed retained tx responses use the lower-case committed execution error code as `status`, matching the `orderStatus` style where terminal failures are reported through the status string:

```json
{
  "found": true,
  "query_height": 180000,
  "app_hash": "0x...",
  "owner": "0x00ee41b8f0dd58806f14b30fb11994673769b25c",
  "cloid": "0x11111111111111111111111111111111",
  "tx_hash": "0x...",
  "tx_index": 1,
  "status": "invalidsettle",
  "action_type": "settle"
}
```

If a failed retained tx has no single committed error code in its payload, `status` remains `"failed"`.

The public `settle`/`repay` actions also surface here, with `action_type` `"settle"` / `"repay"`. Their `user` namespace is the **recovered signer** (settle → margin owner; repay → cash owner), so a counterparty named in the action body cannot find the tx. This window is the only `cloid`-keyed lookup for settle/repay: there is no idempotency, only recent-window visibility bounded by `max_recent_txs`. The same `cloid` resubmitted under a new envelope `nonce` is a separate tx; a lookup returns the latest retained one, and once a tx ages out of the window the response is `found: false`. A failed settle/repay is retained the same way and reports `status` `"invalidsettle"` / `"invalidrepay"` (or another lower-case committed error code).

Open order response:

```json
{
  "found": true,
  "query_height": 180000,
  "app_hash": "0x...",
  "status": "open",
  "order": {
    "status": "open",
    "market_id": 0,
    "oid": 773094113280001,
    "cloid": "0x11111111111111111111111111111111",
    "remaining_qty": "1"
  }
}
```

Pending response (live overlay, accepted but not yet published):

```json
{
  "found": true,
  "query_height": 179999,
  "app_hash": "0x...",
  "status": "pending",
  "order": {
    "status": "pending",
    "oid": null,
    "cloid": "0x11111111111111111111111111111111",
    "owner": "0x0000000000000000000000000000000000000001",
    "market_id": 0,
    "tx_hash": "0xACCEPTEDTX...",
    "created_tx_hash": "0xACCEPTEDTX...",
    "updated_tx_hash": "0xACCEPTEDTX..."
  }
}
```

Null-OID early-failure response (live-only):

```json
{
  "found": true,
  "query_height": 180001,
  "app_hash": "0x...",
  "order": {
    "status": "badnonce",
    "market_id": 0,
    "oid": null,
    "cloid": "0x22222222222222222222222222222222",
    "updated_tx_hash": "0x...",
    "updated_height": 180001
  }
}
```

Terminal order response:

```json
{
  "found": true,
  "query_height": 180000,
  "app_hash": "0x...",
  "order": {
    "status": "filled",
    "market_id": 0,
    "oid": 773094113280001,
    "cloid": "0x11111111111111111111111111111111",
    "filled_qty": "1",
    "remaining_qty": "0",
    "is_resting": false,
    "updated_tx_hash": "0x...",
    "updated_height": 180000
  }
}
```

Not found:

```json
{
  "found": false,
  "query_height": 180000,
  "app_hash": "0x...",
  "owner": "0x0000000000000000000000000000000000000001",
  "market_id": 0,
  "cloid": "0x11111111111111111111111111111111"
}
```
