---
description: Wire an AI agent to trade on Native Core — the safety contract, the bundled MCP server, and driving the SDK directly.
---

# AI Agents & MCP

The Native Core Python SDK is built to be driven by an AI agent, not just a person. This page is the agent-integration centerpiece: one safety contract, then two ways to connect — a bundled MCP server that needs no glue code, or the SDK called directly from your own agent loop.

## Why Native Core is agent-friendly

The API and the SDK are shaped so an autonomous model can trade without ever holding a fund-moving key or parsing prose to know what happened.

- **Scoped, revocable keys.** An agent trades with an **API wallet** — a protocol-level agent key scoped only to placing and cancelling orders. It can never deposit, withdraw, or move funds off Native. Create it in the web app, and revoke or rotate it there any time. A leaked API wallet key can trade the balance but cannot drain it. See [Getting Started](getting-started.md) for how to create one.
- **Reads on, writes gated.** Market data, balances, and order status are always available. The order-placing and cancelling surface is turned on only by an explicit switch, so a read-only agent physically cannot place an order.
- **Outcomes are structured, not prose.** Every trade response is decision-ready: the agent branches on a field (`submission_status`, `next_action`, `order_error`, `state`, `filled_qty`) instead of reading a sentence. A business rejection is **data** in the response body, not an exception.
- **Deterministic reconcile-by-cloid.** An order whose outcome is *uncertain* carries a client order id (`cloid`) you can look up, which is what stops an agent from double-filling after a timeout. It does not extend to an order the response already settled as failed: most failures are never written anywhere, so there is nothing to look up ([which ones](core-concepts.md#what-a-failed-order-leaves-behind)). See the contract below.

## The safety contract for agents

An agent that places money-moving orders must hold to a few rules. The SDK returns each outcome as structured fields so the agent branches on a field instead of remembering the rule.

- **Accepted is not placed.** `submission_status: "accepted"` means the **transaction** landed and executed. The order inside it may still have failed — `tick`, `lotsize`, `badalopx`, `insufficientspotbalance`, `mintradespotntl`, `missingorder` all arrive that way, on a leaf of the `response` envelope. `is_accepted` alone will book a failed order as a success. Check `is_order_failed(resp)`, or just take `next_action(resp)`. The `oid` and the fill are on the response too (`order_oid`, `fill`), so the ordinary path costs no read.
- **Most failed orders cannot be reconciled.** They never entered the book, so polling returns `unknown` and reconciling returns `undetermined`, forever — which, paired with the never-resubmit rule, freezes the agent. See [what a failed order leaves behind](core-concepts.md#what-a-failed-order-leaves-behind). Read the leaf, record the failure, and treat the corrected order as a **new** order with a new `cloid`.
- **Never resubmit an uncertain write.** `SubmissionUncertain` (carrying `.cloids` and `.nonce`) or `submission_status: "timeout"` means the order **may still be live**. Reconcile by `cloid` and act on the result; resubmitting risks a double-fill. Two things *are* safe to resend: a `RateLimited` rejection, which was never admitted, and a timeout where `is_safe_to_resend(resp)` is true (the `Handoff*` family, which never reached a node).
- **Survive a restart.** Generate the `cloid` yourself with `Exchange.random_cloid()`, persist `{intent, cloid}` durably **before** calling `order(..., cloid=cloid)`, and on restart resolve every persisted cloid before placing anything new — `info.batch_order_status([...])` settles up to 20 in one read. This is the idempotency-key pattern: an agent that crashes after sending but before recording an SDK-generated cloid cannot reconcile and may double-fill.
- **Numbers are strings.** Pass `sz` and `limit_px` as `str` or `Decimal`, never `float`. Size with `info.min_order_size(market, price)` and dry-run with `build_order(...)` (which signs and sends nothing) to avoid a precision or minimum-notional rejection. See [Decimals & Units](../decimals-units.md).

`next_action(response)` collapses any trade response into one verdict to branch on (it returns `None` for a non-trade response). There are six, and against the API running today the first two are what an agent sees most:

| `next_action` | Situation | What the agent does |
| --- | --- | --- |
| `USE_RESPONSE_OUTCOME` | accepted, the order worked | Nothing more. Take the `oid` and fill off the response |
| `ORDER_CLOSED_UNFILLED` | accepted, benign cancel | Nothing more. The order is over and nothing filled |
| `FIX_AND_RESUBMIT` | accepted but the order failed, or a rejection other than `RateLimited` | Read the leaf code, fix the input or account state, send a **fresh** order. Nothing to reconcile |
| `BACKOFF_AND_RETRY` | `RateLimited`, or a `Handoff*` timeout — never reached a node | Sleep `retry_after_ms`, then resend the **same** `cloid` |
| `RECONCILE_BY_CLOID` | a timeout that is not safe to resend | `reconcile_by_cloid`; **never** resubmit |
| `READ_ORDER_STATUS` | accepted with no `response` envelope | Read `order_status` once. Only an API older than the release that reports outcomes inline answers this way |

`as_problem_details(failure)` is the one-envelope normalizer: it renders any failure — an exception, a rejected or timeout body, or an accepted write whose order failed — into a single flat `{type, title, retryable, next_action, cloids, …}` shape, so an agent has one place to read regardless of how the outcome arrived. A success or a benign cancel returns `None`.

The wire-level fields these helpers read (`submission_status`, `error.code`, `orderStatus`) are documented in [POST /trade](../post-trade.md) and [POST /info](../post-info.md).

## Option A — the MCP server (no glue code)

The SDK ships an optional [Model Context Protocol](https://modelcontextprotocol.io) server that lets any MCP-speaking assistant (such as Claude Desktop) read the market and — when explicitly enabled — place orders through the SDK, with no glue code from you. Install it with the `mcp` extra:

```bash
pip install "native-core-python-sdk[mcp]"
```

Run it over stdio with the bundled console script, or as a module:

```bash
native-core-mcp
# or
python -m native_core.mcp
```

Two design rules make this surface safe for an autonomous model:

- **Read-only by default.** The order-placing and cancelling tools are registered only when trading is turned on **and** a key and account are present. A read-only deployment does not register the write tools at all — it physically cannot place an order.
- **Credentials come from the environment, never as a model-visible argument.** The model chooses actions; it never sees or sets the key.

### Configuration

The server reads its configuration from these environment variables:

| Variable | Purpose |
| --- | --- |
| `NATIVE_CORE_BUNDLE` | Path to a connection bundle JSON (`{network, accountAddress, agentPrivateKey}`). Preferred — supplies everything on its own. |
| `NATIVE_CORE_NETWORK` | `testnet` (default) when not using a bundle. |
| `NATIVE_CORE_BASE_URL` | Endpoint URL, as an alternative to `NATIVE_CORE_NETWORK`. |
| `NATIVE_CORE_ACCOUNT` | Your account address (the owner main wallet) — needed for account-scoped reads **and** to register the write tools. |
| `NATIVE_CORE_AGENT_KEY` | The API wallet's private key, required for trading. |
| `NATIVE_CORE_ENABLE_TRADING` | Set to `1` (or `true`/`yes`/`on`) to register the write tools. Off by default: without it the server is read-only even if a key is present. |

To connect it to Claude Desktop, add this to its MCP config, pointing `NATIVE_CORE_BUNDLE` at your bundle file:

```jsonc
{
  "mcpServers": {
    "native-core": {
      "command": "native-core-mcp",
      "env": {
        "NATIVE_CORE_BUNDLE": "/path/to/your/bundle.json",
        "NATIVE_CORE_ENABLE_TRADING": "1"
      }
    }
  }
}
```

Drop `NATIVE_CORE_ENABLE_TRADING` (or leave it unset) for a read-only assistant that can watch the market and your orders but cannot trade.

### Tool inventory

**Read tools — always registered:**

| Tool | Does |
| --- | --- |
| `whoami` | The configured account, endpoint, and whether trading is enabled |
| `list_markets` | Tradable markets with their id and precision |
| `get_orderbook` | Top bids and asks for one market |
| `get_balances` | Spot balances for the configured account |
| `list_open_orders` | Open orders — one market, or all markets |
| `get_order` | Current state of one order by its `cloid` (poll a resting order) |
| `get_fills` | Recent executions, newest first |
| `get_min_order_size` | Smallest order size at a price that clears the minimum notional |
| `reconcile_order` | Did an order land? The uncertain / timeout recovery path |

**Write tools — registered only when `NATIVE_CORE_ENABLE_TRADING` is truthy and a key + account are present:**

| Tool | Does |
| --- | --- |
| `preview_order` | Dry run: validate a limit order without placing it (nothing is sent) |
| `place_limit_order` | Place a limit order (`gtc` / `ioc` / `fok` / `alo`) |
| `place_market_order` | Place a market order with a slippage cap in basis points |
| `cancel_order` | Cancel one order by `oid` or `cloid` |
| `cancel_all_orders` | Cancel every open order in one market |

Every **write** result is normalized with the SDK's own helpers, so the assistant sees a decision-ready shape rather than raw API JSON: `ok`, `cloid`, `submission_status`, `state`, `filled_qty`, `next_action`, `tx_hash`, `error`, plus `oid` and — when a leaf failed — `order_error` (the lowercase per-action code) and `order_failed`. Of the reads, only `get_balances` and `list_open_orders` pass the API's JSON straight through; the rest are reshaped.

`ok` is decided by the order's own outcome, not by the transaction: when the `response` envelope is present, any error leaf means the placement did not work. A cancel result is narrower, `{ok, submission_status, error}` only.

The never-resubmit contract is enforced by shape: a write whose outcome the transport could not determine comes back with `next_action = RECONCILE_BY_CLOID` and the `cloid`, and the model is pointed at `reconcile_order` — never told to resend. A gateway `timeout` body is the exception and is split by `is_safe_to_resend`: the three `Handoff*` codes report `BACKOFF_AND_RETRY`, every other timeout reports `RECONCILE_BY_CLOID`.

These tools are thin wrappers over the same SDK calls (`get_order` ≈ `reconcile_by_cloid`, `get_min_order_size` ≈ `min_order_size`, `preview_order` ≈ `build_order`).

{% hint style="info" %}
Use a dedicated, limited API wallet here, never your main wallet. The MCP tools are convenience, not a security boundary: the real protection is that the API wallet is scoped and revocable in the app.
{% endhint %}

## Option B — drive the SDK from your agent code

If your agent does its own tool-calling (not MCP), call the SDK directly and hand the model the same decision-ready fields. The pattern is **read the response first, reconcile only what it left unresolved**:

```python
from native_core import (
    Exchange, is_order_failed, leaf_error_code, next_action,
    RECONCILE_BY_CLOID, USE_RESPONSE_OUTCOME,
)

exchange = Exchange.from_bundle(BUNDLE)   # your connection bundle
order = exchange.place(MARKET, is_buy=True, sz=sz, limit_px=px, tif="gtc")
submission = order["submission"]

if is_order_failed(submission):
    ...   # settled: the order failed. Nothing to reconcile — fix and send a NEW cloid.
    raise SystemExit(leaf_error_code(submission))

verdict = next_action(submission)
if verdict == USE_RESPONSE_OUTCOME:
    oid = order["oid"]                    # the response carries the outcome
elif verdict != RECONCILE_BY_CLOID:
    ...   # ORDER_CLOSED_UNFILLED / FIX_AND_RESUBMIT / BACKOFF_AND_RETRY: no live order
else:
    # Only here is the outcome genuinely unknown.
    verdict = exchange.info.reconcile_by_cloid(
        exchange.effective_account, MARKET, order["cloid"]
    )
    if verdict["undetermined"]:
        ...   # not confirmed yet: keep reconciling by cloid, do NOT re-place
    elif verdict["is_filled"]:
        ...   # fully filled
    elif verdict["filled_qty"] != "0":
        ...   # partially filled and still resting — filled_qty is how much
    else:
        ...   # resting, unfilled (verdict["state"], e.g. "open")
```

For an agent branching on raw responses, use `next_action` as the single switch, and generate the `cloid` up front so a crash between send and record is recoverable:

```python
from native_core import (
    Exchange, SubmissionUncertain, next_action, order_oid, leaf_error_code,
    USE_RESPONSE_OUTCOME, ORDER_CLOSED_UNFILLED, FIX_AND_RESUBMIT,
    BACKOFF_AND_RETRY, RECONCILE_BY_CLOID, READ_ORDER_STATUS,
)

exchange = Exchange.from_bundle(BUNDLE)
cloid = Exchange.random_cloid()

store.put(intent, cloid)                  # persist {intent, cloid} BEFORE sending

try:
    resp = exchange.order(MARKET, is_buy=True, sz=sz, limit_px=px, tif="gtc", cloid=cloid)
except SubmissionUncertain as e:          # signed + sent, outcome unreadable
    for handle in e.cloids:               # a batch has one per leg
        reconcile(MARKET, handle)         # never resubmit
else:
    verdict = next_action(resp)
    if verdict == USE_RESPONSE_OUTCOME:
        oid = order_oid(resp)             # done: the response carries the outcome
    elif verdict == ORDER_CLOSED_UNFILLED:
        ...                               # benign cancel: over, nothing filled, nothing wrong
    elif verdict == FIX_AND_RESUBMIT:
        ...                               # leaf_error_code(resp): fix input, send a NEW cloid
    elif verdict == BACKOFF_AND_RETRY:
        ...                               # RateLimited or Handoff*: sleep, resend the SAME cloid
    elif verdict == RECONCILE_BY_CLOID:
        reconcile(MARKET, cloid)          # indeterminate timeout — never resubmit
    elif verdict == READ_ORDER_STATUS:
        exchange.info.wait_for_open(exchange.effective_account, MARKET, cloid)
```

On restart, resolve every persisted `cloid` before placing anything new. `info.batch_order_status([...])` settles up to 20 in one read; `reconcile_by_cloid` handles the stragglers.

Share one `Exchange` per API wallet across threads. The nonce is a per-instance, lock-guarded counter, so two instances on one key collide.

Deposits, withdrawals and `approveAgent` stay in the web app, signed by the main wallet ([Transaction Signing](../transaction-signing.md)).

An agent that watches orders continuously should take `orderUpdates` and the book off the [WebSocket](streaming.md) rather than polling: `/info` allows about one read per second per client IP, and a pushed feed costs nothing against that budget.

## The docs are agent-readable too

This documentation site is itself consumable by an agent — distinct from the trading MCP server above (that one places orders; this one answers questions about the docs).

- **Every page is Markdown.** Append `.md` to any docs URL to fetch the raw Markdown of that page.
- **Two crawlable indexes.** [`llms.txt`](https://docs.native.org/native-dev/llms.txt) is the machine-readable page list; [`llms-full.txt`](https://docs.native.org/native-dev/llms-full.txt) is every page's Markdown concatenated into one file, for an agent that would rather read the whole site once than crawl it.
- **A read-only docs MCP endpoint** at `https://docs.native.org/native-dev/~gitbook/mcp`, exposing `searchDocumentation`, `getPage`, and `sendFeedback` (report a page that reads wrong or out of date, so the team can fix it) — connect an agent to it for Q&A over these docs.

```jsonc
{
  "mcpServers": {
    "native-docs": {
      "url": "https://docs.native.org/native-dev/~gitbook/mcp"
    }
  }
}
```

The endpoint takes MCP over `POST`, not a browser `GET`. It only reads documentation: it holds no key and cannot trade. Use it for retrieval and grounding, and the `native-core` server above for trading.
