---
description: The core ideas the Native Core Python SDK is built on — read this before wiring a bot.
---

# Core Concepts

A thin, typed, synchronous client over Native Core. Three classes, all returning the API's raw JSON as plain dicts:

| Class | Wraps |
| --- | --- |
| `Info` | [`POST /info`](../post-info.md) — market data, balances, order status |
| `Exchange` | [`POST /trade`](../post-trade.md) — one signed action per call. Owns an `Info` as `exchange.info` |
| `WsClient` | The [WebSocket](../websocket.md) — the same data, pushed |

This page is the ideas every integration leans on, not the wire fields. Those are in the [`POST /trade`](../post-trade.md) and [`POST /info`](../post-info.md) references.

## Accepted is not placed

`order()` and `market_order()` return `{"submission_status": "accepted", "tx_hash": "…", "response": {…}}` once the **transaction** has landed and executed. That is a verdict on the transaction, not on your order. Three different outcomes hide behind it:

| What actually happened | How to read it |
| --- | --- |
| The order rested or filled | `order_oid(resp)` for the `oid`; `fill(resp)` for `{total_sz, avg_px, oid}`, or `None` if nothing filled |
| A benign cancel: an IOC/FOK that found no liquidity, a self-trade-prevention cancel | `is_benign_cancel(leaf_error_code(resp))` |
| The order **failed**: `tick`, `lotsize`, `insufficientspotbalance`, `mintradespotntl`, `missingorder`, `badalopx` | `is_order_failed(resp)`, then `leaf_error_code(resp)` for the code |

Only six envelope-level problems (bad nonce, bad signature, expired tx, malformed tx, bad batch length, feature disabled) and the API's admission refusals come back `rejected`. Every other way an order can go wrong arrives **inside an accepted response**, on the failing action's leaf.

### What a failed order leaves behind

Whether you can look the order up afterwards depends on how far it got.

