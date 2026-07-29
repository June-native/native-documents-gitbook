---
description: What to do with each /trade outcome — when to resend, when to reconcile, and how to never double-fill.
---

# Handle outcomes & timeouts

`POST /trade` is synchronous and returns the same JSON shape whatever the HTTP status, so **branch on `submission_status` in the body, not the status line.** (The one exception is a body the service rejects before it reaches the handler — over the 256 KiB limit — which comes back as plain text with no `submission_status` at all. Guard your parse.) This guide is the decision playbook; the full code catalog is in [Error responses](error-responses.md).

## Read two fields, not one

`submission_status` tells you whether the **transaction** landed. It does **not** tell you whether the **order** succeeded — that lives in the `response` envelope, present on every `accepted` reply.

| `submission_status` | What happened | What to do |
| --- | --- | --- |
| `accepted` | The transaction landed and reached execution. | **Not done — read `response`.** It carries `{"open":…}`, `{"filled":…}`, `{"cancelled":…}`, or `{"error":"<code>"}`. |
| `rejected` | Refused before execution (shaping / rate limit / suspension / expiry / admission), or an envelope-level execution failure. `error.code` says why. | Fix the cause, submit a **fresh** action. One exception below. |
| `timeout` | The outcome was not observed in the 3-second budget, or the submission could not be routed. | Depends on the code — see [below](#reconciling-a-timeout). |

{% hint style="warning" %}
**`accepted` is not success.** An order that failed at execution — insufficient balance, below minimum notional, off the tick grid — still returns `accepted` with **no** top-level `error`; the code appears only as `response.status.error`. A client that branches on `submission_status` alone records a rejected order as live and will keep quoting against a position it never had.
{% endhint %}

```json
{
  "submission_status": "accepted",
  "tx_hash": "0x...",
  "response": { "type": "order", "status": { "error": "insufficientspotbalance" } }
}
```

The full leaf vocabulary is in [POST /trade](post-trade.md#what-accepted-carries). You only need [`orderStatus`](post-info.md#orderstatus) afterwards to reconcile a `timeout`, or to re-read an order later in its life.

## The one safe resend

`RateLimited` (HTTP 429) is the **only** rejection you resend as-is: back off `error.retry_after_ms`, then send the **same signed action**. Every other `rejected` needs a fresh action after you fix the cause — never blindly resend.

## Reconciling a timeout

Not every `timeout` is indeterminate. The `error.code` tells you whether the transaction ever reached a node, and the two cases need opposite handling.

| Code | HTTP | Reached a node? | What to do |
| --- | --- | --- | --- |
| *(none)* — the wait budget elapsed | 200 | **Yes**, it is executing | Reconcile by `cloid`. **Never** resubmit under a new nonce. |
| `HandoffBufferFull:{request_count\|bytes\|signer}` | 503 | **No** — refused before any submission was attempted | Resubmit. There is nothing to reconcile. |
| `HandoffTimeout` / `HandoffMultipleActive` | 503 | **No** — no writable node accepted it | Resubmit, or reconcile first if a duplicate would be costly (see below). |
| `node_unreachable: …` | 504 | **Unknown** — the connection broke mid-submission | Reconcile by `cloid`. **Never** resubmit under a new nonce. |

Treating the whole 503 family as indeterminate silently drops every write for the duration of a leadership handoff, which is why it is worth separating. Treating the 200 and 504 cases as safe to resubmit is how you double-fill.

{% hint style="info" %}
`HandoffBufferFull` is refused before any node is contacted, so a resubmit cannot duplicate. `HandoffTimeout` and `HandoffMultipleActive` mean every attempt either failed to connect or was explicitly refused — so a resubmit is expected to be safe, but the guarantee rests on the service classifying the connection failure correctly. If a duplicate fill would be expensive for you, reconcile by `cloid` first and resubmit only when the lookup comes back empty; you still recover the order, just one round trip later.
{% endhint %}

When a code is not in this table, reconcile.

To reconcile, look the action up by the `cloid` you sent:

* [`orderStatus`](post-info.md#orderstatus) — an order's current lifecycle by `cloid`.
* [`txStatusByCloid`](post-info.md#orderstatus) — a non-order action (`withdraw` / `settle` / `repay`) by `cloid`, within the recent window.

This is why every order should carry a `cloid` — it is your only handle for reconciliation.

## Batches

A [`batch`](post-trade.md#batch) is one envelope with **one** `submission_status`, but its `response.statuses[]` carries one leaf per item, in item order — so an accepted batch already tells you which items rested, filled, or failed. Reconcile by `cloid` via [`batchOrderStatus`](post-info.md#batchorderstatus) only when the envelope came back `timeout`.

## Next steps

* [POST /trade](post-trade.md#what-accepted-carries) — the full `response` envelope
* [Error responses](error-responses.md) — every code and what it means
* [Trade over REST](trade-over-rest.md) — the happy path
