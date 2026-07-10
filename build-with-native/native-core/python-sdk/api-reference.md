---
description: >-
  Every public method, helper, exception, and constant in the Native Core Python
  SDK, mapped to the POST /info and POST /trade calls it makes.
---

# API Reference

The complete surface of `native-core-python-sdk` (import `native_core`): the `Info` and `Exchange` classes, the response and problem helpers, the exception hierarchy, the transport controls, and the constants. Every method lists the underlying gateway call it makes so you can cross-reference the wire behaviour.

{% hint style="warning" %}
**Testnet only · pre-1.0.** The Native Core Python SDK is `v0.1.0` (alpha) and connects to **testnet only** — mainnet is not yet enabled. The API may change before 1.0; pin an exact version: `pip install native-core-python-sdk==0.1.0`.
{% endhint %}

{% hint style="info" %}
This page is a symbol reference. For the field-level semantics of any returned JSON, follow the linked wire reference: reads resolve to [`POST /info`](../post-info.md), writes to [`POST /trade`](../post-trade.md). See also [Decimal units](../decimals-units.md) for raw/display conversion and [Transaction signing](../transaction-signing.md) for the signature the SDK builds for you.
{% endhint %}

Both classes return the gateway's JSON as plain, `TypedDict`-annotated dicts. Construct with `Info.from_bundle(bundle)` / `Exchange.from_bundle(bundle)` (a dict, JSON string, or file path), or by hand with `Info(base_url)` / `Exchange(wallet, base_url, owner=<account_address>)`. Anywhere a `market` argument appears, pass a `"BASE/QUOTE"` symbol (e.g. `"ETH/USDT"`) or its integer market id.

## Info (reads)

Wraps [`POST /info`](../post-info.md). Each call maps to one top-level `type` discriminator, except where noted.

| Method | Returns | `POST /info` type |
| --- | --- | --- |
| `markets()` | Tradable markets with precision (`price_decimals`, `max_price_sig_figs`, `base_quantity_decimals`) | `markets` |
| `assets()` | Assets and their `balance_decimals` / withdraw fees | `assets` |
| `quote_assets()` | Quote-asset allowlist with per-market minimum notional | `quoteAssets` |
| `l2_book(market, depth=20)` | Order book, up to 100 levels (`bids`, `asks`) | `l2Book` |
| `mark_prices(asset_ids=None)` | Mark prices in `usd_atoms`; omit `asset_ids` for all | `markPrices` |
| `oracle_status()` | Oracle health (`available` / `unavailable` + reason) | `oracleStatus` |
| `query_status()` | Current query height and the retained recent-height window | `queryStatus` |
| `user_balances(address)` | Spot balances (`available`, `locked`) per asset | `userBalances` |
| `open_orders(address, market)` | Resting orders in one market | `openOrders` |
| `open_orders_all(address, markets=None)` | Open orders across markets, each tagged with its `market_id` | `openOrders` (one per market) |
| `order_status(oid=None, user=None, market=None, cloid=None)` | One order, by `oid` or by `(user, market, cloid)` | `orderStatus` |
| `batch_order_status(orders)` | Up to 20 order lookups in one request | `batchOrderStatus` |
| `user_fills(address, from_height, to_height, limit)` | Fills in a raw block-height window (≤10,000 blocks) | `userFills` |
| `iter_user_fills(address, start_height=None, ...)` | Iterator over all fills since a height, paged and deduped | `userFills` (paged) |
| `recent_fills(address, blocks=10000)` | Every fill in roughly the last N blocks, window resolved for you | `queryStatus` + `userFills` |
| `tx_status(user, cloid)` | Status of a deposit / withdraw / settle / repay by cloid (not orders) | `txStatusByCloid` |
| `account_status(address)` | Whether an account exists and its freeze state (`active` / `frozen`) | `accountStatus` |
| `spot_credit_account(address)` | Credit-account authorization, freeze status, and USD valuations | `spotCreditAccount` |
| `spot_credit_positions(address)` | Credit-account positions per asset | `spotCreditPositions` |
| `user_agents(address)` | Active API-wallet (agent) slots for an owner | `userAgents` |
| `agent_status(owner, agent_address)` | Whether a wallet is an approved agent: `{approved, slot_id, epoch}` | `userAgents` (derived) |

### Local and polling helpers

Client-side conveniences on `Info`. The first three read only the market metadata fetched once at construction and send no request; the rest poll the gateway.

