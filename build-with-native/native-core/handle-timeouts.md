---
description: What to do with each /trade outcome — when to resend, when to reconcile, and how to never double-fill.
---

# Handle outcomes & timeouts

`POST /trade` is synchronous and always returns the same shape, whatever the HTTP status. **Branch on `submission_status` in the body, not the status line.** This guide is the decision playbook; the full code catalog is in [Error responses](error-responses.md).

## The three outcomes

| `submission_status` | What happened | What to do |
| --- | --- | --- |
| `accepted` | The transaction landed and executed. | Done. To see whether an order rested or filled, read [`orderStatus`](post-info.md#orderstatus) — the response doesn't carry the fill state. |
| `rejected` | Refused (shaping / rate limit / suspension / expiry / admission) or failed at execution. `error.code` says why. | Fix the cause, submit a **fresh** action. One exception below. |
| `timeout` | Outcome not observed in the wait budget, or couldn't be routed. **May still land.** | Reconcile by `cloid`. **Never** resubmit under a new nonce. |

## The one safe resend

`RateLimited` (HTTP 429) is the **only** rejection you resend as-is: back off `error.retry_after_ms`, then send the **same signed action**. Every other `rejected` needs a fresh action after you fix the cause — never blindly resend.

## Reconciling a timeout

A `timeout` is indeterminate: the transaction may still commit in a later block. Resubmitting under a new nonce is the one move that can double-fill you. Instead, look it up by the `cloid` you sent:

* [`orderStatus`](post-info.md#orderstatus) — an order's current lifecycle by `cloid`.
* [`txStatusByCloid`](post-info.md#orderstatus) — a non-order action (`withdraw` / `settle` / `repay`) by `cloid`, within the recent window.

This is why every order should carry a `cloid` — it is your only handle for reconciliation.

## Batches

A [`batch`](post-trade.md#batch) is one envelope with **one** `submission_status` — it does not report per item. Items succeed or fail individually inside the batch. Reconcile **per item** by `cloid` via [`batchOrderStatus`](post-info.md#batchorderstatus).

## Next steps

* [Error responses](error-responses.md) — every code and what it means
* [Trade over REST](trade-over-rest.md) — the happy path
