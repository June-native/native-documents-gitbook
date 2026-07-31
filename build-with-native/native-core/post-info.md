# POST /info

All public reads use one endpoint with a top-level `type` discriminator.

A malformed body, or one whose `type` is missing or not a string, returns HTTP `400` with `error.code: "InvalidInfoRequest"`. A well-formed body whose `type` is not a known query returns HTTP `400` with `error.code: "UnsupportedInfoType"`:

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
  -d '{"type":"assets"}'
```

Response:

See [Decimals & Units](decimals-units.md) for raw/display conversion and validation rules.
```json
{
  "query_height": 180000,
  "app_hash": "0x...",
  "assets": [
    {
      "asset_id": 1,
      "symbol": "USDC",
      "balance_decimals": 8,
      "withdraw_fee": "1",
      "withdraw_fee_atoms": "100000000",
      "issuer": "0x0000000000000000000000000000000000000000",
      "credit_ltv": 100,
      "credit_ltv_setting": null
    },
    {
      "asset_id": 3,
      "symbol": "ETH",
      "balance_decimals": 8,
      "withdraw_fee": "0.001",
      "withdraw_fee_atoms": "100000",
      "issuer": "0x0000000000000000000000000000000000000000",
      "credit_ltv": 100,
      "credit_ltv_setting": null
    }
  ]
}
```

`credit_ltv` is the asset's loan-to-value percentage for credit accounts — the haircut applied when it counts toward a credit line. `credit_ltv_setting` echoes the per-asset override when one is configured, `null` otherwise.

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

Returns the current canonical quote-asset allowlist sorted by `asset_id`. `min_quantity` is the human-readable minimum order notional in the quote token; `min_quantity_atoms` is the raw integer atom quantity encoded as a decimal string for JavaScript safety.
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

### accountingDepositContracts

```json
{ "type": "accountingDepositContracts" }
```

Returns the configured deposit source contracts sorted by `src_chain_id`. No parameters.

```json
{
  "query_height": 180000,
  "app_hash": "0x...",
  "accounting_deposit_contracts": [
    {
      "src_chain_id": 421614,
      "src_contract": "0xaabbccddeeff00112233445566778899aabbccdd"
    }
  ]
}
```

### multisigPolicy

Returns the dynamic non-admin multisig policy for one `scope`. Only the `ACCOUNTING` scope (`2`) is queryable; any other value returns HTTP 400 `InvalidMultisigScope`.

```json
{ "type": "multisigPolicy", "scope": "2" }
```

When a policy is set:

```json
{
  "query_height": 180000,
  "app_hash": "0x...",
  "found": true,
  "policy": {
    "scope": 2,
    "scope_name": "ACCOUNTING",
    "threshold": 2,
    "signers": [
      "0x1111111111111111111111111111111111111111",
      "0x2222222222222222222222222222222222222222"
    ]
  }
}
```

When none is set, `found` is `false` and the body echoes `scope` / `scope_name` instead of `policy`.

### markets

```json
{ "type": "markets" }
```

Response field `markets` is sorted by `market_id`. `price_decimals` and `max_price_sig_figs` are market metadata, as is `base_quantity_decimals`. `quote_balance_decimals` is derived from the quote asset and included as a convenience field for raw notional conversion. See [Decimals & Units](decimals-units.md).
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
      "max_price_sig_figs": 7,
      "credit_trading": true,
      "credit_trading_setting": null,
      "maker_fee_milli_bps": 2000,
      "taker_fee_milli_bps": 10000,
      "fee_setting": null
    }
  ]
}
```

`maker_fee_milli_bps` / `taker_fee_milli_bps` are the market's **effective** fee rates in milli-basis-points — never null, and the only programmatic source of what you will be charged. Divide by 1000 for basis points: `2000` is 2 bps maker, `10000` is 10 bps taker. `fee_setting` echoes the per-market override when one is configured, `null` otherwise.

`credit_trading` is the per-market gate for credit accounts. A credit account may trade a market when its own status is `active` **and** either `credit_trading` is true or the market appears in that account's `credit_trading_whitelisted_market_ids` (see [`spotCreditAccount`](#spotcreditaccount)). `credit_trading_setting` echoes the per-market override, `null` otherwise.

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
  "query_time_ms": 1785338171446,
  "last_update_height": 129475493,
  "last_update_ts_ms": 1785338171296,
  "bids": [
    { "price": "10", "quantity": "1.5", "order_count": 2 }
  ],
  "asks": []
}
```

`query_time_ms` is the block time of the view you read; `last_update_height` / `last_update_ts_ms` are when **this market's** book last changed. Together they are the staleness signal: a wide gap between `last_update_ts_ms` and `query_time_ms` means the book is quiet, not that your read is stale.

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
      "withdraw_nonce": "7",
      "vault_address": "0xcccccccccccccccccccccccccccccccccccccccc"
    }
  ]
}
```

`vault_address` is the [vault contract](../deposit-withdraw/vault-contract.md) the withdrawal pays out from. It is `null` on records written before vault payouts were activated.