| Method | Returns | Sends |
| --- | --- | --- |
| `resolve_market_id(market)` | Integer market id for a `"BASE/QUOTE"` symbol | local — no request (cached metadata) |
| `snap_price(market, price, rounding=ROUND_DOWN)` | `price` rounded to the market's precision, as a wire-ready string | local — no request (cached metadata) |
| `min_order_size(market, price, margin="1.1")` | Smallest size at `price` clearing the quote minimum notional | local — no request (cached metadata) |
| `protection_price(market, is_buy, slippage_bps, ref_price=None)` | Worst acceptable price for a market order, derived from the book | polls `l2Book` (skipped when `ref_price` is given) |
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
| `place(market, is_buy, sz, limit_px, tif, cloid=None, *, confirm=True, timeout=5.0)` | Submits a limit order and waits for the outcome matching its `tif`; returns `{cloid, submission, status, state}` | `order` + polls `orderStatus` |
| `build_order(market, is_buy, sz, limit_px, tif, cloid=None, order_type="limit")` | **Dry run** — runs the same local validation and returns `{action, cloid}` without signing or sending. Nothing leaves the process; no nonce consumed | none (sends nothing) |
| `cancel(market, oid)` | Cancels one order by server order id | `cancel` |
| `cancel_by_cloid(market, cloid)` | Cancels one order by client order id | `cancel` |
| `cancel_all(market)` | Cancels every open order for the effective owner in one market | `cancelAll` |
| `cancel_open(markets=None)` | Cancels every open order across markets, only where orders actually rest; returns `{market_id: TradeResponse}` | `openOrders` scan + `cancelAll` per market |
| `modify(market, oid_or_cloid, replacement)` | Atomically cancels the target and places `replacement` in one action | `modify` |
| `batch(items)` | Up to 10 mixed `order` / `cancel` / `cancelAll` / `modify` items under one nonce; echoes `cloids` | `batch` |
| `set_expires_after(expires_after_ms)` | Sets the instance-level expiry threaded into every signed action (`None` to omit) | local — no request |
| `agent_info()` | Approval status of this wallet on the owner: `{approved, slot_id, epoch}`. Side-effect-free health check | reads `userAgents` |
| `Exchange.random_cloid()` | A fresh random client order id (`0x` + 16 bytes). `@staticmethod` | local — no request |

Also on `Exchange`: the `agent_address` property (the signing wallet), and `effective_account` (the account orders act on — the owner in agent mode, otherwise the wallet address; pass it as the `user` to every `Info` order-status read).

Each write returns the raw gateway response with the client handle echoed in: `submission_status`, `tx_hash`, `error`, `cloid` (or `cloids` for a batch), and `nonce`.

{% hint style="warning" %}
`submission_status: "accepted"` means **admitted to the pipeline, not filled**. Resolve the real outcome by `cloid` with `wait_for_open` / `reconcile_by_cloid` before treating an order as done. On a wire timeout the SDK raises `SubmissionUncertain` (or returns `submission_status: "timeout"`) — reconcile by `cloid` and **never resubmit under a fresh nonce**, or the order may land twice. See [Core concepts](core-concepts.md).
{% endhint %}

A minimal place-then-reconcile loop:

```python
from decimal import Decimal

from native_core import Exchange, is_accepted, order_state

exchange = Exchange.from_bundle("bundle.json")   # testnet bundle: picks the gateway, loads the key
info = exchange.info
MARKET = "ETH/USDT"

# Price a resting bid, size it to the market minimum, submit, and wait for it to rest.
book = info.l2_book(MARKET, depth=1)
px = info.snap_price(MARKET, Decimal(book["asks"][0]["price"]) / 2)
sz = info.min_order_size(MARKET, px)
order = exchange.place(MARKET, is_buy=True, sz=sz, limit_px=px, tif="gtc")
print(order["cloid"], order["state"])            # 0x…  open

# Cancel it, then confirm it left the book.
cancel = exchange.cancel_by_cloid(MARKET, order["cloid"])
assert is_accepted(cancel)
final = info.wait_for_order(exchange.effective_account, MARKET, order["cloid"])
print(order_state(final))                         # cancelled
```

## Response and problem helpers

Re-exported at the top level (`from native_core import ...`). A business rejection is **data on the response body**, not an exception — these classify it. `is_*` / `error_code` / `retry_after_ms` / `next_action` read a trade response (from a write); `order_state` / `is_terminal` / `is_undetermined` / `is_filled` / `filled_quantity` read an order-status response (from a read).

| Helper | Returns | Reads |
| --- | --- | --- |
| `is_accepted(response)` | `True` when `submission_status` is `accepted` | trade response |
| `is_rejected(response)` | `True` when `submission_status` is `rejected` | trade response |
| `is_timeout(response)` | `True` when `submission_status` is `timeout` | trade response |
| `error_code(response)` | The rejection `error.code` string, or `None` | trade response |
| `retry_after_ms(response)` | Back-off hint from `error.retry_after_ms`, or `None` | trade response |
| `is_retryable(response)` | `True` only for `RateLimited` (never admitted, so safe to resend) | trade response |
| `next_action(response)` | One verdict: `POLL_ORDER_STATUS`, `BACKOFF_AND_RETRY`, `RECONCILE_BY_CLOID`, or `FIX_AND_RESUBMIT` (`None` for a non-trade response) | trade response |
| `order_state(response)` | The order's status string (e.g. `open`, `filled`, `cancelled`, `unknown`) | order-status response |
| `is_terminal(status)` | `True` when a status string is terminal | status string |
| `is_undetermined(status)` | `True` when a status string is not yet resolved | status string |
| `is_filled(response)` | `True` when the order is fully filled | order-status response |
| `filled_quantity(response)` | Filled base quantity as a string (`"0"` when none) | order-status response |
| `as_problem_details(failure)` | Renders any exception or rejected/timeout body into one flat `{type, title, retryable, next_action, cloids, ...}` envelope | exception or trade response |
| `retry_on_rate_limit(action, ...)` | Wraps a write callable to resend **only** on `RateLimited`, honouring `retry_after_ms` | callable |
| `random_cloid()` | A fresh client order id (`0x` + 16 bytes) | — |

