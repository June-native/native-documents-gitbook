---
description: >-
  Every public method, helper, exception, and constant in the Native Core Python
  SDK, mapped to the POST /info and POST /trade calls it makes.
---

# API Reference

The public surface of `native-core-python-sdk` (import `native_core`), as of **2.0.0**: the `Info`, `Exchange` and `WsClient` classes, the response and problem helpers, the exception hierarchy, the transport controls, and the constants. Every method lists the underlying API call it makes so you can cross-reference the wire behaviour.

{% hint style="info" %}
This page is a symbol reference. For the field-level semantics of any returned JSON, follow the linked wire reference: reads resolve to [`POST /info`](../post-info.md), writes to [`POST /trade`](../post-trade.md). See also [Decimal units](../decimals-units.md) for raw/display conversion and [Transaction signing](../transaction-signing.md) for the signature the SDK builds for you.
{% endhint %}

Both classes return the API's JSON as plain, `TypedDict`-annotated dicts. Construct with `Info.from_bundle(bundle)` / `Exchange.from_bundle(bundle)` (a dict, JSON string, or file path), or by hand with `Info(base_url)` / `Exchange(wallet, base_url, owner=<account_address>)`. Anywhere a `market` argument appears, pass a `"BASE/QUOTE"` symbol (e.g. `"ETH/USDT"`) or its integer market id.

## Info (reads)

Wraps [`POST /info`](../post-info.md). Each call maps to one top-level `type` discriminator, except where noted.

