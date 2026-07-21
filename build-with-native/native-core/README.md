---
layout:
  width: wide
  title:
    visible: true
  description:
    visible: true
  tableOfContents:
    visible: true
  outline:
    visible: true
  pagination:
    visible: true
  metadata:
    visible: false
  tags:
    visible: true
  actions:
    visible: true
---

# Native Core API

The Native Core API is the direct integration surface for the on-chain CLOB. Clients submit signed business actions with `POST /trade` and read business state with `POST /info`. It serves market makers, trading bots, AI agents, and anyone integrating directly with Native Core.

Operational, recovery, and node-private interfaces are intentionally not part of this contract.

Native Core runs on **mainnet** (`https://api.native.org`), with a **testnet** (`https://api-test.native.org`) for integration and testing. See [environments](api-access.md#environments) for base URLs and signing chain ids.

## Start here

New to Native Core? The guides take you from zero to a working integration — start with your first order over REST:

{% content-ref url="trade-over-rest.md" %}
[trade-over-rest.md](trade-over-rest.md)
{% endcontent-ref %}

{% content-ref url="guides.md" %}
[guides.md](guides.md)
{% endcontent-ref %}

Prefer a client library? The official Python SDK wraps both endpoints, signs `/trade` for you, and ships an MCP server for AI agents:

{% content-ref url="python-sdk/" %}
[python-sdk](python-sdk/)
{% endcontent-ref %}

## Conventions

* `API_URL` examples use `https://api.native.org` (mainnet); the testnet base URL is `https://api-test.native.org`.
* Hex values are `0x`-prefixed lowercase strings unless noted otherwise.
* Protocol numeric fields in signed actions are decimal strings; integer-valued fields (ids, nonces) also accept an unsigned JSON integer, while decimal price/quantity/amount fields must be strings.
* Business query responses include `query_height` and `app_hash` when a query view is available. `query_height` is the execution height represented by the published read view.
* `POST /trade` accepts an optional `x-trace-id` HTTP header for request correlation; when it is absent or invalid the API mints one, and the `/trade` response always echoes it in `x-trace-id`. `POST /info` does not process or return `x-trace-id`.
* `POST /trade` is **synchronous**: it waits for the transaction's execution outcome and returns `submission_status` — `accepted` (the transaction landed and executed), `rejected` (refused, or failed at execution), or `timeout` (outcome not observed within the wait budget) — with a `tx_hash` and, on a non-success, an `error.code`. It reports that a write **landed**, not an order's fill state: read `orderStatus` (or `openOrders` / `userFills`) to see whether an order rested or filled, and to reconcile a `timeout` by `cloid`.

## Reference & concepts

The exhaustive contract — every endpoint, field, and error code — and the models it assumes:

{% content-ref url="reference.md" %}
[reference.md](reference.md)
{% endcontent-ref %}

{% content-ref url="concepts.md" %}
[concepts.md](concepts.md)
{% endcontent-ref %}
