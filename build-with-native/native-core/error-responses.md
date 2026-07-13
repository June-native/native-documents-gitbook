---
description: The /trade error model — read the response body, not the HTTP status.
---

# Error responses

`POST /trade` is **synchronous** and always returns the **same JSON trade-response shape**, whatever the HTTP status (`200`, `400`, `429`, `503`, `504`). Branch on the body, not the status line. The envelope carries `submission_status`, and any non-successful outcome carries an `error.code`. A business rejection (below minimum, bad precision, insufficient balance) is **data**, not a transport error: read `error.code` and act on it.

Only the transport layer raises — a non-trade-response `4xx`/`5xx` body, or a wire failure before any response arrived. A decodable trade response is always returned to you as-is.

## submission_status

The call blocks for the on-chain outcome and returns it directly.

| `submission_status` | When | Do next |
| --- | --- | --- |
| `resting` / `filled` / `cancelled` | An order / modify executed and reached this lifecycle state. | Done. Optionally re-read via [`orderStatus`](post-info.md#orderstatus). |
| `rejected` | The write was refused — request-shaping, gateway (rate limit / suspension / expiry), node admission, or an **order that failed at execution**. `error.code` says why; `tx_hash` is present once canonical bytes exist. | If `RateLimited`, back off `error.retry_after_ms` and resend the same signed action. Otherwise fix the cause and submit a **fresh** action. |
| `success` / `failed` | A non-order action (`withdraw` / `settle` / `repay` / `approveAgent` / `revokeAgent`) committed, or failed. `failed` carries `error.code`. | On `failed`, fix and resubmit a fresh action. |
| `timeout` | The outcome wasn't observed within the wait budget, or the submission couldn't be routed to the active node. **May still land.** | Reconcile by `cloid`. **Never** resubmit under a new nonce — double-fill risk. |

{% hint style="warning" %}
`timeout` is not `failed` — the transaction may still commit in a later block. Reconcile by `cloid` (via [`orderStatus`](post-info.md#orderstatus) / [`txStatusByCloid`](post-info.md#orderstatus)); resubmitting under a new nonce is the one move that can double-fill you.
{% endhint %}

A `timeout` has two shapes: the wait budget elapsed → HTTP `200` with **no** `error`; the submission couldn't be routed → HTTP `503`/`504` with an `error.code` (see the gateway codes below).

## Where a code comes from

Every `error.code` comes from one of four layers, and the **spelling tells you which**:

| Layer | Style | Examples | HTTP | `submission_status` |
| --- | --- | --- | --- | --- |
| Request-shaping | lowercase `snake_case` | `invalid_json`, `invalid_quantity_precision`, `missing_cloid` | 400 | `rejected` (no `tx_hash`) |
| Gateway | `CamelCase` / prefixed | `RateLimited`, `PlaceOrderSuspended`, `ExpiredTx`, `HandoffTimeout`, `node_unreachable: …` | 429 / 503 / 504 / 200 | `rejected` or `timeout` |
| Node admission | `CamelCase`, verbatim from the node | `MinTradeSpotNtl`, `DuplicateCloid`, `InsufficientSpotBalance`, `AccountFrozen` | 200 | `rejected` |
| Execution | lowercase (the variant name) | `tick`, `badnonce`, `insufficientspotbalance` | 200 | `rejected` (order) / `failed` (non-order) |

One condition can surface at two layers with different spellings — an under-minimum order is usually caught at admission as `MinTradeSpotNtl`, but the same failure at execution reads `mintradespotntl`. Match on the code you actually receive.

## Errors you'll actually hit

The wire carries `error.code`, not display copy — the **Message** column is illustrative UI text you'd render from the code. The full request-shaping and node-admission code tables are in [transaction-signing.md](transaction-signing.md); this is the curated subset an integrator meets in practice.

| Code | What it means | Message a user sees | Fix |
| --- | --- | --- | --- |
| `RateLimited` | Per-signer request rate exceeded (50 req/s per signer); never admitted. HTTP `429`, carries `error.retry_after_ms`. | "Too many requests — retrying shortly." | Back off `retry_after_ms`, then resend the same signed action. The only safe resend. |
| `PlaceOrderSuspended` | The write path is degraded, so **place-order** actions (`order` / `modify` / a `batch` with one) are suspended; `cancel` / `cancelAll` still go through. HTTP `503`, `error.retry_after_ms: 1000`. | "Placing orders is paused — try again shortly." | Back off and retry; keep cancelling if you need to reduce exposure. |
| `HandoffTimeout` / `HandoffBufferFull:{request_count\|bytes\|signer}` / `HandoffMultipleActive` (`timeout`) | The submission couldn't be routed to a single writable node (leadership handoff / backpressure). `submission_status: "timeout"`, HTTP `503`, `error.retry_after_ms: 1000`. The action may still land. | "Order submitted — confirming status." | Reconcile by `cloid`; never resubmit under a new nonce. |
| `node_unreachable: …` (`timeout`) | Node admission couldn't be reached; `submission_status: "timeout"`, HTTP `504`. The action may still land. | "Order submitted — confirming status." | Reconcile by `cloid`; never resubmit under a new nonce. |
| `MinTradeSpotNtl` | Order, modify replacement, or batch item is below the market's quote-asset minimum notional. Market orders use their protection price. | "Order must have a minimum value of 10 USDC." | Size up so `price × quantity` clears the minimum. |
| `invalid_price_precision` / `invalid_quantity_precision` | `price` / `quantity` has more fractional digits than the market's `price_decimals` / `base_quantity_decimals`. | "Price has too many decimal places for this market." | Snap to market precision before signing; send strings, never floats. |
| `DuplicateCloid` | The `cloid` is already open for this owner and market, or the tx repeats a `(market_id, cloid)`. | "An order with this ID already exists." | Use a fresh `cloid` per order. When reconciling a `timeout`, look the existing one up — don't resend. |
| `ExpiredTx` | The signed `expires_after_ms` had already passed when execution reached the tx. | "Order expired before it was placed." | Widen `expires_after_ms`, re-sign, resubmit. |
| `InsufficientSpotBalance` | Balance-mode precheck found too little available balance for the order reserve. | "Insufficient balance." | Fund the account's quote asset — deposit from your main wallet (no faucet; bring Arbitrum Sepolia testnet assets). |
| `DirectSignerIsActiveAgent` | An active API-wallet key signed in direct-owner mode. An agent key may never sign as an owner. | "This API wallet can't trade as an owner." | Sign in agent mode — pass the owner `accountAddress` alongside the agent key. |
| `AgentEpochMismatch` | The signed `agent_epoch` doesn't match the live agent-slot epoch. | "Session expired — reconnecting." | Re-resolve `agent_epoch` from [`userAgents`](post-info.md#useragents) and resubmit. If it persists, the API wallet was revoked or re-approved — create a new one in the web app. |
| `AccountFrozen` | The account was frozen by an operator; while frozen, only `cancel` / `cancelAll` are admitted — new orders and modifies are rejected. | "Your account is frozen — trading is paused." | Stop placing orders until the freeze is lifted; cancels still go through. |

## Batch

A [`batch`](post-trade.md#batch) is one `/trade` call under one envelope nonce, so the envelope gets **one** `submission_status`. That status reflects the batch's admission and overall outcome — it does **not** report each item. Items execute in array order, and **each item may individually succeed or fail** inside the batch execution result. Reconcile **per item** by `cloid` via [`batchOrderStatus`](post-info.md#batchorderstatus) (up to 20 lookups per call, which covers a full `1..=10`-item batch). Don't infer item outcomes from the envelope status.

## Execution-level failures

An admitted action still runs against the book and **can fail at execution**. Because `/trade` is synchronous, that failure comes back **on the `/trade` response itself** — for an order, `submission_status: "rejected"` with a lowercase `error.code`; for a non-order action, `submission_status: "failed"` with a lowercase `error.code`. The same outcome is also readable afterward via [`orderStatus`](post-info.md#orderstatus) by `cloid`, which you only need when the write came back `timeout`.

Execution codes are the lowercase variant names — e.g. `tick`, `badnonce`, `insufficientspotbalance` — distinct from the CamelCase admission codes above.

| Code | What it means | Fix |
| --- | --- | --- |
| `tick` | A non-integer `price` exceeded the market's `max_price_sig_figs`. The order was admitted, then rejected at execution — `/trade` returns `submission_status: "rejected"` with `error.code: "tick"`. | Snap the price to the market's `price_decimals` / `max_price_sig_figs` before signing. The [Python SDK](python-sdk/README.md) checks this locally (`LocalValidationError`) and never sends it; see [Decimals & units](decimals-units.md#valid-invalid-examples). |

## See also

{% content-ref url="transaction-signing.md" %}
[transaction-signing.md](transaction-signing.md)
{% endcontent-ref %}

{% content-ref url="api-access.md" %}
[api-access.md](api-access.md)
{% endcontent-ref %}
