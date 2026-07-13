---
description: The /trade error model — read the response body, not the HTTP status.
---

# Error responses

A `POST /trade` call returns the **same JSON trade-response shape** whether the HTTP status is `200`, `400`, `429`, `503`, or `504`. Branch on the body, not the status line. The envelope carries `submission_status` — one of `accepted`, `rejected`, `timeout` — and a rejection carries an `error.code`. A business rejection (below minimum, bad precision, insufficient balance) is **data**, not a transport error: read `error.code` and act on it, don't treat it as a failed request.

Only the transport layer raises — a non-trade-response `4xx`/`5xx` body, or a wire failure before any response arrived. A decodable trade response is always returned to you as-is.

{% tabs %}
{% tab title="accepted" %}
Admitted to the node pipeline. **Not** filled — execution is asynchronous.

```json
{
  "submission_status": "accepted",
  "tx_hash": "0x..."
}
```

Poll [`orderStatus`](post-info.md#orderstatus) by `cloid` for the settled outcome.
{% endtab %}

{% tab title="rejected" %}
Never admitted. `error.code` says why.

```json
{
  "submission_status": "rejected",
  "tx_hash": "0x...",
  "error": {
    "code": "MinTradeSpotNtl"
  }
}
```

A `RateLimited` rejection additionally carries `error.retry_after_ms`. Request-local rejections (validation before canonical bytes are assembled) usually omit `tx_hash`; rejections after byte assembly include it.
{% endtab %}

{% tab title="timeout" %}
Node admission could not be reached. HTTP `504`. The action **may still land**.

```json
{
  "submission_status": "timeout",
  "error": {
    "code": "node_unreachable: ..."
  }
}
```

Reconcile by `cloid` — never resubmit under a new nonce.
{% endtab %}
{% endtabs %}

## Outcomes

| `submission_status` | Meaning | Do next |
| --- | --- | --- |
| `accepted` | Entered the pipeline — **not** filled. | Poll [`orderStatus`](post-info.md#orderstatus) by `cloid` until it reaches a resting or terminal state. |
| `rejected` + `RateLimited` | Throttled; never admitted. | The **only** safe resend. Back off `error.retry_after_ms`, resend the same signed action (same `cloid`). |
| `rejected` (other) | Business rejection. | Fix the cause, then submit a **fresh** action. |
| `timeout` | Indeterminate — may still land. | Reconcile by `cloid`. **Never** resubmit — a fresh nonce risks a double-fill. |

{% hint style="warning" %}
`accepted` is not `filled`, and `timeout` is not `failed`. Both require a follow-up read keyed on `cloid`. Resubmitting a `timeout` under a new nonce is the one move that can double-fill you.
{% endhint %}

## Errors you'll actually hit

The wire carries `error.code`, not display copy — the **Message** column is illustrative UI text you'd render from the code. The full request-shaping and node-admission code tables are in [transaction-signing.md](transaction-signing.md); this is the curated subset an integrator meets in practice.

| Code | What it means | Message a user sees | Fix |
| --- | --- | --- | --- |
| `RateLimited` | Per-signer request rate exceeded; never admitted. Carries `error.retry_after_ms`. | "Too many requests — retrying shortly." | Back off `retry_after_ms`, then resend the same signed action. The only safe resend. |
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

A [`batch`](post-trade.md#batch) is one `/trade` call under one envelope nonce, so the envelope gets **one** `submission_status`. `accepted` means the batch entered the pipeline — it does **not** mean every item succeeded. Items execute in array order, and **each item may individually succeed or fail** inside the batch execution result. Reconcile **per item** by `cloid` via [`batchOrderStatus`](post-info.md#batchorderstatus) (up to 20 lookups per call, which covers a full `1..=10`-item batch). Don't infer item outcomes from the envelope status.

## Execution-level failures

Every code above is an **admission** outcome — the gateway returns it on the `/trade` response (or as `submission_status`). Execution is a separate, later stage: an `accepted` action still runs against the book and **can fail there**. An execution failure does **not** appear on the `/trade` response — you discover it by polling [`orderStatus`](post-info.md#orderstatus) by `cloid` (an `accepted` order that later reads back as failed).

| Code | What it means | Fix |
| --- | --- | --- |
| `tick` | A non-integer `price` exceeded the market's `max_price_sig_figs`. The action was admitted, then rejected at execution — `/trade` returned `accepted`, and the failure shows up on `orderStatus`. | Snap the price to the market's `price_decimals` / `max_price_sig_figs` before signing. The [Python SDK](python-sdk/README.md) checks this locally (`LocalValidationError`) and never sends it; see [Decimals & units](decimals-units.md#valid-invalid-examples). |

## See also

{% content-ref url="transaction-signing.md" %}
[transaction-signing.md](transaction-signing.md)
{% endcontent-ref %}

{% content-ref url="api-access.md" %}
[api-access.md](api-access.md)
{% endcontent-ref %}
