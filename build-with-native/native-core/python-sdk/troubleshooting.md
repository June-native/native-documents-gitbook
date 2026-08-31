---
description: Common Native Core Python SDK errors — the surfaced string, what it means, and the fix.
---

# Troubleshooting

A business rejection is **data on the response body**, not an exception (read it with `error_code`, `is_rejected`, `next_action`). The SDK raises for problems it catches locally (`LocalValidationError` — before signing a write, and also on a read whose inputs or the API's own state make the call unanswerable), a transport failure (`NetworkError`), a signed write whose outcome cannot be read (`SubmissionUncertain`), a refused WebSocket subscription (`SubscriptionError`), and a body that is not a trade response (`ClientError` / `ServerError` — usually a 4xx/5xx, but `ClientError` also covers a refused `/info` read that arrives with **HTTP 200**). This page maps the symptom you actually see — a raised exception or a surfaced `error.code` — to its cause and fix.

## Common errors

| You see (surfaced string / class) | Meaning | Fix |
| --- | --- | --- |
| `LocalValidationError: … has more than N decimal places` / `… has more than N significant figures` | `sz` or `limit_px` exceeds the market's quantity precision or the price's `max_price_sig_figs`. The SDK validates before signing and never silently rounds. | Snap the values: `px = info.snap_price(market, price)` and `sz = info.min_order_size(market, px)`. |
| `LocalValidationError: … order notional … is below the minimum of … (use Info.min_order_size to size up)` | `price × quantity` is under the quote asset's minimum notional. | Size up with `info.min_order_size(market, px)`, which returns the smallest size that clears the minimum. |
| `LocalValidationError: invalid tif '…' (expected one of gtc, ioc, fok, alo)` · `invalid order_type '…'` · `market orders require tif ioc or fok, got '…'` | Unknown or mismatched time-in-force / order type. | Use `tif` in `gtc`/`ioc`/`fok`/`alo`; a `market_order` takes only `ioc` or `fok`. |
| `LocalValidationError: decimal values must be str or Decimal, got float` | You passed a `float` for a price or quantity. | Pass a `str` or `Decimal` — `"0.01"` or `Decimal("0.01")`, never `0.01`. |
| `LocalValidationError: wallet 0x… is not an active agent for owner 0x…` (at construction) | The API wallet is not an approved agent on that account — it was revoked or replaced, or the `owner` is wrong. | Confirm with `info.agent_status(owner, agent)` / `info.user_agents(owner)`; create a fresh API wallet in the web app if the key was rotated. |
| `LocalValidationError: no order book available for … the market is unknown, or the gateway's query view is not ready yet` · `no ask/bid on … to derive a protection price from` | `protection_price` could not read a book. `no order book available` means the market id is unknown. `no ask/bid on …` means the market exists but has nothing resting on the side you would cross — which is every market for a few seconds after a restart, since a book is not part of the startup snapshot. | Check the market id for the first. For the second, retry after a moment, or pass `ref_price=` to skip the book read. Catch it as `native_core.Error`; in 1.x this was a bare `KeyError` that `except Error` missed. |
| `LocalValidationError: start_height … precedes the retained window (oldest …); those fills are pruned` | `iter_user_fills` was asked to start before the retained block window. | Read `info.query_status()` for `oldest_available_height`, or pass `clamp=True` to follow the window as it advances. |
| `ClientError` with `status_code` **200** (raised from a fills read) | An `/info` query fell outside the retained block window. The API answers HTTP 200 with an `error` object *and* an empty list, so a naive read of `["fills"]` sees "no trading happened". | `iter_user_fills` / `recent_fills` already raise here. Elsewhere, check `info_error(response)` before trusting an empty list. |
| `LocalValidationError: this connection already holds 10 subscriptions…` · `SubscriptionError: Too many subscriptions` · `no answer to subscribe within …s` | A subscribe was refused. Ten per connection is the cap; the SDK refuses locally first, and the server's own refusal appears only if you raised `max_subscriptions`. One connection per client IP is configured but currently counted rather than enforced. | Multiplex onto the one socket instead of opening more. Carries `.reason` and `.subscription`; see [Streaming](streaming.md). |
| `DirectSignerIsActiveAgent` (rejected) | You built `Exchange(wallet, base_url)` with an API wallet key but no `owner`, so the SDK signed as a direct owner — but the API knows the key is an active agent, which may never sign as an owner. | Pass `owner=<accountAddress>`, or use `Exchange.from_bundle(bundle)`, which sets the owner for you. |
| `AgentEpochMismatch` (rejected) | The signed `agent_epoch` was stale relative to the on-chain approval. | The SDK re-resolves the epoch from `userAgents` and retries **once** automatically. If it persists, the API wallet was revoked or re-approved — create a new one in the app. |
| `InsufficientSpotBalance` (rejected) · `insufficientspotbalance` (leaf) | The account is not funded in the market's quote asset. **The same condition has two spellings**: CamelCase on a `rejected` body when admission catches it, lowercase on the `response` envelope's leaf when execution does. | Deposit the quote asset from your main wallet in the web app; the account is created on the first deposit. Check both `error_code(resp)` and `leaf_error_code(resp)`. |
| `submission_status: "accepted"` with a failing leaf | **The transaction landed; your order did not.** `tick`, `lotsize`, `badalopx`, `insufficientspotbalance`, `mintradespotntl` and `missingorder` all arrive this way. `is_accepted` is `True`, `is_order_failed` is `True`, `next_action` returns `FIX_AND_RESUBMIT`. | Read `leaf_error_code(resp)` and fix the input or the account state. Do **not** poll or reconcile: a failed order is never written to `/info`, so the leaf is the only trace it existed. What you send next is a fresh order with a fresh `cloid`. |
| `SubmissionUncertain` (raised) · `submission_status: "timeout"` | A signed write whose outcome cannot be read — the order **may still have landed**. Raised on an HTTP transport failure, and on any 5xx or dropped connection under `WsClient.post_action`. Carries `.cloids` (a list; `.cloid` is the first, `None` for a handle-less action) and `.nonce`. | Reconcile **every** entry in `.cloids` with `info.reconcile_by_cloid(user, market, cloid)` — a batch has one per **order** leg, so the list is shorter than the batch when it also carries cancels. Not `wait_for_order`: a resting order has no terminal state, so it can only time out. **Never** resubmit, unless `is_safe_to_resend(resp)` is `True` (the `Handoff*` family), where `next_action` returns `BACKOFF_AND_RETRY` and resending the same `cloid` is correct. |
| `submission_status: "rejected"` (with an `error.code`) | A business rejection: your input or account state was refused. It is data, not an exception; `next_action` returns `FIX_AND_RESUBMIT`, or `BACKOFF_AND_RETRY` when the code is `RateLimited` (see below). | Read `error_code(resp)`, fix the cause, then submit a fresh order. |
| `NetworkError` (raised) | A transport failure (timeout, connection, DNS) before any response arrived. On a **read**, nothing was submitted. | Retry the read. On a **write**, a transport failure surfaces as `SubmissionUncertain` instead — reconcile by cloid, never blind-resubmit. |
| `RateLimited` (rejected) | Rate limited — the request was **never admitted**, so resending is safe. Carries `error.retry_after_ms`; `next_action` returns `BACKOFF_AND_RETRY`. | Sleep `retry_after_ms(resp)` ms, then resend the same order. `is_retryable(resp)` is `True` only here. `retry_on_rate_limit(action)` wraps a write to do exactly this. |

## An accepted write whose order failed

The most common way to get this wrong. `submission_status: "accepted"` says the **transaction** landed and executed, nothing about your order:

```python
from native_core import error_code, is_accepted, is_order_failed, leaf_error_code, order_oid

resp = exchange.order(MARKET, is_buy=True, sz="0.01", limit_px="1700.005", tif="alo")

is_accepted(resp)        # True  — the transaction landed
is_order_failed(resp)    # True  — but the order did not: the price is off-tick
leaf_error_code(resp)    # "tick"
```

Gate every write on both checks, the way the SDK's own examples do:

```python
if not (is_accepted(resp) and not is_order_failed(resp)):
    raise SystemExit(f"write failed: {leaf_error_code(resp) or error_code(resp)}")
oid = order_oid(resp)     # free — no /info read
```

There is nothing to reconcile here. A failed order never entered the book, so `orderStatus` returns `found: false` and `reconcile_by_cloid` polls to its deadline and reports `undetermined` for an order you already know is dead. Fix the input and send a **fresh** order with a fresh `cloid`.

An unfilled IOC is not a failure: `ioccancel`, `fokcancel`, `selftradepreventioncancel` and `marketordernoliquidity` are benign cancels, which `is_order_failed` already excludes and `is_benign_cancel(code)` identifies.

## Reconcile an uncertain write — never resubmit

A wire timeout is the one outcome you must not retry blindly. Catch it, then settle the order by its `cloid`:

```python
from native_core import Exchange, SubmissionUncertain

exchange = Exchange.from_bundle("bundle.json")   # your connection bundle
MARKET = "ETH/USDT"

try:
    resp = exchange.order(MARKET, is_buy=True, sz="0.01", limit_px="1000.00", tif="gtc")
except SubmissionUncertain as e:
    # Signed and sent, then timed out. The order MAY be live — do NOT re-place.
    verdict = exchange.info.reconcile_by_cloid(exchange.effective_account, MARKET, e.cloid)
    if verdict["undetermined"]:
        ...   # still not confirmed: keep reconciling by cloid, never resubmit
    elif verdict["is_filled"]:
        ...   # fully filled
    elif verdict["filled_qty"] != "0":
        ...   # partially filled and resting — verdict["filled_qty"] is how much
    else:
        ...   # resting, unfilled (verdict["state"], e.g. "open")
```

The same applies when `order()` returns `submission_status: "timeout"` rather than raising — **with one exception**. `is_safe_to_resend(resp)` is `True` for `HandoffTimeout`, `HandoffMultipleActive` and `HandoffBufferFull:*`, which prove the transaction never reached a node: back off by `retry_after_ms(resp)` and send the same `cloid` again. Treating those as indeterminate silently discards orders that were safe to resend.

For crash safety, generate the `cloid` yourself with `Exchange.random_cloid()`, persist `{intent, cloid}` **before** calling `order(..., cloid=cloid)`, and resolve every persisted cloid on restart — `info.batch_order_status([...])` settles up to 20 in one read.

## Retry only on RateLimited

`RateLimited` is the only **rejection** that is safe to resend, because it was never admitted. Resend the *same* signed request, which means a fixed `cloid` — `order()` mints a new one on every call, so a bare re-call is a different order, not a retry:

```python
import time

from native_core import Exchange, is_retryable, next_action, retry_after_ms, retry_on_rate_limit

cloid = Exchange.random_cloid()              # one handle, reused by the retry
resp = retry_on_rate_limit(
    lambda: exchange.order(MARKET, is_buy=True, sz="0.01", limit_px="1000.00", tif="gtc", cloid=cloid)
)
print(next_action(resp))                     # branch on the result — do not drop it
```

By hand, the same thing: `is_retryable(resp)` is `True` only for `RateLimited`, and `retry_after_ms(resp)` is the wait.

{% hint style="info" %}
This does not apply to a write sent over `WsClient.post_action`. There a rate limit arrives as a raised `ClientError` with `status_code` 429, not as a response body, so `is_retryable` never sees it and `retry_on_rate_limit` re-raises without retrying. Catch `ClientError` and back off yourself, or submit over HTTP with `Exchange.order`.
{% endhint %}

## Gotchas

* **Pass `str` or `Decimal`, never `float`.** `"0.01"` or `Decimal("0.01")`, not `0.01`. A `float` raises `LocalValidationError` before signing; the SDK never rounds silently.
* **One `Exchange` per API wallet.** The nonce is a per-instance, lock-guarded, monotonic ms counter — a single instance is safe to share across threads. Two instances (or two processes) on the same key hand out colliding nonces and draw seemingly random rejections.
* **Match the network.** Build from the bundle whose `network` (`mainnet` or `testnet`) you mean to trade — a key signed for one network is rejected on the other.
* **The account must be funded to trade.** Deposit from your main wallet in the web app first — the account is created on the first deposit (on testnet there's no faucet; bring Arbitrum Sepolia assets). Otherwise orders come back `InsufficientSpotBalance`.
* **`accepted` is not `placed`.** `submission_status: "accepted"` is a verdict on the **transaction**. Your order may still have failed inside it, on a leaf of the `response` envelope. Check `is_order_failed(resp)` — never `is_accepted` alone — and take the `oid` and fill from `order_oid(resp)` / `fill(resp)` rather than spending an `/info` read on them.
* **Check `found`, then check the levels.** `l2_book` omits `bids` and `asks` **entirely** when `found` is false, so `book["asks"]` raises a bare `KeyError` that `except Error` cannot catch. `found: false` means the market id is unknown. A market with no book yet — every market for a few seconds after a restart — answers `found: true` with both sides empty, so guard for that separately.

## See also

* [Transaction Signing](../transaction-signing.md) — the full `POST /trade` error-code tables and signing rules behind these rejections.
* [Decimals & Units](../decimals-units.md) — market precision (decimal places, significant figures, minimum notional) behind the `LocalValidationError` precision rows.
* [Core Concepts](core-concepts.md) — the safety contract (accepted vs. placed, reconcile by cloid, one Exchange per wallet) in full.
* [API Reference](api-reference.md) — the response helpers (`error_code`, `next_action`, `is_retryable`, …) and the exception hierarchy.
