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

The Native Core API is the direct integration surface for the on-chain CLOB. Clients submit signed business actions with `POST /trade` and read business state with `POST /info`, or stream both over a [WebSocket](websocket.md) connection. It serves market makers, trading bots, AI agents, and anyone integrating directly with Native Core.

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

* `API_URL` examples use `https://api.native.org` (mainnet); for testnet and signing chain ids, see [environments](api-access.md#environments).
* Hex values are `0x`-prefixed lowercase strings unless noted otherwise.
* Protocol numeric fields in signed actions are decimal strings; integer-valued fields (ids, nonces) also accept an unsigned JSON integer, while decimal price/quantity/amount fields must be strings.
* Business query responses carry `query_height` and `app_hash` when a query view is available — the execution height represented by the published read view.
* `POST /trade` is **synchronous**: it blocks for the execution outcome and returns `submission_status` (`accepted` / `rejected` / `timeout`) with a `tx_hash`. It reports that a write **landed**, not an order's fill state — read the outcome per the [Handle outcomes & timeouts](handle-timeouts.md) playbook.

## Reference & concepts

The exhaustive contract — every endpoint, field, and error code — and the models it assumes:

{% content-ref url="reference.md" %}
[reference.md](reference.md)
{% endcontent-ref %}

{% content-ref url="concepts.md" %}
[concepts.md](concepts.md)
{% endcontent-ref %}
