---
description: A thin, typed, synchronous Python client over the Native Core gateway's POST /info and POST /trade.
---

# Python SDK

`native-core-python-sdk` is a thin, typed, synchronous Python client over the Native Core gateway. It wraps the two REST endpoints — reads through `POST /info`, writes through `POST /trade` — as two classes that return the gateway's JSON as plain, `TypedDict`-annotated dicts. Import name `native_core`; requires Python 3.10+; runtime deps are `requests`, `eth-account`, and `eth-utils`.

```bash
pip install native-core-python-sdk
```

{% hint style="warning" %}
**Testnet only · pre-1.0.** The Native Core Python SDK is `v0.1.0` (alpha) and connects to **testnet only** — mainnet is not yet enabled. The API may change before 1.0; pin an exact version: `pip install native-core-python-sdk==0.1.0`.
{% endhint %}

## What it is

Two classes, split by direction. `Info` handles reads (market data, balances, order status); `Exchange` handles writes (place, cancel, modify, batch) and owns an internal `Info` as `exchange.info`. Both build from a connection bundle you mint once in the web app:

```python
from native_core import Exchange, is_accepted, order_state

exchange = Exchange.from_bundle("bundle.json")   # dict, JSON string, or file path
info = exchange.info                             # reads share the same client
```

* **`Info` + `Exchange`.** One reads, one writes. Construct with `Info.from_bundle(...)` / `Exchange.from_bundle(...)`, or `Info(base_url)` / `Exchange(wallet, base_url, owner=...)` by hand.
* **Poll-based.** Calls are synchronous and blocking. `exchange.place(...)` submits and waits for the matching outcome; `info.wait_for_open`, `info.wait_for_order`, and `info.reconcile_by_cloid` resolve an order by its `cloid`.
* **Decision-ready fields.** A business rejection is data on the response body, not an exception. Helpers such as `is_accepted`, `is_rejected`, `is_timeout`, `next_action`, and `order_state` collapse a response into one field to branch on, so you never parse prose.

## What it is NOT

* **No WebSocket or streaming.** There is no async API and no push feed — you poll. Wrap calls in your own executor if you need concurrency, and share **one** `Exchange` per API wallet across threads (the nonce is a per-instance, lock-guarded, monotonic counter; two instances on one key collide nonces).
* **It cannot move funds.** The SDK signs with an API wallet, a protocol-level agent key scoped only to placing and cancelling orders. Deposits, withdrawals, and approving or revoking the API wallet happen in the Native web app with your main wallet — never in the SDK.
* **Mainnet is not enabled.** This release trades on testnet only. Constructing an `Info` or `Exchange` against the mainnet gateway raises `LocalValidationError`.

## In this section

{% content-ref url="getting-started.md" %}
[getting-started.md](getting-started.md)
{% endcontent-ref %}

{% content-ref url="core-concepts.md" %}
[core-concepts.md](core-concepts.md)
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

* [../api-access.md](../api-access.md) — calling the raw gateway directly, without the SDK.
* [../post-trade.md](../post-trade.md) and [../post-info.md](../post-info.md) — the wire reference for the two endpoints the SDK wraps.
