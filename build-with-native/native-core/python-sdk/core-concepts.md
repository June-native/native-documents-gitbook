---
description: The eight ideas the Native Core Python SDK is built on — read this before wiring a bot.
---

# Core Concepts

The SDK is a thin, typed, synchronous client over the two gateway endpoints. `Info` wraps [`POST /info`](../post-info.md) (market data, balances, order status); `Exchange` wraps [`POST /trade`](../post-trade.md) (one signed action per call) and owns an internal `Info` as `exchange.info`. Both return the gateway's raw JSON as plain dicts. This page expands the eight ideas every integration leans on. It does not re-document the wire fields — for those, see the [`POST /trade`](../post-trade.md) and [`POST /info`](../post-info.md) references.

{% hint style="warning" %}
**Testnet only · pre-1.0.** The Native Core Python SDK is `v0.1.0` (alpha) and currently runs on **testnet only**. The API may change before 1.0; pin an exact version: `pip install native-core-python-sdk==0.1.0`.
{% endhint %}

## Accepted is not filled

A write returns as soon as the transaction enters the node pipeline, not when it executes. `order()` and `market_order()` return `{"submission_status": "accepted", "tx_hash": "…"}` — admitted, but not yet open, filled, or rejected. Resolve the real outcome by `cloid`.

```python
from native_core import Exchange, is_accepted

exchange = Exchange.from_bundle("bundle.json")   # a testnet bundle
resp = exchange.order("ETH/USDT", is_buy=True, sz="0.01", limit_px="1000.00", tif="gtc")

assert is_accepted(resp)                          # in the pipeline — NOT done
# Resolve the settled state by cloid before treating the order as placed:
snapshot = exchange.info.wait_for_open(exchange.effective_account, "ETH/USDT", resp["cloid"])
```

Pick the wait that matches the order's time-in-force:

| Method | Use for | Returns when |
| --- | --- | --- |
| `info.wait_for_open(user, market, cloid)` | a resting order (`gtc` / `alo`) | the order is `open` (or terminal) |
| `info.wait_for_order(user, market, cloid)` | an order that finishes (`ioc` / `fok` / `market`, or after a cancel) | a terminal state such as `filled` or `cancelled` |

`exchange.place(...)` runs the submit and the matching wait in one call, returning `{cloid, submission, status, state}`, so you rarely call either wait directly. Do not call `wait_for_order` on a resting order — a resting order has no terminal state, so the call just times out; that is what `wait_for_open` is for.

## Every order is reconcilable by cloid

Every order carries a client order id (`cloid`). `order()` and `market_order()` return the `cloid` and the `nonce` they used; `batch()` returns `cloids` (one per order leg) plus the shared `nonce`. Pass your own `cloid` or let the SDK generate one — either way the response echoes it, so an order is never handle-less.

If a write times out on the wire, the SDK raises `SubmissionUncertain` — the single most important exception. The order **may still be live**. It carries `.cloid` and `.nonce`; reconcile the outcome, and **never resubmit under a fresh nonce** (that risks a double-fill).

```python
from native_core import Exchange, SubmissionUncertain

try:
    resp = exchange.order("ETH/USDT", is_buy=True, sz="0.01", limit_px="1000.00", tif="gtc")
    cloid = resp["cloid"]
except SubmissionUncertain as exc:
    cloid = exc.cloid                             # signed + sent, outcome unknown — do NOT resend

verdict = exchange.info.reconcile_by_cloid(exchange.effective_account, "ETH/USDT", cloid)
if verdict["undetermined"]:
    ...   # not confirmed within the timeout — keep reconciling by cloid, never re-place
elif verdict["is_filled"]:
    ...   # fully filled
elif verdict["filled_qty"] != "0":
    ...   # partially filled and still resting
else:
    ...   # resting, unfilled (verdict["state"], e.g. "open")
```

`info.reconcile_by_cloid(user, market, cloid)` is the one-call recovery path. It returns `{state, undetermined, is_filled, filled_qty, status}`. It never reports "definitely never landed": order status cannot distinguish a not-yet-indexed order from one that never arrived, so an unconfirmed order stays `undetermined` rather than inviting the forbidden resubmit. The same rule applies to a `submission_status` of `"timeout"` on a response body — reconcile, never resend.