| Method | Returns | `POST /info` type |
| --- | --- | --- |
| `markets()` | Tradable markets with precision (`price_decimals`, `max_price_sig_figs`, `base_quantity_decimals`) | `markets` |
| `assets()` | Assets and their `balance_decimals` / withdraw fees | `assets` |
| `quote_assets()` | Quote-asset allowlist with per-market minimum notional | `quoteAssets` |
| `l2_book(market, depth=20)` | Order book, up to 100 levels. **`bids` and `asks` are absent entirely when `found` is false** — check `found` before indexing | `l2Book` |
| `mark_prices(asset_ids=None)` | Mark prices in `usd_atoms`; omit `asset_ids` for all | `markPrices` |
| `oracle_status()` | Oracle health (`available` / `unavailable` + reason) | `oracleStatus` |
| `query_status()` | Current query height and the retained recent-height window | `queryStatus` |
| `user_balances(address)` | Spot balances (`available`, `locked`) per asset | `userBalances` |
| `open_orders(address, market)` | Resting orders in one market | `openOrders` |
| `open_orders_all(address, markets=None, *, per_market=False)` | Open orders across markets, each tagged with its `market_id`. **One** read by default; logs a warning if the API truncated the list | `openOrders` (one call, `market_id: -1`) |
| `order_status(oid=None, user=None, market=None, cloid=None)` | One order, by `oid` or by `(user, market, cloid)` | `orderStatus` |
| `batch_order_status(orders)` | Up to **20** order selectors in one read, results in request order. The cap is checked locally, before a request is spent | `batchOrderStatus` |
| `user_fills(address, from_height, to_height, limit)` | Fills in a raw block-height window (≤10,000 blocks) | `userFills` |
| `iter_user_fills(address, start_height=None, ...)` | Iterator over all fills since a height, paged and deduped | `userFills` (paged) |
| `recent_fills(address, blocks=10000)` | Every fill in roughly the last N blocks, window resolved for you | `queryStatus` + `userFills` |
| `account_status(address)` | Whether an account exists and its freeze state (`active` / `frozen`) | `accountStatus` |
| `spot_credit_account(address)` | Credit-account authorization, freeze status, and USD valuations | `spotCreditAccount` |
| `spot_credit_positions(address)` | Credit-account positions per asset | `spotCreditPositions` |
| `credit_trading_allowed(account, market)` | Whether a `spot_credit_account` response may trade `market` on credit | local — no request |
| `user_agents(address)` | Active API-wallet (agent) slots for an owner | `userAgents` |
| `agent_status(owner, agent_address)` | Whether a wallet is an approved agent: `{approved, slot_id, epoch}` | `userAgents` (derived) |
| `ws(**kwargs)` | A [`WsClient`](#wsclient-streaming) reusing this `Info`'s market table, so subscribing by symbol costs no extra request | local — no request |

{% hint style="info" %}
`credit_trading_allowed` combines the two halves the API publishes separately, and it exists because doing that by hand has a trap: `credit_trading_whitelisted_market_ids` holds market ids as **strings** while every other market id in the API is an integer, so `market_id in whitelist` is always `False` and never raises.
{% endhint %}

### Local and polling helpers

Client-side conveniences on `Info`. The first three read only the market metadata fetched once at construction and send no request; the rest poll the API.

| Method | Returns | Sends |
| --- | --- | --- |
| `resolve_market_id(market)` | Integer market id for a `"BASE/QUOTE"` symbol | local — no request (cached metadata) |
| `snap_price(market, price, rounding=ROUND_DOWN)` | `price` rounded to the market's precision, as a wire-ready string | local — no request (cached metadata) |
| `min_order_size(market, price, margin="1.1")` | Smallest size at `price` clearing the quote minimum notional | local — no request (cached metadata) |
| `protection_price(market, is_buy, slippage_bps, ref_price=None)` | Worst acceptable price for a market order, derived from the book. Raises `LocalValidationError` when no book is published or the crossed side is empty | polls `l2Book` (skipped when `ref_price` is given) |
| `wait_for_open(user, market, cloid, timeout=5.0)` | Poll until the order is resting (`open`) or terminal | polls `orderStatus` |
| `wait_for_order(user, market, cloid, timeout=5.0)` | Poll until the order is terminal (`filled` / `cancelled` / …) | polls `orderStatus` |
| `reconcile_by_cloid(user, market, cloid, timeout=5.0)` | One-call verdict `{state, undetermined, is_filled, filled_qty, status}` for the uncertain/timeout recovery path | polls `orderStatus` |

{% hint style="info" %}
Call `wait_for_open` for an order you expect to rest (`gtc` / `alo`) and `wait_for_order` for one you expect to finish (`ioc` / `fok` / `market`, or after a cancel). Calling `wait_for_order` on a resting order just times out — it has no terminal state. `exchange.place(...)` picks the right wait for you.
{% endhint %}

## Exchange (writes)

Wraps [`POST /trade`](../post-trade.md); one signed action per call. `Exchange` owns an internal `Info` as `exchange.info`, so reads and writes share one client. Pass `sz` / `limit_px` / `protection_px` as `str` or `Decimal` — **never `float`** (the SDK validates before signing and raises `LocalValidationError` rather than round silently).

| Method | What it does | `POST /trade` action |
| --- | --- | --- |
| `order(market, is_buy, sz, limit_px, tif, cloid=None)` | Places one limit order (`tif` is `gtc` / `ioc` / `fok` / `alo`). Echoes `cloid` and `nonce` | `order` |
| `market_order(market, is_buy, sz, protection_px=None, tif="ioc", cloid=None, *, slippage_bps=None)` | Places a market order (`ioc` / `fok`). Pass `protection_px` (worst acceptable price) **or** `slippage_bps` to derive it from the book | `order` (`order_type: "market"`) |
| `place(market, is_buy, sz, limit_px, tif, cloid=None, *, confirm=True, timeout=5.0)` | Submits a limit order, then reads the matching `orderStatus` snapshot. Returns `{cloid, submission, status, state, oid}`; `status` and `state` are `None` when the read was skipped | `order` + `orderStatus` |
| `build_order(market, is_buy, sz, limit_px, tif, cloid=None, order_type="limit")` | **Dry run** — runs the same local validation and returns `{action, cloid}` without signing or sending. Nothing leaves the process; no nonce consumed | none (sends nothing) |
| `cancel(market, oid)` | Cancels one order by server order id | `cancel` |
| `cancel_by_cloid(market, cloid)` | Cancels one order by client order id | `cancel` |
| `cancel_all(market)` | Cancels every open order for the effective owner in one market | `cancelAll` |
| `cancel_open(markets=None)` | Cancels every open order across markets, only where orders actually rest; returns `{market_id: TradeResponse}` | `openOrders` scan + `cancelAll` per market |
| `modify(market, oid_or_cloid, replacement)` | Atomically cancels the target and places `replacement` in one action | `modify` |
| `batch(items)` | Up to 10 mixed `order` / `cancel` / `cancelAll` / `modify` items under one nonce; echoes `cloids`. Per-leg outcomes come from `batch_legs(response)` | `batch` |
| `set_expires_after(expires_after_ms)` | Sets the instance-level expiry threaded into every signed action (`None` to omit) | local — no request |
| `sign_action(action)` | Signs an action into a ready-to-send `/trade` body **without** sending it, for submitting over [`WsClient.post_action`](#wsclient-streaming). Consumes a nonce, so the body is single-use | local — no request |
| `agent_info()` | Approval status of this wallet on the owner: `{approved, slot_id, epoch}`. Side-effect-free health check | reads `userAgents` |
| `ws(**kwargs)` | A [`WsClient`](#wsclient-streaming) on the same endpoint, reusing this `Exchange`'s market table | local — no request |
| `Exchange.random_cloid()` | A fresh random client order id (`0x` + 16 bytes). `@staticmethod` | local — no request |

Also on `Exchange`: the `agent_address` property (the signing wallet), and `effective_account` (the account orders act on — the owner in agent mode, otherwise the wallet address; pass it as the `user` to every `Info` order-status read).

Each write returns the raw API response with the client handle echoed in: `submission_status`, `tx_hash`, `error`, `cloid` (or `cloids` for a batch), `nonce`, and — on an accepted write — **`response`**, the outcome envelope carrying the assigned `oid`, the fill total and average price, the cancelled order ids, and one sub-envelope per batch leg.

{% hint style="danger" %}
**`accepted` is a verdict on the transaction, not on your order.** Only six envelope-level problems and the API's admission refusals come back `rejected`. A per-action failure — `tick`, `lotsize`, `badalopx`, `insufficientspotbalance`, `mintradespotntl`, `missingorder` — arrives as `accepted`, with the reason on a leaf of the `response` envelope. Check `is_order_failed(resp)` (or branch on `next_action`) before treating the order as live.

Most failures leave no `/info` record, so the leaf is the only trace — see [what a failed order leaves behind](core-concepts.md#what-a-failed-order-leaves-behind).

Take the `oid` and the fill from `order_oid(resp)` / `fill(resp)` at no read cost.

On a wire timeout the SDK raises `SubmissionUncertain` (or returns `submission_status: "timeout"`): reconcile by `cloid` and **never resubmit**, unless `is_safe_to_resend(resp)` is true. See [Core concepts](core-concepts.md#accepted-is-not-placed).
{% endhint %}

A minimal place-then-cancel loop:

```python
from decimal import Decimal

from native_core import Exchange, is_accepted, is_order_failed, leaf_error_code, order_state

exchange = Exchange.from_bundle("bundle.json")   # from your bundle: picks the endpoint, loads the key
info = exchange.info
MARKET = "ETH/USDT"

# Price a resting bid, size it to the market minimum, submit, and wait for it to rest.
book = info.l2_book(MARKET, depth=1)
if not book.get("found"):
    raise SystemExit("unknown market")           # found=false means the id is unknown
levels = book.get("asks") or book.get("bids") or []
if not levels:
    raise SystemExit("no resting orders on this market yet")
px = info.snap_price(MARKET, Decimal(levels[0]["price"]) / 2)
sz = info.min_order_size(MARKET, px)
order = exchange.place(MARKET, is_buy=True, sz=sz, limit_px=px, tif="gtc")
if is_order_failed(order["submission"]):
    raise SystemExit(f"order failed: {leaf_error_code(order['submission'])}")
print(order["cloid"], order["oid"], order["state"])   # 0x…  1234  open

# Cancel it, then confirm it left the book. A cancel against an order that is already
# gone is accepted too, with a "missingorder" leaf — so check the order, not the status.
cancel = exchange.cancel_by_cloid(MARKET, order["cloid"])
assert is_accepted(cancel) and not is_order_failed(cancel)
final = info.wait_for_order(exchange.effective_account, MARKET, order["cloid"])
print(order_state(final))                         # cancelled
```

## WsClient (streaming)

Wraps the [WebSocket](../websocket.md): nine push feeds on one connection, delivered on a background thread. Synchronous like the rest of the SDK — you never write `async`. Build one with `Info.ws()` / `Exchange.ws()` to reuse a loaded market table, or `WsClient(base_url)` directly. Call `connect()` before subscribing.

Every `subscribe_*` takes an optional `callback`. Pass one and frames go to it; leave it out and they go to the shared queue you read with `stream()`.

| Method | Feed | Topic |
| --- | --- | --- |
| `subscribe_trades(market, callback=None)` | every trade printed, as it happens | one market |
| `subscribe_l2_book(market, callback=None)` | full book snapshot, each message replacing the last | one market |
| `subscribe_bbo(market, callback=None)` | best bid / best offer, pushed only when either changes | one market |
| `subscribe_all_mids(callback=None)` | mid price for every two-sided market, as one table | all markets |
| `subscribe_user_fills(address, callback=None)` | fills from the account's own point of view. The first message replays the most recent 100 | one address |
| `subscribe_order_updates(address, callback=None)` | order lifecycle, as status words: `open`, `filled`, `canceled`, `selfTradeCanceled`, and the `<reason>Rejected` family (`badAloPxRejected`, `iocCancelRejected`, `fokCancelRejected`, `marketOrderNoLiquidityRejected`) | one address |
| `subscribe_open_orders(address, callback=None)` | every resting order, as a complete replacement | one address |
| `subscribe_spot_state(address, callback=None)` | spot balances, as a complete replacement | one address |
| `subscribe_spot_credit_state(address, callback=None)` | credit positions and credit line | one address |
| `subscribe(subscription, callback=None)` | raw subscription body — the escape hatch for a feed newer than the SDK | — |

Each returns a `Subscription`; pass it back to `unsubscribe(subscription)`. Both block until the server answers and raise `SubscriptionError` on a refusal.

| Member | Meaning |
| --- | --- |
| `connect()` | Opens the connection and returns `self`, so `WsClient(url).connect()` chains |
| `close()` | Closes it and ends any `stream()` |
| `connected` | Property: whether the socket is up |
| `subscriptions` | Property: the live `Subscription` list |
| `stream(timeout=None)` | Iterator over frames from every callback-less subscription, each the full `{"channel", "data"}` envelope. `timeout` ends it after that many idle seconds |
| `dropped_messages` | Count of frames discarded because the `stream()` buffer overflowed |
| `post_info(payload)` | Runs an `/info` read over this connection. Same answer and same exceptions as HTTP |
| `post_action(signed_body)` | Submits an already-signed `/trade` body. Build it with `Exchange.build_order` then `Exchange.sign_action` |

Constructor arguments worth setting:

| Argument | Default | Meaning |
| --- | --- | --- |
| `max_subscriptions` | `10` | **Local** refusal threshold mirroring the server's. Raising it buys no extra allowance |
| `max_inflight_posts` | `1` | As above |
| `ping_interval` | `20` s | Keeps the connection alive against the server's 60-second idle close |
| `reconnect` | on | Restores subscriptions by itself |
| `on_reconnect` | `None` | Fires once they are back. Re-read anything that gapped here |
| `on_error` | `None` | Server errors belonging to no call of yours |

{% hint style="warning" %}
**A write over the socket reports less than the same write over HTTP.** When the API answers with anything other than 2xx it discards the trade response, so there is no `submission_status`, no `tx_hash` and no envelope left to read. `post_action` raises instead: `ClientError` for a refusal that cannot have executed (bad signature or nonce, or a 429), and `SubmissionUncertain` for anything that might still land. Prefer `Exchange.order` for submitting. A write here is never retried automatically, not even on a rate limit.
{% endhint %}

{% hint style="info" %}
**The socket and HTTP use different field names for the same thing, and one pair is a trap.** `Info.user_balances` gives `available` / `locked`; `spotState` gives `total` / `hold`, where `total` is available **plus** locked. Reading `total` where you used to read `available` overstates your free balance whenever an order is resting. Book levels differ too: `price` / `quantity` / `order_count` over HTTP, `px` / `sz` / `n` over the socket. See [WebSocket](../websocket.md) for the full mapping.
{% endhint %}

The feed payloads are exported as `TypedDict`s — `WsBook`, `WsTrade`, `WsBbo`, `WsAllMids`, `WsUserFills`, `WsFill`, `WsOrder`, `WsOrderInner`, `WsOpenOrders`, `WsSpotState`, `WsSpotBalance`, `WsSpotCreditState`, `WsLevel` — so streaming code type-checks under `mypy --strict` the way the HTTP surface already does.

For a runnable walkthrough see [Streaming](streaming.md).

## Response and problem helpers

Re-exported at the top level (`from native_core import ...`). A business rejection is **data on the response body**, not an exception — these classify it. They come in three families, and a correct integration uses all three.

### The transaction

Read the top level of a `/trade` response. None of these can tell you whether your **order** worked.

| Helper | Returns |
| --- | --- |
| `is_accepted(response)` | `True` when `submission_status` is `accepted`, meaning the transaction landed and executed. **Pair it with `is_order_failed`** |
| `is_rejected(response)` | `True` when `submission_status` is `rejected` |
| `is_timeout(response)` | `True` when `submission_status` is `timeout` |
| `error_code(response)` | The rejection `error.code` string (CamelCase), or `None` |
| `retry_after_ms(response)` | Back-off hint from `error.retry_after_ms`, or `None` |
| `is_retryable(response)` | `True` only for `RateLimited` (never admitted, so safe to resend) |
| `is_safe_to_resend(response)` | `True` for the `HandoffTimeout` / `HandoffMultipleActive` / `HandoffBufferFull:*` timeouts, which prove the transaction never reached a node |
| `next_action(response)` | One verdict string to branch on, folding all three families together (`None` for a non-trade response) |

### The order, inside the `response` envelope

Present on an accepted write. This is the only place a per-action failure is reported **on the write response**, and for most codes the only place it is reported at all.

| Helper | Returns |
| --- | --- |
| `is_order_failed(response)` | `True` when a leaf carries a genuine failure. Benign cancels excluded, so an unfilled IOC is not a failure |
| `leaf_error_code(response)` | The first failing leaf's code, **lowercase** (`tick`, `lotsize`, `insufficientspotbalance`, `ioccancel`, …), or `None` |
| `leaf_errors(response)` | Every failing leaf's code in request order, benign cancels included |
| `is_benign_cancel(code)` | `True` for `ioccancel`, `fokcancel`, `selftradepreventioncancel`, `marketordernoliquidity`. **Takes a code string, not a response** |
| `order_oid(response)` | The assigned order id off the write itself, or `None` |
| `fill(response)` | The first `filled` leaf — `{total_sz, avg_px, oid}` as display strings — or `None` when nothing filled |
| `batch_legs(response)` | One sub-envelope per batch leg, in request order, so leg *N*'s outcome is `legs[N]["status"]`. Empty for a non-batch |
| `trade_envelope(response)` | The raw `response` envelope, or `None` when the write was not accepted |
| `trade_outcomes(response)` | Every outcome leaf flattened in request order. Not leg-indexed — use `batch_legs` for that |

### The order-status snapshot, and the rest

| Helper | Returns | Reads |
| --- | --- | --- |
| `order_state(response)` | The order's status string (e.g. `open`, `filled`, `cancelled`, `unknown`) | order-status response |
| `is_terminal(status)` | `True` when a status string is terminal | status string |
| `is_undetermined(status)` | `True` when a status string is not yet resolved | status string |
| `is_filled(response)` | `True` when the order is fully filled | order-status response |
| `filled_quantity(response)` | Filled base quantity as a string (`"0"` when none) | order-status response |
| `info_error(response)` | The `error` object on an `/info` response, or `None`. **A refused read answers HTTP 200 with an error and an empty list** | any `/info` response |
| `as_problem_details(failure)` | Renders any failing trade body — rejected, timed out, or accepted with a failed order — into one flat `{type, title, retryable, next_action, cloids, ...}` envelope. `None` for a success or a benign cancel | exception or trade response |
| `retry_on_rate_limit(action, ...)` | Wraps a write callable to resend **only** on `RateLimited`, honouring `retry_after_ms` | callable |
| `random_cloid()` | A fresh client order id (`0x` + 16 bytes) | — |

`next_action` collapses any trade response into the one branch an agent takes:

| `next_action` | Situation | What to do |
| --- | --- | --- |
| `USE_RESPONSE_OUTCOME` | accepted, the order worked | Nothing more. The `oid` and the fill are already on the response |
| `ORDER_CLOSED_UNFILLED` | accepted, benign cancel | Nothing more. The order is over and nothing filled |
| `FIX_AND_RESUBMIT` | accepted but the order failed, or a rejection other than `RateLimited` | Fix the input or the account state. There is nothing to reconcile; what you send next is a fresh order |
| `BACKOFF_AND_RETRY` | `RateLimited`, or a `Handoff*` timeout — never reached a node | Sleep `retry_after_ms`, then resend the **same** `cloid` |
| `RECONCILE_BY_CLOID` | a timeout that is not safe to resend | `reconcile_by_cloid`; **never** resubmit under a fresh nonce |
| `READ_ORDER_STATUS` | accepted with no `response` envelope at all | Read `order_status` once. Only an API older than the release that reports outcomes inline answers this way |

## Exceptions

Everything subclasses `native_core.Error`. Business rejections are **not** exceptions — they arrive in the response body (above). The SDK raises only for transport failures, non-trade bodies, and problems caught before signing.

| Exception | Raised when |
| --- | --- |
| `Error` | Base class for every SDK exception |
| `LocalValidationError` | A problem caught locally: bad precision, below the minimum notional, an unknown `tif` or market, an over-long batch, no book to derive a `protection_price` from, or (at construction) an API wallet that is not an approved agent |
| `NetworkError` | A transport failure (timeout, connection, DNS) **before any response arrived** |
| `SubmissionUncertain` | A signed write whose outcome cannot be read: a transport failure over HTTP, or a non-2xx answer to `WsClient.post_action`. The most important one. Carries `.cloids` (`.cloid` for a single order) and `.nonce`; reconcile with `reconcile_by_cloid` and never resubmit |
| `SubscriptionError` | A WebSocket subscribe or unsubscribe was refused or went unanswered. Carries `.reason` and `.subscription` |
| `ClientError` | A body that is **not** a trade response and reports a client-side problem. Usually an HTTP 4xx, but also an **HTTP 200** whose body carries an `/info` `error` (a read outside the retained window). Has `status_code`, `error_code`, `error_message` |
| `ServerError` | An HTTP 5xx whose body is **not** a trade response. Has `status_code` |

{% hint style="warning" %}
`SubmissionUncertain` is the one place not to reach for `wait_for_order`: an order that rests has no terminal state, so that wait can only time out. Use `reconcile_by_cloid`, which waits for either.
{% endhint %}

`ErrorCode` is a convenience enum of known rejection codes: `RateLimited`, `ExpiredTx`, `WrongChainId`, `DirectSignerIsActiveAgent`, `AgentEpochMismatch`, `InsufficientSpotBalance`.

It covers only the **CamelCase** codes on a rejected body's `error.code`. Two families are deliberately not members:

* execution-stage failures, which are lowercase and live on the `response` leaf — read them with `leaf_error_code`
* the `Handoff*` timeout codes, matched by `is_safe_to_resend` rather than by equality

So `error_code(resp) == ErrorCode.INSUFFICIENT_SPOT_BALANCE` misses the lowercase `insufficientspotbalance` the same condition produces at execution.

## Transport controls

`Info` and `Exchange` (and both `from_bundle` constructors) accept the same transport keyword arguments. On `Exchange` the trace controls apply to both writes and the internal `Info`'s reads. `WsClient` shares only `rate_limit_retries`; its own connection settings are listed under [WsClient](#wsclient-streaming).

| Argument | Default | Meaning |
| --- | --- | --- |
| `timeout` | `30` | Per-request deadline in seconds. Pass `None` for no deadline |
| `pool_maxsize` | `100` | Connection-pool size |
| `rate_limit_retries` | `3` | How many times to retry an HTTP 429 internally, backing off by the server's hint (capped at 2000 ms). Pass `0` to surface the 429 at once |
| `on_request` | `None` | `on_request(url_path, body, trace_id)` — called before each request; `trace_id` is the id being sent, or `None` |
| `on_response` | `None` | `on_response(url_path, status, body, elapsed_ms, trace_id)` — called after each response; `trace_id` is the `x-trace-id` the API echoed, or `None` |
| `trace_id_factory` | `None` | Called once per request for the `x-trace-id` to send; omit to send no header. You supply the id — the SDK never generates one |

```python
import uuid

from native_core import Exchange

exchange = Exchange.from_bundle(
    "bundle.json",
    timeout=10,
    trace_id_factory=lambda: str(uuid.uuid4()),
    on_response=lambda path, status, body, ms, trace_id: print(path, status, ms, trace_id),
)
```

To capture only the id the API itself generates, send none and read the `trace_id` argument of `on_response`. Because a 429 is retried internally, an unexplained latency spike on one call can be a hidden retry — `on_response` fires once per attempt, so the trace ids tell them apart.

## Constants

Exposed under `native_core.constants`.

| Constant | Value | Notes |
| --- | --- | --- |
| `MAINNET_API_URL` | `https://api.native.org` | Mainnet base URL |
| `TESTNET_API_URL` | `https://api-test.native.org` | Testnet base URL |
| `MAINNET_CHAIN_ID` | `696969` | Signed into every action; a key signed for one network is rejected on the other with `WrongChainId` |
| `TESTNET_CHAIN_ID` | `969696` | As above |
| `NETWORK_URLS` | network name → endpoint URL | The bundle's `network` field maps through this |
| `resolve_chain_id(base_url)` | chain id, or `None` | `None` for a URL that is not one of the two public endpoints |
| `normalize_base_url(base_url)` | the URL without a trailing slash | Makes the chain-id lookup insensitive to it |

A self-hosted deployment resolves to no chain id, so pass `chain_id=` to `Exchange` explicitly — construction raises `LocalValidationError` otherwise.

## See also

* [Getting started](getting-started.md) — create an API wallet and place your first order.
* [Core concepts](core-concepts.md) — accepted-vs-placed, the reconcile-by-cloid contract, and one `Exchange` per key.
* [Streaming](streaming.md) — the `WsClient` walkthrough.
* [AI agents and MCP](ai-agents-and-mcp.md) — driving the SDK from an agent, and the bundled `native-core-mcp` server.
* [Troubleshooting](troubleshooting.md) — what each error means and how to fix it.
* [`POST /info`](../post-info.md), [`POST /trade`](../post-trade.md) and [WebSocket](../websocket.md) — the wire reference for the endpoints these classes wrap.
* [Decimal units](../decimals-units.md) and [Transaction signing](../transaction-signing.md) — raw/display conversion and the signature the SDK builds.