### deposits

Per-user retained deposit records. Requires `user`; the response key is `user`.

```json
{ "type": "deposits", "user": "0x0000000000000000000000000000000000000001" }
```

```json
{
  "query_height": 180000,
  "app_hash": "0x...",
  "user": "0x0000000000000000000000000000000000000001",
  "deposits": [
    {
      "tx_hash": "0x...",
      "block_height": 179998,
      "tx_index": 0,
      "block_timestamp_ms": 1717000000200,
      "asset_id": 1,
      "amount_atoms": "1000000",
      "src_chain_id": 421614,
      "src_contract": "0xaabbccddeeff00112233445566778899aabbccdd",
      "deposit_nonce": "42"
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

### accountStatus

Whether an account exists and its freeze state.

```json
{
  "type": "accountStatus",
  "user": "0x0000000000000000000000000000000000000001"
}
```

```json
{
  "query_height": 180000,
  "app_hash": "0x...",
  "owner": "0x0000000000000000000000000000000000000001",
  "found": true,
  "account_index": 5,
  "status": "active"
}
```

`found` is `true` once the account exists (it is created on its first deposit). `status` is `"active"` or `"frozen"`, and is `null` when `found` is `false`. `account_index` is the protocol's internal account index, `null` before the account exists.

### spotCreditAccount

{% hint style="info" %}
`spotCreditAccount` and `spotCreditPositions` read a **credit account** — a protocol-granted account type distinct from the default spot (balance) account that [`userBalances`](post-info.md#userbalances) reads. If you have not been granted a credit line, these report no position. See [Account Types](account-types.md).
{% endhint %}

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

Pass `market_id: -1` (as the number `-1` or the string `"-1"`) to get the owner's open orders across **every** market in one call. The response echoes `"market_id": -1` and applies the same 500-item cap and `truncated` flag over the union.

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
      "owner_index": 5,
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

`original_qty` is the submitted order quantity. For an order that partially fills and then rests on the book, `filled_qty` is nonzero and `remaining_qty` is the currently open quantity. `owner_index` is the protocol's internal account index for the owner; it also appears on the order objects returned by `orderStatus`.

### userAgents

```json
{
  "type": "userAgents",
  "user": "0x0000000000000000000000000000000000000001"
}
```

```json
{
  "query_height": 180000,
  "app_hash": "0x...",
  "owner": "0x0000000000000000000000000000000000000001",
  "agents": [
    { "slot_id": 0, "agent": "0x00000000000000000000000000000000000000aa", "epoch": 3 }
  ]
}
```

`agents` holds the active agent (API-wallet) slots for the owner. To sign `/trade` writes, read the `epoch` of the slot whose `agent` matches your API-wallet address and pass it as the envelope `agent_epoch`. `agent` is `null` for an empty slot.

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

`limit` must be positive and is capped at `max_limit` (500). Two different failure shapes, and the difference matters to your parser:

* A `limit` that is **missing, non-numeric, or beyond `u32`** is rejected with **HTTP 400** and `InvalidFillsQuery` — there is no `fills` key at all.
* A `limit` of `0`, a `from_height` above `to_height`, or a range wider than the recent window returns **HTTP 200** with an in-band `error` object (`InvalidFillsQuery` or `HistoryWindowExceeded`) and an empty `fills` array. On success the response echoes the effective `limit`, your `requested_limit`, and `max_limit`.

```json
{
  "query_height": 180000,
  "app_hash": "0x...",
  "owner": "0x0000000000000000000000000000000000000001",
  "from_height": 179900,
  "to_height": 180000,
  "limit": 100,
  "requested_limit": 100,
  "max_limit": 500,
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

`orderStatus` checks the current open-order view, then falls back to the latest retained status record for that `oid` or `(user, market_id, cloid)`. Status records are action-result records, not a complete lifecycle stream: a successful incoming order writes one record even when it rests on the book, but later passive maker fills are exposed through `userFills` and open-order state changes rather than a new status record for the maker order. Explicit cancel and modify actions do write status records for the affected resting order. Outcome-stage failed order attempts also write retained status records when execution has enough order context: `IocCancel`, `FokCancel`, `BadAloPx`, `MarketOrderNoLiquidity`, and `InsufficientSpotCredit` can be queried by `oid` or `(user, market_id, cloid)` inside the recent query window.

A partially filled order that is still on the book reports `status: "partiallyfilledresting"`, not `"open"` — match on both if you are testing for "still working".

For an in-flight order — a `/trade` you just sent — you do **not** poll `orderStatus` to learn the outcome: the synchronous `/trade` response already carries it in its [`response` envelope](post-trade.md#what-accepted-carries). A just-submitted tx that the query view has not yet advanced to reads back `found: false` here until its block is published.

A query that carries neither a parseable `oid` nor a complete `user` + `market_id` + `cloid` triple is rejected with **HTTP 400** and `InvalidOrderStatusQuery`. A malformed `market_id` or `cloid` is rejected the same way, as `InvalidMarketId` / `InvalidCloid`.

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