**Survive a restart.** An SDK-generated `cloid` is only known after the call returns; a crash after sending but before recording it cannot be reconciled. For crash safety, generate the `cloid` yourself and persist `{intent, cloid}` durably **before** you send:

```python
from native_core import Exchange

cloid = Exchange.random_cloid()                   # 0x + 16 bytes
persist({"intent": "buy 0.01 ETH/USDT", "cloid": cloid})   # durable, BEFORE sending
resp = exchange.order("ETH/USDT", is_buy=True, sz="0.01", limit_px="1000.00", tif="gtc", cloid=cloid)
# On restart, reconcile every persisted cloid before placing anything new.
```

This is the idempotency-key pattern: the `cloid` is the key, and reconciling it on restart is what makes a retry safe.

## One Exchange per API wallet

The nonce is a **per-instance, lock-guarded, monotonic millisecond counter**. A single `Exchange` is therefore safe to share across threads — but two instances (or two processes) signing with the same API-wallet key hand out colliding nonces and draw seemingly random rejections. Construct one `Exchange` per API wallet and share it; never one per worker.

```python
exchange = Exchange.from_bundle("bundle.json")    # construct once
# ... hand the SAME exchange to every worker thread; do not build a second one on this key.
```

The client is synchronous and poll-based. If you need concurrency, wrap calls in your own executor and still share the one `Exchange`.

## Numbers are strings, never floats

Pass `sz` and `limit_px` (and every price/quantity) as `str` or `Decimal` — `"0.01"`, not `0.01`. Each market has a fixed precision: decimal places, significant figures, and a minimum notional. The SDK validates against that metadata before signing and raises `LocalValidationError` rather than silently round your number down.

```python
from decimal import Decimal

book = info.l2_book("ETH/USDT", depth=1)
ref = book["asks"] or book["bids"]
px = info.snap_price("ETH/USDT", Decimal(ref[0]["price"]) / 2)   # -> wire-ready str
sz = info.min_order_size("ETH/USDT", px)                         # smallest size clearing min-notional
exchange.order("ETH/USDT", is_buy=True, sz=sz, limit_px=px, tif="gtc")
```

`info.snap_price(market, price)` and `info.min_order_size(market, price)` round a value to what the market accepts and return a string. Native executes on integer atoms under the hood — see [Decimals & Units](../decimals-units.md) for the raw/display conversion model that makes floats unsafe.

## Errors live in the response body

Error handling keys on the **response body, not the HTTP status**. The gateway returns the same trade-response shape for HTTP 200/400/429/503/504, with `submission_status` in `{accepted, rejected, timeout}`. A business rejection — `{"submission_status": "rejected", "error": {"code": …}}` — is **data, not an exception**. Read it with the top-level helpers: `is_accepted`, `is_rejected`, `is_timeout`, `error_code`, `retry_after_ms`, `is_retryable`.

`next_action(response)` collapses any trade response into one verdict to branch on (and returns `None` for a non-trade response):

| `next_action` | Situation | What to do |
| --- | --- | --- |
| `POLL_ORDER_STATUS` | `accepted` — in the pipeline | Poll `wait_for_open` / `wait_for_order` (or `reconcile_by_cloid`) for the settled state |
| `BACKOFF_AND_RETRY` | `RateLimited` — never admitted | Sleep `retry_after_ms`, then resend the same order (the only safe resend) |
| `RECONCILE_BY_CLOID` | `timeout` — indeterminate | `reconcile_by_cloid`; **never** resubmit |
| `FIX_AND_RESUBMIT` | other rejection | Fix the input or account state, then submit fresh |

```python
from native_core import next_action, RECONCILE_BY_CLOID

verdict = next_action(resp)
if verdict == RECONCILE_BY_CLOID:
    exchange.info.reconcile_by_cloid(exchange.effective_account, "ETH/USDT", resp["cloid"])
```

Order-side helpers read an order-status snapshot: `order_state`, `is_terminal`, `is_undetermined`, `is_filled`, `filled_quantity`. `as_problem_details(failure)` renders any exception or rejected/timeout body into one flat envelope, and `retry_on_rate_limit(action)` wraps a write to resend only on `RateLimited`. The SDK raises only for a transport failure (`NetworkError`), an uncertain write (`SubmissionUncertain`), a non-trade body (`ClientError` / `ServerError`), or a problem caught before signing (`LocalValidationError`). For the full response shapes and the rejection `error.code` catalog, see [Transaction signing](../transaction-signing.md).

