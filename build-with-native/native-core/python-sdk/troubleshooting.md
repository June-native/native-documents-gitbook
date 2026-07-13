---
description: Common Native Core Python SDK errors — the surfaced string, what it means, and the fix.
---

# Troubleshooting

A business rejection is **data on the response body**, not an exception (read it with `error_code`, `is_rejected`, `next_action`). The SDK raises only for problems it catches before signing (`LocalValidationError`), a transport failure (`NetworkError`), a signed-then-timed-out write (`SubmissionUncertain`), or a non-trade 4xx/5xx body (`ClientError` / `ServerError`). This page maps the symptom you actually see — a raised exception or a surfaced `error.code` — to its cause and fix.

{% hint style="warning" %}
**Testnet only · pre-1.0.** The Native Core Python SDK is `v0.1.0` (alpha) and currently runs on **testnet only**. The API may change before 1.0; pin an exact version: `pip install native-core-python-sdk==0.1.0`.
{% endhint %}

## Common errors

| You see (surfaced string / class) | Meaning | Fix |
| --- | --- | --- |
| `LocalValidationError: … has more than N decimal places` / `… has more than N significant figures` | `sz` or `limit_px` exceeds the market's quantity precision or the price's `max_price_sig_figs`. The SDK validates before signing and never silently rounds. | Snap the values: `px = info.snap_price(market, price)` and `sz = info.min_order_size(market, px)`. |
| `LocalValidationError: … order notional … is below the minimum of … (use Info.min_order_size to size up)` | `price × quantity` is under the quote asset's minimum notional. | Size up with `info.min_order_size(market, px)`, which returns the smallest size that clears the minimum. |
| `LocalValidationError: invalid tif '…' (expected one of gtc, ioc, fok, alo)` · `invalid order_type '…'` · `market orders require tif ioc or fok, got '…'` | Unknown or mismatched time-in-force / order type. | Use `tif` in `gtc`/`ioc`/`fok`/`alo`; a `market_order` takes only `ioc` or `fok`. |
| `LocalValidationError: decimal values must be str or Decimal, got float` | You passed a `float` for a price or quantity. | Pass a `str` or `Decimal` — `"0.01"` or `Decimal("0.01")`, never `0.01`. |
| `LocalValidationError: wallet 0x… is not an active agent for owner 0x…` (at construction) | The API wallet is not an approved agent on that account — it was revoked or replaced, or the `owner` is wrong. | Confirm with `info.agent_status(owner, agent)` / `info.user_agents(owner)`; create a fresh API wallet in the web app if the key was rotated. |
| `DirectSignerIsActiveAgent` (rejected) | You built `Exchange(wallet, base_url)` with an API wallet key but no `owner`, so the SDK signed as a direct owner — but the API knows the key is an active agent, which may never sign as an owner. | Pass `owner=<accountAddress>`, or use `Exchange.from_bundle(bundle)`, which sets the owner for you. |
| `AgentEpochMismatch` (rejected) | The signed `agent_epoch` was stale relative to the on-chain approval. | The SDK re-resolves the epoch from `userAgents` and retries **once** automatically. If it persists, the API wallet was revoked or re-approved — create a new one in the app. |
| `InsufficientSpotBalance` (rejected) | The account is not funded in the market's quote asset. | Deposit from your main wallet in the web app (no faucet — bring Arbitrum Sepolia testnet assets). The account is created on the first deposit. |
| `SubmissionUncertain` (raised) · `submission_status: "timeout"` | The write was signed and sent, then the wire timed out — the order **may still have landed**. `SubmissionUncertain` carries `.cloid` and `.nonce`; `next_action` returns `RECONCILE_BY_CLOID`. | Resolve the real outcome by cloid: `info.reconcile_by_cloid(user, market, cloid)` (or `wait_for_order`). **Never** resubmit under a fresh nonce — double-fill risk. |
| `submission_status: "rejected"` (with an `error.code`) | A business rejection: your input or account state was refused. It is data, not an exception; `next_action` returns `FIX_AND_RESUBMIT`. | Read `error_code(resp)`, fix the cause, then submit a fresh order. |
| `NetworkError` (raised) | A transport failure (timeout, connection, DNS) before any response arrived. On a **read**, nothing was submitted. | Retry the read. On a **write**, a transport failure surfaces as `SubmissionUncertain` instead — reconcile by cloid, never blind-resubmit. |
| `RateLimited` (rejected) | Rate limited — the request was **never admitted**, so resending is safe. Carries `error.retry_after_ms`; `next_action` returns `BACKOFF_AND_RETRY`. | Sleep `retry_after_ms(resp)` ms, then resend the same order. `is_retryable(resp)` is `True` only here. `retry_on_rate_limit(action)` wraps a write to do exactly this. |

## Reconcile an uncertain write — never resubmit

A wire timeout is the one outcome you must not retry blindly. Catch it, then settle the order by its `cloid`:

```python
from native_core import Exchange, SubmissionUncertain

exchange = Exchange.from_bundle("bundle.json")   # a testnet bundle
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

The same rule applies when `order()` returns `submission_status: "timeout"` rather than raising. For crash safety, generate the `cloid` yourself with `Exchange.random_cloid()`, persist `{intent, cloid}` **before** calling `order(..., cloid=cloid)`, and reconcile every persisted cloid on restart.

## Retry only on RateLimited

`RateLimited` is the only rejection that is safe to resend, because it was never admitted:

```python
from native_core import is_retryable, retry_after_ms
import time

resp = exchange.order(MARKET, is_buy=True, sz="0.01", limit_px="1000.00", tif="gtc")
if is_retryable(resp):                      # True only for RateLimited
    time.sleep(retry_after_ms(resp) / 1000)
    resp = exchange.order(MARKET, is_buy=True, sz="0.01", limit_px="1000.00", tif="gtc")
```

## Gotchas

* **Pass `str` or `Decimal`, never `float`.** `"0.01"` or `Decimal("0.01")`, not `0.01`. A `float` raises `LocalValidationError` before signing; the SDK never rounds silently.
* **One `Exchange` per API wallet.** The nonce is a per-instance, lock-guarded, monotonic ms counter — a single instance is safe to share across threads. Two instances (or two processes) on the same key hand out colliding nonces and draw seemingly random rejections.
* **Testnet only.** This release runs on testnet — build from a `testnet` bundle.
* **The account must be funded to trade.** Deposit from your main wallet in the web app first (no faucet — bring Arbitrum Sepolia testnet assets); the account is created on the first deposit. Otherwise orders come back `InsufficientSpotBalance`.
* **`accepted` is not `filled`.** A raw `order()` / `market_order()` returning `submission_status: "accepted"` only means the order entered the pipeline. Resolve the real state by cloid with `reconcile_by_cloid` (or `wait_for_open` for a resting order) before treating it as done.

## See also

* [../transaction-signing.md](../transaction-signing.md) — the full `POST /trade` error-code tables and signing rules behind these rejections.
* [../decimals-units.md](../decimals-units.md) — market precision (decimal places, significant figures, minimum notional) behind the `LocalValidationError` precision rows.
* [core-concepts.md](core-concepts.md) — the safety contract (accepted vs. filled, reconcile by cloid, one Exchange per wallet) in full.
* [api-reference.md](api-reference.md) — the response helpers (`error_code`, `next_action`, `is_retryable`, …) and the exception hierarchy.
