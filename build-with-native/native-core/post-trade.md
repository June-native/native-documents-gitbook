# POST /trade

Submits one signed action.

```sh
curl -sS -X POST "$API_URL/trade" \
  -H 'content-type: application/json' \
  -H 'x-trace-id: client-trace-001' \
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

| Field              | Required              | Description                                                                                                                                                                                                                                          |
| ------------------ | --------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `action`           | yes                   | Action object. Public top-level types: `order`, `cancel`, `cancelAll`, `modify`, `batch`, `withdraw`, `settle`, `repay`, `approveAgent`, and `revokeAgent`. Internal operator/accounting writes are not part of this public contract. See the full public list below.         |
| `nonce`            | yes                   | Decimal string `u64` Unix millisecond timestamp nonce. Use current `Date.now()`/Unix ms; if sending multiple requests in the same millisecond for the same signer, increment locally so each signed nonce is unique and monotonically nondecreasing. |
| `agent_epoch`      | no                    | Decimal string `u64`; required only for agent-signed requests. Omit for owner-signed requests.                                                                                                                                                       |
| `expires_after_ms` | no                    | Decimal string `u64` Unix milliseconds. An envelope already past `expires_after_ms` at the gateway clock is fast-failed with `submission_status: "rejected"`, `error.code: "ExpiredTx"` (before the node hop); execution also enforces expiry against the committed block timestamp.                                                                                                                                                          |
| `auth_scheme`      | no                    | `"legacy"` (default) or `"eip712"`. Public `withdraw`, `settle`, `repay`, `approveAgent`, and `revokeAgent` require `"eip712"`; public trading actions (`order`/`cancel`/`cancelAll`/`modify`/`batch`) require `"legacy"`. See [EIP-712 signing](transaction-signing.md#eip-712-signing-auth_scheme-eip712).                        |
| `signature`        | yes for public actions | `0x`-prefixed 65-byte recoverable secp256k1 signature. Legacy v1, or — when `auth_scheme="eip712"` — an EIP-712 v4 single signature. Mutually exclusive with `signatures`.                                                                          |
| `signatures`       | no for public actions | Array of `0x`-prefixed 65-byte signatures for an internal multisig request. Public actions reject this field with `signatures_not_allowed_for_action`; internal multisig submissions are not part of this public contract. Mutually exclusive with `signature`. |

The envelope is **strict**: exactly one of `signature` or `signatures` must be present (neither or both → `must provide exactly one of signature or signatures`), and any field not in the table above is rejected as `invalid_json`. Numeric fields (`nonce`, `agent_epoch`, `expires_after_ms`) accept a decimal string **or** an unsigned JSON integer, with the string form preferred above 2^53. Only `action`, `nonce`, `agent_epoch`, and `expires_after_ms` are folded into the signed payload; `auth_scheme`, `signature`, and `signatures` are transport fields that select and carry the proof (see [Transaction Signing](transaction-signing.md)).

Public actions are single-signature; sending `signatures` with a public action is rejected with `signatures_not_allowed_for_action`. The transaction **authority** used for nonce/rate-limit and `txStatusByCloid` is the recovered signer.

Nonce validation is authority-scoped. Execution accepts nonces within the committed block timestamp window (`block_timestamp_ms - 2 days` through `block_timestamp_ms + 1 day`), rejects duplicates, and retains the latest 100 consumed nonces per authority. When the retained window is full, a new nonce must be greater than the current minimum retained nonce.

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

`POST /trade` is **synchronous**: the request blocks while the transaction is admitted, executed on-chain, and the outcome is read back. Typical latency is a block or two; the wait budget is **3 seconds**, so set your client timeout above that or you will abandon replies that were about to arrive.

Response envelope:

```jsonc
{
  "submission_status": "<status>",   // always present
  "tx_hash": "0x…",                  // present once canonical bytes exist; omitted on a request-shaping reject
  "error": {                         // present only on a non-successful outcome
    "code": "<code>",
    "retry_after_ms": 1000           // present only on RateLimited / PlaceOrderSuspended / TooManyPending / Handoff*
  },
  "response": { … }                  // present only on `accepted` — the per-order outcome, see below
}
```

`submission_status` answers **"did the transaction land?"**, and it has exactly three values:

* `accepted` — the transaction landed and reached execution. There is no top-level `error`. **This does not mean the order succeeded** — see [what `accepted` carries](#what-accepted-carries) below.
* `rejected` — the write never reached execution: request-shaping, rate limit, expiry, place-order suspension, or node admission. `error.code` carries the reason; `tx_hash` is present once canonical bytes exist. A handful of envelope-level execution failures also land here — `badnonce`, `badsignature`, `expiredtx`, `malformedtx`, `featuredisabled` — returned in their CamelCase display form (`BadNonce`, …).
* `timeout` — the outcome was not observed within the 3-second budget, or the submission could not be routed. Whether it can still land depends on the code — see [timeout](#timeout-can-it-still-land).

### What `accepted` carries

On `accepted` the reply includes a fourth field, `response`, holding the actual per-order outcome. **You do not need an `/info` round trip to learn whether an order rested or filled.**

`response.type` is the action type (`order`, `cancel`, `cancelAll`, `modify`, `batch`, or `default` for a non-order action). Where the outcome sits depends on the action:

* `order`, `cancel`, `modify` — one `status`, holding one leaf.
* `cancelAll` — `statuses[]`, one leaf per cancelled order. An empty array cancelled nothing and is still a success.
* `batch` — `statuses[]`, one **full sub-response** per item, in item order. Each element repeats the same `{"type", …}` shape, so a batch leaf sits one level deeper, at `statuses[i].status`. See [batch](#batch).

Each leaf is one of four shapes, keyed by its single field:

| Leaf | Meaning |
| --- | --- |
| `{"open":{"oid","cloid"}}` | The order rested on the book |
| `{"filled":{"total_sz","avg_px","oid","cloid"}}` | The order filled. `total_sz` and `avg_px` are display values, formatted exactly as `/info` formats them. |
| `{"cancelled":{"oid","cloid"}}` | The order was cancelled — by an explicit cancel, or by its own time-in-force / self-trade rule |
| `{"error":"<code>"}` | **The order failed at execution** — e.g. `insufficientspotbalance`, `mintradespotntl`, `tick`, `lotsize`, `missingorder` |

{% hint style="warning" %}
A per-order failure keeps `submission_status: "accepted"` and puts the code in the `{"error":…}` leaf, with **no** top-level `error`. Branching on `submission_status` alone reads a rejected order as a success. Always inspect `response`.
{% endhint %}

{% tabs %}
{% tab title="Rested" %}
```json
{
  "submission_status": "accepted",
  "tx_hash": "0x...",
  "response": {
    "type": "order",
    "status": {
      "open": { "oid": 1964626153570560, "cloid": "0x4e5e6ffddbed6ed66c3d02cab8a4cac6" }
    }
  }
}
```
{% endtab %}
{% tab title="Filled" %}
```json
{
  "submission_status": "accepted",
  "tx_hash": "0x...",
  "response": {
    "type": "order",
    "status": {
      "filled": {
        "total_sz": "0.01",
        "avg_px": "1941.34",
        "oid": 1949585043882752,
        "cloid": "0x39791fa07ff1be03663c735d1f9cfd4a"
      }
    }
  }
}
```
{% endtab %}
{% tab title="Failed at execution" %}
```json
{
  "submission_status": "accepted",
  "tx_hash": "0x...",
  "response": {
    "type": "order",
    "status": { "error": "insufficientspotbalance" }
  }
}
```

The transaction landed; the order did not enter the book. There is no top-level `error` — the code is only in the leaf.
{% endtab %}
{% tab title="Rejected" %}
```json
{
  "submission_status": "rejected",
  "tx_hash": "0x...",
  "error": { "code": "MinTradeSpotNtl" }
}
```

A node-admission reject (CamelCase, verbatim): the order notional was below the market's quote-asset minimum. The transaction never reached execution, so there is no `response`.
{% endtab %}
{% tab title="Timeout" %}
```json
{
  "submission_status": "timeout",
  "tx_hash": "0x..."
}
```
{% endtab %}
{% endtabs %}

Request-shaping rejections (e.g. `invalid_quantity_precision`) carry no `tx_hash` because canonical bytes were never assembled; admission rejections include one.

### `timeout` — can it still land?

The `error.code` tells you, and the two cases need opposite handling:

| Code | HTTP | Did it reach a node? | Do next |
| --- | --- | --- | --- |
| *(none)* — the wait budget elapsed | 200 | **Yes.** It was admitted and is executing. | Reconcile by `cloid`. **Never** resubmit under a new nonce. |
| `HandoffBufferFull:{request_count\|bytes\|signer}` | 503 | **No.** Refused before any submission was attempted. | Resubmit. Nothing will be there to reconcile. |
| `HandoffTimeout` / `HandoffMultipleActive` | 503 | **No.** No writable node accepted it. | Resubmit; reconcile first if a duplicate would be costly. |
| `node_unreachable: …` | 504 | **Unknown.** The connection broke mid-submission and the node may already hold it. | Reconcile by `cloid`. **Never** resubmit under a new nonce. |

When in doubt, treat it as the 504 case and reconcile. The [outcomes playbook](handle-timeouts.md#reconciling-a-timeout) has the reasoning behind each row.

Beyond per-action outcomes, the API can refuse a write for operational reasons: `RateLimited` (HTTP 429 — the per-IP budget of 1 request/second, or the per-signer 1000/second, with `error.retry_after_ms`), `TooManyPending` (HTTP 503 with `error.retry_after_ms: 50` — too many synchronous writes are already in flight; retry immediately, it is transient), `PlaceOrderSuspended` (HTTP 503 — while the write path is degraded, only `cancel`/`cancelAll` and an all-cancel `batch` are accepted so you can reduce exposure; `order`, `modify`, any `batch` that mixes in a non-cancel item, and an empty `batch` are refused), `ExpiredTx` (HTTP 200), and the routing codes `HandoffTimeout` / `HandoffBufferFull:{request_count|bytes|signer}` / `HandoffMultipleActive` (HTTP 503) and `node_unreachable` (HTTP 504), which come back as `submission_status: "timeout"`. A request body over 256 KiB is rejected with HTTP 413. See the full `/trade` error-code table in [Error responses](error-responses.md).

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

`cancelAll` carries no `oid` and no `cloid` in the request. The `missing_oid_or_cloid` parse error does not apply to it. Agent signatures are accepted (same allowlist as `cancel`). Submit precheck classifies it (and any pure-`cancelAll` or `cancel`/`cancelAll`-only batch) as a pure cancel: oracle freshness, frozen SpotCreditAccount, mark coverage, quote-min-notional, and duplicate cloid checks are skipped at admission.

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

* `items` must contain `1..=10` items. Anything outside that range fails while the API assembles the canonical bytes, so it comes back `rejected` with [`encode_error: LengthOverflow`](error-responses.md#full-trade-error-code-reference) and no `tx_hash`.
* Items execute in array order.
* The batch has one envelope nonce. Individual items may succeed or fail inside the batch execution result.

#### Reading a batch response

`statuses[]` answers the request item for item. Each element is a **full sub-response**, not a bare leaf, so the outcome you want is at `statuses[i].status`:

```json
{
  "submission_status": "accepted",
  "tx_hash": "0x...",
  "response": {
    "type": "batch",
    "statuses": [
      {
        "type": "order",
        "status": {
          "open": { "oid": 1964626153570560, "cloid": "0x4e5e6ffddbed6ed66c3d02cab8a4cac6" }
        }
      },
      {
        "type": "order",
        "status": { "error": "insufficientspotbalance" }
      }
    ]
  }
}
```

The first item rested; the second never entered the book. `submission_status` stays `accepted` for both, because it describes the envelope and not the items.

A failure leaf carries only `error` — no `oid`, no `cloid`. Array position is your only link back to the item you sent, so keep your own `items` array to match against.

A `cancelAll` item is itself multi-result, so it nests one level further, with bare leaves under its own `statuses[]`:

```json
{
  "type": "cancelAll",
  "statuses": [
    { "cancelled": { "oid": 1964626153571328 } },
    { "cancelled": { "oid": 1949585043882752 } }
  ]
}
```

### withdraw

User single-signature withdrawal (tag 32). On success it debits `amount` from the signer owner's **available** balance. The asset's `withdraw_fee_atoms` is **recorded** (in the event and `/info withdraws`) but **not** deducted; `amount` must be strictly greater than the fee and at least the configured `min_withdraw_atoms` for `(dst_chain_id, asset_id)`. `amount` and `withdraw_nonce` are raw atoms/values. Must use `signature`; `signatures` is rejected (`signatures_not_allowed_for_action`). New requests must include a fixed 16-byte hex `cloid` used only for `txStatusByCloid`; it is not an idempotency key.

Requires `auth_scheme:"eip712"`. See [EIP-712 signing](transaction-signing.md#eip-712-signing-auth_scheme-eip712).

Clients should use the current Unix millisecond timestamp for `withdraw_nonce` and locally increment it if sending multiple withdrawals for the same account in the same millisecond.

```json
{
  "action": {
    "type": "withdraw",
    "asset_id": "1",
    "amount": "500000",
    "dst_chain_id": "1",
    "dst_address": "0xbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbb",
    "withdraw_nonce": "1717000000004",
    "cloid": "0x22222222222222222222222222222222"
  },
  "nonce": "1717000000004",
  "auth_scheme": "eip712",
  "signature": "0x..."
}
```

Withdraw consumes a windowed-unique business nonce with 3-day retention: a nonce at/below the pruned floor or already retained for its account window is rejected (`WithdrawDuplicateNonce`). A failed withdraw burns the envelope `nonce` but not the business nonce, so a retry reuses the business nonce under a new envelope `nonce`.

Node admission also fail-fast rejects withdraw actions that the current committed state already proves invalid: missing accounting config, missing asset/config, invalid account shape, duplicate committed business nonce, withdraw amount/fee/minimum failures, or insufficient withdraw cash. Once a withdraw is accepted into ingress, its business nonce is also held in a live-only pending overlay, so a concurrent replay of the same business nonce is rejected before block inclusion. This overlay is not canonical state and is retired after the accepted transaction's result publishes to QueryView.

Parse errors include `missing_cloid` and `invalid_cloid`. Historical WAL records encoded before this field existed still replay without a cloid and are not queryable by `txStatusByCloid`.

### settle

{% hint style="info" %}
`settle` and `repay` move value between the two account types. A `SpotCreditAccount` is the **credit account**; a balance-mode / cash account is the default **spot account**. See [Account Types](account-types.md).
{% endhint %}

SpotCreditAccount de-risking (tag 33). The signer must be an **Active** `SpotCreditAccount` (the margin owner). It moves `amount` of `asset_id` out of the signer's long margin position (`actual_qty > 0`) into `cash_account`'s **available** balance, requiring the signer's post-position `available_usd >= 0`. `cash_account` may be **any existing balance-mode account** (it must not be a SpotCreditAccount). `asset_id`/`amount` are raw atoms. `cloid` is a **required** 16-byte hex client operation id. Must use `signature`; `signatures` is rejected (`signatures_not_allowed_for_action`).

Requires `auth_scheme:"eip712"`. See [EIP-712 signing](transaction-signing.md#eip-712-signing-auth_scheme-eip712).

```json
{
  "action": {
    "type": "settle",
    "asset_id": "3",
    "amount": "4000",
    "cash_account": "0x1111111111111111111111111111111111111111",
    "cloid": "0x000102030405060708090a0b0c0d0e0f"
  },
  "nonce": "1717000000005",
  "auth_scheme": "eip712",
  "signature": "0x..."
}
```

Parse errors: `missing_cloid` (cloid absent), `invalid_cloid` (not 16 bytes), `invalid_cash_account` (not a 20-byte hex address), `invalid_asset_id`. Execution errors include `InvalidSettle` (signer not a credit account, `cash_account` missing/credit, zero amount, no settleable long, or over-settle), `SpotCreditAccountFrozen` (frozen signer), `OracleMarkPriceMissing` (a residual nonzero-net asset lacks a fresh mark), and `InsufficientSpotCredit` (post `available_usd < 0`). A full settle that clears the asset's net to zero needs no mark.

Node admission may return these same settle errors before block inclusion when the current committed state already proves the settle invalid. Execution remains authoritative for any transaction accepted into ingress.

### repay

SpotCreditAccount de-risking (tag 34). The signer must be a **balance-mode** cash account. It spends `amount` of `asset_id` from the signer's available balance to reduce `margin_account`'s short (`actual_qty < 0`) toward zero. `margin_account` may be **any existing SpotCreditAccount**, Active **or Frozen** (repay does not unfreeze). There is **no** `available_usd` check and **no** oracle dependency — repay strictly de-risks. `asset_id`/`amount` are raw atoms; `cloid` is required. Must use `signature`; `signatures` is rejected.

Requires `auth_scheme:"eip712"`. See [EIP-712 signing](transaction-signing.md#eip-712-signing-auth_scheme-eip712).

```json
{
  "action": {
    "type": "repay",
    "asset_id": "3",
    "amount": "5000",
    "margin_account": "0x2222222222222222222222222222222222222222",
    "cloid": "0x0f0e0d0c0b0a09080706050403020100"
  },
  "nonce": "1717000000006",
  "auth_scheme": "eip712",
  "signature": "0x..."
}
```

Parse errors: `missing_cloid`, `invalid_cloid`, `invalid_margin_account`, `invalid_asset_id`. Execution errors include `InvalidRepay` (signer is a credit account, `margin_account` missing/non-credit, zero amount, no short, or over-repay past zero) and `InsufficientSpotBalance` (signer's cash is too low).

Node admission may return these same repay errors before block inclusion when the current committed state already proves the repay invalid. Execution remains authoritative for any transaction accepted into ingress.

Settle/repay carry **no** business nonce and provide **no** idempotency: the `cloid` is used only for `txStatusByCloid` lookups within the recent query window (see [txStatusByCloid](post-info.md#txstatusbycloid)). The envelope `nonce` is the only replay protection — the same `cloid` resubmitted under a new envelope `nonce` is a distinct transaction. The lookup is keyed on the **recovered signer** (settle → margin owner; repay → cash owner); a counterparty cannot find the tx by `cloid`.

### approveAgent

Approves an agent (API-wallet) signing key on one of the owner's agent slots. **Owner-signed**: sign with the main wallet under `auth_scheme:"eip712"` — an API-wallet key cannot sign it. Carries no `agent_epoch`, takes exactly one `signature`, and cannot appear inside a `batch`. After approval, subsequent agent-signed writes reference the slot's current epoch via `agent_epoch` (read it from [userAgents](post-info.md#useragents)).

| Field      | Required | Values                                                        |
| ---------- | -------- | ------------------------------------------------------------- |
| `type`     | yes      | `"approveAgent"`                                              |
| `slot_id`  | yes      | Owner agent slot, `0`–`3`.                                    |
| `agent`    | yes      | `0x`-prefixed 20-byte agent (API-wallet) signing address.    |

```json
{
  "action": {
    "type": "approveAgent",
    "slot_id": "0",
    "agent": "0xcccccccccccccccccccccccccccccccccccccccc"
  },
  "nonce": "1717000000007",
  "auth_scheme": "eip712",
  "signature": "0x..."
}
```

Parse errors: `invalid_agent_slot` (slot outside `0`–`3`), `invalid_agent` (not a 20-byte hex address). A legacy signature is rejected with `legacy_signature_not_accepted`; supplying `agent_epoch` is rejected with `eip712_agent_epoch_not_allowed`.

### revokeAgent

Clears the agent approval on one owner slot. **Owner-signed** under `auth_scheme:"eip712"`, same constraints as `approveAgent` (no `agent_epoch`, single signature, not batchable). After revocation, agent-signed writes from that key are rejected by node admission.

| Field     | Required | Values                    |
| --------- | -------- | ------------------------- |
| `type`    | yes      | `"revokeAgent"`           |
| `slot_id` | yes      | Owner agent slot, `0`–`3`. |

```json
{
  "action": {
    "type": "revokeAgent",
    "slot_id": "0"
  },
  "nonce": "1717000000008",
  "auth_scheme": "eip712",
  "signature": "0x..."
}
```

Parse errors: `invalid_agent_slot`. Same EIP-712 gating as `approveAgent`.
