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

{% hint style="warning" %}
**Testnet only.** Native Core is currently available on **testnet only**, at `https://api-test.native.org`. Everything in this section targets testnet.
{% endhint %}

## Start here

New to Native Core? Start with the access model — environments, the API-wallet model, and how to send your first request:

{% content-ref url="api-access.md" %}
[api-access.md](api-access.md)
{% endcontent-ref %}

Prefer a client library? The official Python SDK wraps both endpoints, signs `/trade` for you, and ships an MCP server for AI agents:

{% content-ref url="python-sdk/" %}
[python-sdk](python-sdk/)
{% endcontent-ref %}

## Conventions

* `API_URL` examples use `https://api-test.native.org`.
* Hex values are `0x`-prefixed lowercase strings unless noted otherwise.
* Protocol numeric fields in signed actions are decimal strings; integer-valued fields (ids, nonces) also accept an unsigned JSON integer, while decimal price/quantity/amount fields must be strings.
* Business query responses include `query_height` and `app_hash` when a query view is available. `query_height` is the execution height represented by the published read view.
* `POST /trade` accepts an optional `x-trace-id` HTTP header for request correlation; when it is absent or invalid the API mints one, and the `/trade` response always echoes it in `x-trace-id`. `POST /info` does not process or return `x-trace-id`.
* `POST /trade` is **synchronous**: it returns the transaction's executed outcome in `submission_status` — an order's lifecycle state (`resting`/`filled`/`cancelled`/`rejected`), `success`/`failed` for non-order actions, or `timeout` when the outcome is not observed within the wait budget — with a `tx_hash` and, on failure, an `error.code`. Business queries such as `orderStatus`, `openOrders`, `userFills`, `userBalances`, `spotCreditPositions`, `spotCreditAccount`, `batchOrderStatus`, `queryStatus`, `oracleStatus`, and `markPrices` re-read state or reconcile a `timeout` by `cloid`; they are no longer the primary way to learn a write's outcome.

## Concepts

Terminology, the API-wallet and nonce model, and the integer-atom number model:

{% content-ref url="notation.md" %}
[notation.md](notation.md)
{% endcontent-ref %}

{% content-ref url="nonces-and-api-wallets.md" %}
[nonces-and-api-wallets.md](nonces-and-api-wallets.md)
{% endcontent-ref %}

{% content-ref url="decimals-units.md" %}
[decimals-units.md](decimals-units.md)
{% endcontent-ref %}

## Endpoints & signing

The two REST endpoints, how to sign a write, and how to read an error:

{% content-ref url="post-info.md" %}
[post-info.md](post-info.md)
{% endcontent-ref %}

{% content-ref url="post-trade.md" %}
[post-trade.md](post-trade.md)
{% endcontent-ref %}

{% content-ref url="transaction-signing.md" %}
[transaction-signing.md](transaction-signing.md)
{% endcontent-ref %}

{% content-ref url="error-responses.md" %}
[error-responses.md](error-responses.md)
{% endcontent-ref %}

## Changelog

```
2026-07-13: v0.6 - synchronous /trade (returns the executed outcome — order lifecycle status, success/failed, or timeout — rather than admission-only); per-signer rate limiting (RateLimited, 429); place-order suspension breaker (PlaceOrderSuspended, 503); submit-routing timeout codes (HandoffTimeout/HandoffBufferFull/HandoffMultipleActive/node_unreachable); gateway ExpiredTx fast-fail; 64 KiB/256 KiB request body limits (413); x-trace-id scoped to /trade; approveAgent/revokeAgent actions; and deposits/accountingDepositContracts/multisigPolicy info queries.
2026-06-11: v0.5 - add withdraw/settle/repay actions with EIP-712 signing, x-trace-id, queryStatus/accountingWithdrawTokens/withdraws/batchOrderStatus info queries, assets withdraw fees, and revised /trade error codes.
2026-05-30: v0.4 - add cancelAll action, trading-fee fields on userFills, orderStatus pending/null-OID overlay, assets.issuer, expanded /trade error codes, and signing example refresh.
2026-05-13: v0.3 - admission precheck and margin controls, migrate quantity precision to markets, and add quote-asset minimum/allowlist admin and query APIs (plus related fixes).
2026-05-13: v0.2 - add decimals related, put sections into subpages
2026-05-12: v0.1 - initial base, with tx signing
```
