---
description: Wire an AI agent to trade on Native Core — the safety contract, the bundled MCP server, and driving the SDK directly.
---

# AI Agents & MCP

The Native Core Python SDK is built to be driven by an AI agent, not just a person. This page is the agent-integration centerpiece: one safety contract, then two ways to connect — a bundled MCP server that needs no glue code, or the SDK called directly from your own agent loop.

## Why Native Core is agent-friendly

The API and the SDK are shaped so an autonomous model can trade without ever holding a fund-moving key or parsing prose to know what happened.

- **Scoped, revocable keys.** An agent trades with an **API wallet** — a protocol-level agent key scoped only to placing and cancelling orders. It can never deposit, withdraw, or move funds off Native. Create it in the web app, and revoke or rotate it there any time. A leaked API wallet key can trade the balance but cannot drain it. See [getting-started.md](getting-started.md) for how to create one.
- **Reads on, writes gated.** Market data, balances, and order status are always available. The order-placing and cancelling surface is turned on only by an explicit switch, so a read-only agent physically cannot place an order.
- **Outcomes are structured, not prose.** Every trade response is decision-ready: the agent branches on a field (`submission_status`, `next_action`, `state`, `filled_qty`) instead of reading a sentence. A business rejection is **data** in the response body, not an exception.
- **Deterministic reconcile-by-cloid.** Every order carries a client order id (`cloid`) you can look up. That single primitive is what stops an agent from double-filling after a timeout — the core of the safety contract below.

## The safety contract for agents

An agent that places money-moving orders must hold to a few rules. The SDK returns each outcome as structured fields so the agent branches on a field instead of remembering the rule.

- **Accepted is not filled.** A raw `order()` / `market_order()` returning `submission_status: "accepted"` means the transaction **landed and executed** — not that the order rested or filled (it also covers a benign IOC/FOK cancel). Read the order's status by `cloid` — `wait_for_open` for a resting order, or `info.reconcile_by_cloid(user, market, cloid)` — for the `oid` and fill.
- **Never resubmit an uncertain write.** A wire timeout raises `SubmissionUncertain` (carrying `.cloid` and `.nonce`) or returns `submission_status: "timeout"`. The order **may still be live**. Reconcile by `cloid` and act on the result; resubmitting under a fresh nonce risks a double-fill. Only a `RateLimited` rejection is safe to resend — it was never admitted.
- **Survive a restart.** Generate the `cloid` yourself with `Exchange.random_cloid()`, persist `{intent, cloid}` durably **before** calling `order(..., cloid=cloid)`, and on restart reconcile every persisted cloid before placing anything new. This is the idempotency-key pattern: an agent that crashes after sending but before recording an SDK-generated cloid cannot reconcile and may double-fill.
- **Numbers are strings.** Pass `sz` and `limit_px` as `str` or `Decimal`, never `float`. Size with `info.min_order_size(market, price)` and dry-run with `build_order(...)` (which signs and sends nothing) to avoid a precision or minimum-notional rejection. See [decimals-units.md](../decimals-units.md).

`next_action(response)` collapses any trade response into one verdict to branch on (it returns `None` for a non-trade response):

| `next_action` | Situation | What the agent does |
| --- | --- | --- |
| `READ_ORDER_STATUS` | accepted — landed & executed | Read the order's status once (`wait_for_open` / `wait_for_order`) for the `oid` and fill — the outcome is already settled |
| `RECONCILE_BY_CLOID` | timeout — indeterminate | `reconcile_by_cloid`; **never** resubmit |
| `BACKOFF_AND_RETRY` | `RateLimited` — never admitted | Sleep `retry_after_ms`, then resend the same order |
| `FIX_AND_RESUBMIT` | other rejection | Fix the input or account state, then submit fresh |

`as_problem_details(failure)` is the one-envelope normalizer: it renders any failure — an exception, or a rejected / timeout body — into a single flat `{type, title, retryable, next_action, cloids, …}` shape, so an agent has one place to read regardless of how the outcome arrived.