`next_action` collapses any trade response into the one branch an agent takes:

| `next_action` | Situation | What to do |
| --- | --- | --- |
| `POLL_ORDER_STATUS` | accepted — in the pipeline | Poll `wait_for_open` / `wait_for_order` / `reconcile_by_cloid` for the settled state |
| `RECONCILE_BY_CLOID` | timeout — indeterminate | `reconcile_by_cloid`; **never** resubmit under a fresh nonce |
| `BACKOFF_AND_RETRY` | `RateLimited` — never admitted | Sleep `retry_after_ms`, then resend the same order |
| `FIX_AND_RESUBMIT` | other rejection | Fix the input or account state, then submit fresh |

## Exceptions

Everything subclasses `native_core.Error`. Business rejections are **not** exceptions — they arrive in the response body (above). The SDK raises only for transport failures, non-trade bodies, and problems caught before signing.

| Exception | Raised when |
| --- | --- |
| `Error` | Base class for every SDK exception |
| `LocalValidationError` | Before signing: bad precision, below the minimum notional, an unknown `tif` or market, a mainnet gateway, or (at construction) an API wallet that is not an approved agent |
| `NetworkError` | A transport failure (timeout, connection, DNS) **before any response arrived** |
| `SubmissionUncertain` | A write was signed and sent, then timed out — the most important one. Carries `.cloid` and `.nonce`; reconcile with `reconcile_by_cloid` / `wait_for_order` and never resubmit |
| `ClientError` | An HTTP 4xx whose body is **not** a trade response. Has `status_code`, `error_code`, `error_message` |
| `ServerError` | An HTTP 5xx whose body is **not** a trade response. Has `status_code` |

`ErrorCode` is a convenience enum of known rejection codes: `RateLimited`, `ExpiredTx`, `WrongChainId`, `DirectSignerIsActiveAgent`, `AgentEpochMismatch`, `InsufficientSpotBalance`.

## Transport controls

`Info` and `Exchange` (and both `from_bundle` constructors) accept the same transport keyword arguments. On `Exchange` the trace controls apply to both writes and the internal `Info`'s reads.

| Argument | Default | Meaning |
| --- | --- | --- |
| `timeout` | `30` | Per-request deadline in seconds. Pass `None` for no deadline |
| `pool_maxsize` | `100` | Connection-pool size |
| `on_request` | `None` | `on_request(url_path, body, trace_id)` — called before each request; `trace_id` is the id being sent, or `None` |
| `on_response` | `None` | `on_response(url_path, status, body, elapsed_ms, trace_id)` — called after each response; `trace_id` is the `x-trace-id` the gateway echoed, or `None` |
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

To capture only the id the gateway itself generates, send none and read the `trace_id` argument of `on_response`.

## Constants

Exposed under `native_core.constants`.

| Constant | Value | Notes |
| --- | --- | --- |
| `TESTNET_API_URL` | `https://api-test.native.org` | The only usable gateway today |
| `MAINNET_API_URL` | `https://api.native.org` | **Defined but blocked** — constructing `Info` / `Exchange` against it raises `LocalValidationError` |
| `TESTNET_CHAIN_ID` | `969696` | Part of every signed payload |
| `MAINNET_CHAIN_ID` | `696969` | Defined for completeness only |
| `NETWORK_URLS` | `{"testnet": ..., "mainnet": ...}` | Logical network name → gateway URL; the bundle's `network` field maps through this |
| `CHAIN_ID_BY_URL` | `{url: chain_id}` | The chain id is derived from the gateway URL |

{% hint style="warning" %}
Mainnet is not usable through the SDK yet — an API wallet can only be approved on the testnet site. A mainnet bundle or a hand-built client against `MAINNET_API_URL` raises `LocalValidationError` at construction.
{% endhint %}

## See also

* [Getting started](getting-started.md) — mint an API wallet and place your first order.
* [Core concepts](core-concepts.md) — accepted-vs-filled, the reconcile-by-cloid contract, and one `Exchange` per key.
* [AI agents and MCP](ai-agents-and-mcp.md) — driving the SDK from an agent, and the bundled `native-core-mcp` server.
* [Troubleshooting](troubleshooting.md) — what each error means and how to fix it.
* [`POST /info`](../post-info.md) and [`POST /trade`](../post-trade.md) — the wire reference for the two endpoints these classes wrap.
* [Decimal units](../decimals-units.md) and [Transaction signing](../transaction-signing.md) — raw/display conversion and the signature the SDK builds.