| Leaf code | What it leaves behind |
| --- | --- |
| `tick` `lotsize` `insufficientspotbalance` `mintradespotntl` `missingorder` | **Nothing.** The leaf is the only trace. Don't poll, don't reconcile |
| `badalopx` `insufficientspotcredit` | An [`orderStatus`](../post-info.md#orderstatus) row and an `orderUpdates` frame |
| `ioccancel` `fokcancel` `marketordernoliquidity` | The same, and benign: the order simply didn't fill |

Either way, `is_accepted` alone books a failed order as a successful write.

```python
from native_core import Exchange, is_accepted, is_order_failed, leaf_error_code, order_oid, fill

exchange = Exchange.from_bundle("bundle.json")   # your connection bundle
resp = exchange.order("ETH/USDT", is_buy=True, sz="0.01", limit_px="1000.00", tif="gtc")

if not is_accepted(resp):
    ...                                           # rejected or timeout — see below
elif is_order_failed(resp):
    raise SystemExit(f"order failed: {leaf_error_code(resp)}")   # e.g. "tick"
else:
    oid = order_oid(resp)                         # assigned id, straight off the write
    filled = fill(resp)                           # {total_sz, avg_px, oid}, or None
```

The `oid` and the fill ride back on the response, so the ordinary path costs **no** `/info` read. Reads are capped at one per second per client IP ([rate limits](../api-access.md#rate-limits-errors)), which is what makes that worth doing.

You still need a read to follow an order's later life, such as a resting bid that fills minutes after you placed it. Pick the wait that matches the order's time-in-force:

| Method | Use for | Returns when |
| --- | --- | --- |
| `info.wait_for_open(user, market, cloid)` | a resting order (`gtc` / `alo`) | the order is `open` (or terminal) |
| `info.wait_for_order(user, market, cloid)` | an order that finishes (`ioc` / `fok` / `market`, or after a cancel) | a terminal state such as `filled` or `cancelled` |

`exchange.place(...)` submits and runs the matching wait in one call, returning `{cloid, submission, status, state, oid}`. You rarely need either wait directly.

It skips the wait when the response already settled the order: a rejection, a timeout, a failed order, or a benign cancel. `status` and `state` are then `None`, so branch on `submission` or on `next_action` instead.

Never call `wait_for_order` on a resting order. A resting order has no terminal state, so the call only times out. `wait_for_open` is the one for that.

## Reconcile by cloid, but only what is unresolved

Every order carries a client order id (`cloid`). `order()` and `market_order()` return the `cloid` and the `nonce` they used; `batch()` returns `cloids` (one per order leg) plus the shared `nonce`. Pass your own `cloid` or let the SDK generate one — either way the response echoes it, so an order is never handle-less.

Reconcile in one situation only: **the response never told you what happened.** The SDK raises `SubmissionUncertain` there, carrying `.cloids` and `.nonce`. The order may still be live, so reconcile it and **never resubmit under a fresh nonce** — that risks a double-fill.

Three things raise it:

* a transport failure over HTTP
* a **5xx** answer to a write sent over the WebSocket
* a connection that dies before the answer arrives

A 4xx over the WebSocket raises `ClientError` instead. It cannot have executed.

{% hint style="warning" %}
Do **not** reconcile an order the response already settled. For a failure that [leaves no record](#what-a-failed-order-leaves-behind), `reconcile_by_cloid` polls to its deadline and reports `undetermined` — and the never-resubmit rule then freezes that order forever.

Read `is_order_failed` first. A failed order is finished; the corrected one you send next is a **fresh** order with a fresh `cloid`.
{% endhint %}

```python
from native_core import Exchange, SubmissionUncertain, is_accepted, is_order_failed, leaf_error_code

try:
    resp = exchange.order("ETH/USDT", is_buy=True, sz="0.01", limit_px="1000.00", tif="gtc")
except SubmissionUncertain as exc:
    resp, cloid = None, exc.cloid                 # signed + sent, outcome unknown — do NOT resend

if resp is not None and is_accepted(resp) and is_order_failed(resp):
    raise SystemExit(f"order failed: {leaf_error_code(resp)}")   # settled — nothing to reconcile
if resp is not None:
    cloid = resp["cloid"]

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

`info.reconcile_by_cloid(user, market, cloid)` is the one-call recovery path. It returns `{state, undetermined, is_filled, filled_qty, status}`. It never reports "definitely never landed": order status cannot distinguish a not-yet-indexed order from one that never arrived, so an unconfirmed order stays `undetermined` rather than inviting the forbidden resubmit.

A `submission_status` of `"timeout"` follows the same rule, **with one exception**. `is_safe_to_resend(resp)` is true for `HandoffTimeout`, `HandoffMultipleActive` and `HandoffBufferFull:*`, which prove the transaction was never admitted by a node, so it cannot have executed: sleep `retry_after_ms(resp)` and resend the same `cloid`. Every other timeout, including the plain wait-budget one that carries no error code, is indeterminate — reconcile, never resend.

**Survive a restart.** An SDK-generated `cloid` is only known after the call returns; a crash after sending but before recording it cannot be reconciled. For crash safety, generate the `cloid` yourself and persist `{intent, cloid}` durably **before** you send:

```python
from native_core import Exchange

cloid = Exchange.random_cloid()                   # 0x + 16 bytes
persist({"intent": "buy 0.01 ETH/USDT", "cloid": cloid})   # durable, BEFORE sending
resp = exchange.order("ETH/USDT", is_buy=True, sz="0.01", limit_px="1000.00", tif="gtc", cloid=cloid)
# On restart, resolve every persisted cloid before placing anything new.
```

This is the idempotency-key pattern: the `cloid` is the key, and resolving it on restart is what makes a retry safe. Resolve them in bulk — `info.batch_order_status([...])` takes up to 20 selectors in **one** read, against a budget of one read per second — and fall back to `reconcile_by_cloid` only for the ones that come back unresolved.

## One Exchange per API wallet

The nonce is a **per-instance, lock-guarded, monotonic millisecond counter**. A single `Exchange` is therefore safe to share across threads — but two instances (or two processes) signing with the same API-wallet key hand out colliding nonces and draw seemingly random rejections. Construct one `Exchange` per API wallet and share it; never one per worker.

```python
exchange = Exchange.from_bundle("bundle.json")    # construct once
# ... hand the SAME exchange to every worker thread; do not build a second one on this key.
```

The client is synchronous (blocking). If you need concurrency, wrap calls in your own executor and still share the one `Exchange`.

## Numbers are strings, never floats

Pass `sz` and `limit_px` (and every price/quantity) as `str` or `Decimal` — `"0.01"`, not `0.01`. Each market has a fixed precision: decimal places, significant figures, and a minimum notional. The SDK validates against that metadata before signing and raises `LocalValidationError` rather than silently round your number down.

```python
from decimal import Decimal

book = info.l2_book("ETH/USDT", depth=1)
if not book.get("found"):
    raise RuntimeError("no book yet")                            # found=false omits bids/asks entirely
ref = book.get("asks") or book.get("bids")
px = info.snap_price("ETH/USDT", Decimal(ref[0]["price"]) / 2)   # -> wire-ready str
sz = info.min_order_size("ETH/USDT", px)                         # smallest size clearing min-notional
exchange.order("ETH/USDT", is_buy=True, sz=sz, limit_px=px, tif="gtc")
```

Guard the book in two steps, not one.

`found: false` means the market id is unknown, and the response omits `bids` and `asks` **entirely** — the keys are absent, not empty, so `book["asks"]` raises a bare `KeyError` that no `except Error` catches.

`found: true` with both sides empty is the other case: a real market with nothing resting, which is every market for a few seconds after a restart.

`info.snap_price(market, price)` and `info.min_order_size(market, price)` round a value to what the market accepts and return a string. Native executes on integer atoms under the hood — see [Decimals & Units](../decimals-units.md) for the raw/display conversion model that makes floats unsafe.

## Errors live in the response body

Error handling keys on the **response body, not the HTTP status**. The API returns the same trade-response shape for HTTP 200/400/429/503/504, with `submission_status` in `{accepted, rejected, timeout}`. A business rejection — `{"submission_status": "rejected", "error": {"code": …}}` — is **data, not an exception**. Two families of helper read it, and you need both:

| Reads | Helpers |
| --- | --- |
| The **transaction** | `is_accepted` `is_rejected` `is_timeout` `error_code` `retry_after_ms` `is_retryable` `is_safe_to_resend` |
| The **order** inside it | `is_order_failed` `leaf_error_code` `leaf_errors` `is_benign_cancel` `order_oid` `fill` `batch_legs` `trade_envelope` `trade_outcomes` |

The first family cannot see a failed order. The second exists only on an accepted write.

`next_action(response)` folds both families into one verdict to branch on (and returns `None` for a non-trade response):

| `next_action` | Situation | What to do |
| --- | --- | --- |
| `USE_RESPONSE_OUTCOME` | accepted, the order worked | Nothing more. The `oid` and the fill are already on the response |
| `ORDER_CLOSED_UNFILLED` | accepted, benign cancel | Nothing more. The order is over and nothing filled |
| `FIX_AND_RESUBMIT` | accepted but the order failed, or a rejection other than `RateLimited` | Fix the input or the account state. There is nothing to reconcile; what you send next is a fresh order |
| `BACKOFF_AND_RETRY` | `RateLimited`, or a `Handoff*` timeout — never reached a node | Sleep `retry_after_ms`, then resend the **same** `cloid` |
| `RECONCILE_BY_CLOID` | a timeout that is not safe to resend | `reconcile_by_cloid`; **never** resubmit |
| `READ_ORDER_STATUS` | accepted with no `response` envelope at all | Read `order_status` once. Only an API older than the release that reports outcomes inline answers this way |

```python
import time

from native_core import (
    next_action, USE_RESPONSE_OUTCOME, ORDER_CLOSED_UNFILLED,
    FIX_AND_RESUBMIT, BACKOFF_AND_RETRY, RECONCILE_BY_CLOID,
    order_oid, leaf_error_code, error_code, retry_after_ms,
)

verdict = next_action(resp)
if verdict == USE_RESPONSE_OUTCOME:
    oid = order_oid(resp)                              # done — no read needed
elif verdict == ORDER_CLOSED_UNFILLED:
    ...                                                # nothing rested, nothing wrong
elif verdict == FIX_AND_RESUBMIT:
    print(leaf_error_code(resp) or error_code(resp))   # fix, then send a NEW cloid
elif verdict == BACKOFF_AND_RETRY:
    time.sleep((retry_after_ms(resp) or 1000) / 1000)  # resend the SAME cloid
elif verdict == RECONCILE_BY_CLOID:
    exchange.info.reconcile_by_cloid(exchange.effective_account, "ETH/USDT", resp["cloid"])
```

Three more helpers worth knowing:

* `order_state` / `is_terminal` / `is_undetermined` / `is_filled` / `filled_quantity` read an order-status snapshot.
* `as_problem_details(failure)` flattens any failing trade body into one envelope. A benign cancel returns `None`.
* `retry_on_rate_limit(action)` wraps a write to resend only on `RateLimited`.

The SDK raises for five things: a transport failure (`NetworkError`), an uncertain write (`SubmissionUncertain`), a refused subscription (`SubscriptionError`), a non-trade body (`ClientError` / `ServerError`), and a problem caught locally (`LocalValidationError`).

Full response shapes and the `error.code` catalog: [Transaction Signing](../transaction-signing.md).

## Markets by symbol or id

Anywhere a `market` argument appears, pass a `"BASE/QUOTE"` symbol (`"ETH/USDT"`) or its integer market id — they are interchangeable. Symbols and precision are fetched once when `Info` is constructed.

```python
market_id = info.resolve_market_id("ETH/USDT")    # -> int, e.g. 2
markets = info.markets()["markets"]               # list; each market carries price_decimals, base_quantity_decimals, max_price_sig_figs
```

`info.resolve_market_id(market)` resolves a symbol (or passes an id through); an unknown symbol raises `LocalValidationError`, not a bare `KeyError`. Per-market precision comes from `info.markets()`; `snap_price` / `min_order_size` apply it for you.

## Agent vs owner mode

The SDK trades with an **API wallet** — a protocol-level agent key scoped only to placing and cancelling orders; it can never move funds. `Exchange.from_bundle(bundle)` builds an **agent-mode** `Exchange`: the bundle's `agentPrivateKey` signs, and its `accountAddress` is the **owner** the orders act on. The owner address is used only locally — it never goes on the wire; the API recovers the signer from the signature.

The **agent epoch** is resolved live at construction from `userAgents` (not taken from the bundle, which may be stale), and refreshed once automatically if a write is rejected with `AgentEpochMismatch`. Check approval any time without side effects:

```python
print(exchange.agent_info())                      # {"approved": True, "slot_id": 0, "epoch": 10}
print(exchange.agent_address)                      # the signing (agent) wallet
print(exchange.effective_account)                  # the account orders act on: the owner in agent mode
```

If the API wallet is not an approved agent on that owner, the constructor raises `LocalValidationError`; `agent_info()` reports it as data instead of throwing. Deposits, withdrawals, and `approveAgent` / `revokeAgent` are signed by your **main wallet in the web app**, never in the SDK.

## Networks

The SDK runs on both **mainnet** (`https://api.native.org`) and **testnet** (`https://api-test.native.org`). The bundle's `network` field selects which, and `from_bundle` picks the endpoint from it. An API wallet is created and approved on that network's site and is bound to it — a key signed for one network is rejected on the other.

## Polling, streaming & freshness

`Info` reads are poll-only over [`POST /info`](../post-info.md), and the read budget is the binding constraint: **one read per second per client IP** ([rate limits](../api-access.md#rate-limits-errors)). The unit is the IP, not the address — two bots on one host, or two behind one office NAT, share the single budget however many addresses they sign with.

That budget will not serve a tight quoting loop, so do not try to poll your way to one. Stream instead: `WsClient` pushes `l2Book`, `bbo`, `trades`, `orderUpdates` and `userFills` live over a single connection, and `Info.ws()` / `Exchange.ws()` build one that reuses the market table already loaded. Keep the HTTP poll for reconciliation, at around one read per second.

`wait_for_open` / `wait_for_order` poll internally at 50/100/200/400/800/1600/3200ms against a ~5s default deadline, so budget each call as several reads rather than one.

Every read view carries `query_height` and `app_hash` — the committed height the answer reflects. `queryStatus` exposes the retained recent-height window (`oldest_available_height`, `latest_available_height`, `recent_query_window_blocks`), which bounds how far back a windowed read such as `userFills` can reach. Treat precision and symbol metadata (`markets`, `assets`) as **slowly-changing** — fetch it once at startup (the SDK does this when `Info` is constructed) — and re-fetch balances, orders, and fills every loop.

{% hint style="warning" %}
**An `/info` read can fail while returning HTTP 200.** A windowed read that reaches outside the retained height window answers 200 with an `error` object *and* an empty result list, so code that goes straight for `["fills"]` reads a refusal as "nothing traded". Check `info_error(response)` before trusting an empty list. `iter_user_fills` and `recent_fills` already check it for you. `iter_user_fills` raises `ClientError` on a refused page; `recent_fills` passes `clamp=True`, so it re-aligns to the window instead of raising, and can skip the oldest blocks in the range.
{% endhint %}

## Cancel on disconnect

There is **no server-side scheduled-cancel** (dead-man switch) today, and dropping your WebSocket connection does not cancel anything either — a crashed or partitioned process leaves its resting orders on the book. Run the switch **client-side**: cancel every open order on shutdown, on any unhandled exception, and on a heartbeat timeout.

```python
try:
    run_strategy(exchange)
finally:
    exchange.cancel_open()   # dead-man switch: nothing left resting after exit
```

`exchange.cancel_open(...)` finds where the effective account actually rests (via `open_orders_all`, one read) and issues one `cancel_all` per market with orders; cancels are always admitted, even for a frozen account. Pass `markets=[...]` to bound the scan when you already know where you traded. [Examples](examples.md) has the pattern wired into a `finally:` shutdown pass with a heartbeat.

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