## Markets by symbol or id

Anywhere a `market` argument appears, pass a `"BASE/QUOTE"` symbol (`"ETH/USDT"`) or its integer market id — they are interchangeable. Symbols and precision are fetched once when `Info` is constructed.

```python
market_id = info.resolve_market_id("ETH/USDT")    # -> int, e.g. 2
markets = info.markets()["markets"]               # list; each market carries price_decimals, base_quantity_decimals, max_price_sig_figs
```

`info.resolve_market_id(market)` resolves a symbol (or passes an id through); an unknown symbol raises `LocalValidationError`, not a bare `KeyError`. Per-market precision comes from `info.markets()`; `snap_price` / `min_order_size` apply it for you.

## Agent vs owner mode

The SDK trades with an **API wallet** — a protocol-level agent key scoped only to placing and cancelling orders; it can never move funds. `Exchange.from_bundle(bundle)` builds an **agent-mode** `Exchange`: the bundle's `agentPrivateKey` signs, and its `accountAddress` is the **owner** the orders act on. The owner address is used only locally — it never goes on the wire; the gateway recovers the signer from the signature.

The **agent epoch** is resolved live at construction from `userAgents` (not taken from the bundle, which may be stale), and refreshed once automatically if a write is rejected with `AgentEpochMismatch`. Check approval any time without side effects:

```python
print(exchange.agent_info())                      # {"approved": True, "slot_id": 0, "epoch": 10}
print(exchange.agent_address)                      # the signing (agent) wallet
print(exchange.effective_account)                  # the account orders act on: the owner in agent mode
```

If the API wallet is not an approved agent on that owner, the constructor raises `LocalValidationError`; `agent_info()` reports it as data instead of throwing. Deposits, withdrawals, and `approveAgent` / `revokeAgent` are signed by your **main wallet in the web app**, never in the SDK.

## Testnet only

This release runs on **testnet** (`https://api-test.native.org`) and nothing else. The bundle's `network` is `testnet`, and `from_bundle` picks the gateway from it. An API wallet is created and approved on the testnet site, and is bound to that network.

## Polling & freshness

Reads are **poll-only** over [`POST /info`](../post-info.md). Block time is ~50ms; `wait_for_open` / `wait_for_order` poll internally (starting at 50ms and backing off) against a ~5s default deadline. For a standing reconcile loop — open orders, fills, balances — poll on a **fixed interval** tuned to the strategy, not the block rate: a few hundred milliseconds for a tight quoting loop, low seconds for slower reconciliation. Polling faster than block time only burns requests without seeing new state.

Every read view carries `query_height` and `app_hash` — the committed height the answer reflects. `queryStatus` exposes the retained recent-height window (`oldest_available_height`, `latest_available_height`, `recent_query_window_blocks`), which bounds how far back a windowed read such as `userFills` can reach. Market and asset metadata (`markets`, `assets`) may be served from an in-process cache for up to ~10s, so treat precision and symbol metadata as **slowly-changing** — fetch it once at startup (the SDK does this when `Info` is constructed) — and re-fetch balances, orders, and fills every loop.

## Cancel on disconnect

There is **no server-side scheduled-cancel** (dead-man switch) today, and because the gateway is poll-only it never observes a dropped bot — a crashed or partitioned process leaves its resting orders on the book. Run the switch **client-side**: cancel every open order on shutdown, on any unhandled exception, and on a heartbeat timeout.

```python
try:
    run_strategy(exchange)
finally:
    exchange.cancel_open()   # dead-man switch: nothing left resting after exit
```

`exchange.cancel_open(...)` finds where the effective account actually rests (via `open_orders_all`) and issues one `cancel_all` per market with orders; cancels are always admitted, even for a frozen account. Pass `markets=[...]` to bound the scan when you already know where you traded. `examples/market_maker_bot.py` wires this in a `finally:` shutdown pass — see [examples.md](examples.md) for the runnable pattern.

---

## Next

{% content-ref url="getting-started.md" %}
[getting-started.md](getting-started.md)
{% endcontent-ref %}

{% content-ref url="../transaction-signing.md" %}
[transaction-signing.md](../transaction-signing.md)
{% endcontent-ref %}

{% content-ref url="../decimals-units.md" %}
[decimals-units.md](../decimals-units.md)
{% endcontent-ref %}