The wire-level fields these helpers read (`submission_status`, `error.code`, `orderStatus`) are documented in [post-trade.md](../post-trade.md) and [post-info.md](../post-info.md).

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

Every result is normalized with the SDK's own helpers, so the assistant sees a decision-ready shape (`state`, `filled_qty`, `next_action`) rather than raw API JSON. The never-resubmit contract is enforced by shape: a timed-out or uncertain write comes back with `next_action = RECONCILE_BY_CLOID` and the `cloid`, and the model is pointed at `reconcile_order` — never told to resend. These tools are thin wrappers over the same SDK calls (`get_order` ≈ `reconcile_by_cloid`, `get_min_order_size` ≈ `min_order_size`, `preview_order` ≈ `build_order`).

{% hint style="info" %}
Use a dedicated, limited API wallet here, never your main wallet. The MCP tools are convenience, not a security boundary: the real protection is that the API wallet is scoped and revocable in the app.
{% endhint %}

## Option B — drive the SDK from your agent code

If your agent does its own tool-calling (not MCP), call the SDK directly and hand the model the same decision-ready fields. The pattern is **place, then reconcile by `cloid`** — and branch on the verdict, never resubmit on an uncertain one:

```python
from native_core import Exchange

exchange = Exchange.from_bundle(BUNDLE)   # a testnet bundle
order = exchange.place(MARKET, is_buy=True, sz=sz, limit_px=px, tif="gtc")

# Resolve the real outcome by cloid — never resubmit on an uncertain one.
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
from native_core import Exchange, next_action, SubmissionUncertain

exchange = Exchange.from_bundle(BUNDLE)
cloid = Exchange.random_cloid()

store.put(intent, cloid)                  # persist {intent, cloid} BEFORE sending

try:
    resp = exchange.order(MARKET, is_buy=True, sz=sz, limit_px=px, tif="gtc", cloid=cloid)
except SubmissionUncertain as e:          # signed + sent, then timed out
    reconcile(MARKET, e.cloid)            # look up e.cloid; never resubmit
else:
    verdict = next_action(resp)
    if verdict == "READ_ORDER_STATUS":
        exchange.info.wait_for_open(exchange.effective_account, MARKET, cloid)
    elif verdict == "RECONCILE_BY_CLOID":
        reconcile(MARKET, cloid)          # timeout body — never resubmit
    elif verdict == "BACKOFF_AND_RETRY":
        ...                               # RateLimited: sleep retry_after_ms, resend same order
    elif verdict == "FIX_AND_RESUBMIT":
        ...                               # other rejection: fix input, submit fresh
```

On restart, reconcile every persisted `cloid` with `reconcile_by_cloid` before placing anything new. Share one `Exchange` per API wallet across threads — the nonce is a per-instance, lock-guarded, monotonic counter, and two instances on the same key collide nonces. Deposits, withdrawals, and `approveAgent` stay in the web app, signed by the main wallet ([transaction-signing.md](../transaction-signing.md)).

## The docs are agent-readable too

This documentation site is itself consumable by an agent — distinct from the trading MCP server above (that one places orders; this one answers questions about the docs).

- **Every page is Markdown.** Append `.md` to any docs URL to fetch the raw Markdown of that page.
- **`llms.txt` index.** A machine-readable index of the site an agent can crawl for the full page list.
- **A read-only docs MCP endpoint** at `https://docs.native.org/native-dev/~gitbook/mcp`, exposing `searchDocumentation` and `getPage` — connect an agent to it for Q&A over these docs.

```jsonc
{
  "mcpServers": {
    "native-docs": {
      "url": "https://docs.native.org/native-dev/~gitbook/mcp"
    }
  }
}
```

This endpoint only reads documentation; it holds no key and cannot trade. Use it for retrieval and grounding, and the `native-core` server above for trading.
