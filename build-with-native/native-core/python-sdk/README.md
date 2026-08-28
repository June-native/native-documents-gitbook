---
description: >-
  A thin, typed, synchronous Python client over Native Core's POST /info, POST
  /trade, and WebSocket feeds.
---

# Python SDK

`native-core-python-sdk` is a thin, typed, synchronous Python client over Native Core. Three classes over two transports: reads through `POST /info`, writes through `POST /trade`, live data over the [WebSocket](../websocket.md). All of them return the API's JSON as plain, `TypedDict`-annotated dicts. Import name `native_core`; requires Python 3.10+; runtime deps are `requests`, `eth-account`, `eth-utils`, and `websocket-client`.

```bash
pip install native-core-python-sdk==2.0.0
```

These pages document **2.0.0**. Upgrading from 1.x? Read [Accepted is not placed](core-concepts.md#accepted-is-not-placed) first — `submission_status: "accepted"` no longer means your order succeeded.

## What it is

`Info` handles reads (market data, balances, order status). `Exchange` handles writes (place, cancel, modify, batch) and owns an internal `Info` as `exchange.info`. `WsClient` streams. `Info` and `Exchange` build from a connection bundle you create once in the web app:

```python
from native_core import Exchange, is_accepted, is_order_failed

exchange = Exchange.from_bundle("bundle.json")   # dict, JSON string, or file path
info = exchange.info                             # reads share the same client
ws = info.ws().connect()                         # streaming, on the same endpoint
```

* **`Info` + `Exchange` + `WsClient`.** One reads, one writes, one streams. Construct the first two with `Info.from_bundle(...)` / `Exchange.from_bundle(...)`, or `Info(base_url)` / `Exchange(wallet, base_url, owner=...)` by hand. `WsClient` is not built from a bundle: use `info.ws()` / `exchange.ws()`, which pass the loaded market table in, or `WsClient(url)`.
* **Synchronous.** Calls block for the result. `exchange.place(...)` submits, then reads the matching `orderStatus` unless the response already settled the order; `info.wait_for_open`, `info.wait_for_order`, and `info.reconcile_by_cloid` resolve an order by its `cloid`.
* **Decision-ready fields.** A business rejection is data on the response body, not an exception. `submission_status` is the verdict on the **transaction**; the `response` envelope is the verdict on your **order**. `is_accepted` reads the first, `is_order_failed` / `leaf_error_code` read the second, and `next_action` folds both into one of six strings to branch on.

## What it is NOT

* **Not async.** There is no `async`/`await` anywhere. HTTP calls block — wrap them in your own executor if you need concurrency, and share **one** `Exchange` per API wallet across threads (the nonce is a per-instance, lock-guarded, monotonic counter; two instances on one key collide nonces). `WsClient` does run one background thread: stream callbacks execute on it, so a callback that blocks stalls the reader.
* **It cannot move funds.** The SDK signs with an API wallet, a protocol-level agent key scoped only to placing and cancelling orders. Deposits, withdrawals, and approving or revoking the API wallet happen in the Native web app with your main wallet — never in the SDK.
* **Mainnet and testnet.** It trades on whichever network your connection bundle names.

## In this section

{% content-ref url="getting-started.md" %}
[getting-started.md](getting-started.md)
{% endcontent-ref %}

{% content-ref url="core-concepts.md" %}
[core-concepts.md](core-concepts.md)
{% endcontent-ref %}

{% content-ref url="streaming.md" %}
[streaming.md](streaming.md)
{% endcontent-ref %}

{% content-ref url="api-reference.md" %}
[api-reference.md](api-reference.md)
{% endcontent-ref %}

{% content-ref url="ai-agents-and-mcp.md" %}
[ai-agents-and-mcp.md](ai-agents-and-mcp.md)
{% endcontent-ref %}

{% content-ref url="examples.md" %}
[examples.md](examples.md)
{% endcontent-ref %}

{% content-ref url="troubleshooting.md" %}
[troubleshooting.md](troubleshooting.md)
{% endcontent-ref %}

## See also

* [../api-access.md](../api-access.md) — calling the raw API directly, without the SDK.
* [../post-trade.md](../post-trade.md) and [../post-info.md](../post-info.md) — the wire reference for the two endpoints the SDK wraps.
