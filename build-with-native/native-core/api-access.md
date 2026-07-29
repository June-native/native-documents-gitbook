---
description: How to reach Native Core — the endpoints, environments, API wallets, signing, and limits.
---

# API Access

Native Core exposes two REST endpoints and one WebSocket endpoint:

* `POST /info` — **reads**. One endpoint for all public reads, dispatched by a top-level `type` field (market metadata, order books, balances, order status, fills).
* `POST /trade` — **writes**. One client-signed action per call (`order`, `cancel`, `cancelAll`, `modify`, `batch`, and the owner-signed `withdraw` / `settle` / `repay` / `approveAgent` / `revokeAgent`).
* `/ws` — **streaming**. Push channels for books, trades, fills, and order updates, plus the same `/info` and `/trade` bodies sent over the socket. See [WebSocket](websocket.md).

This page is the front door for anyone integrating **directly** with Native Core: market makers, trading bots, AI-agent builders, and aggregators.

There are two ways to integrate:

* **Call the API directly** — the endpoints documented on this page, plus the [WebSocket](websocket.md) surface when you need push instead of polling. Full control, any language.
* **Use the [Native Core Python SDK](python-sdk/README.md)** — a thin, typed client that wraps both endpoints and handles `/trade` signing, nonces, and the synchronous outcome (reconciling a `timeout` by `cloid`) for you.

Signing `/trade` by hand means building a canonical binary payload with recoverable secp256k1 signatures and per-signer monotonic nonces — fully specified below for building your own client. The Python SDK implements it for you.

## Environments

The API is served over HTTPS. Native Core runs on **mainnet**, with a **testnet** for integration and testing. Each environment has its own base URL and signing chain id:

| Environment | Base URL | WebSocket | `network` | Chain id (signing) |
| --- | --- | --- | --- | --- |
| **Mainnet** | `https://api.native.org` | `wss://api.native.org/ws` | `mainnet` | `696969` |
| Testnet | `https://api-test.native.org` | `wss://api-test.native.org/ws` | `testnet` | `969696` |

The chain id is folded into every signed `/trade` payload and must match the environment you target — sign for the wrong one and the API recovers a different authority and rejects the write (see [Transaction signing](transaction-signing.md)). The examples below use mainnet:

```bash
API_URL=https://api.native.org
```

You fund a trading account by depositing assets from your main wallet. See the [access model](#access-model-api-wallets) below for depositing and creating the API wallet.

## Access model — API wallets

Writes are authorized by an **API wallet**: a protocol-level *agent* key scoped only to placing and cancelling orders. An API wallet can trade your account's balance, but it can **never** move funds off Native — deposits, withdrawals, and agent approval all require your main wallet. That scoping (and the fact that it is revocable) is what makes it safe to run in an unattended bot.

An API wallet is just an **agent keypair you generate locally**, authorized on one of your account's agent slots (`0`–`3`) by a single owner-signed `approveAgent`. Set it up directly against the API — no browser required:

1. **Generate an agent keypair** locally (any secp256k1 key). The private key is your only signing secret; it never leaves your process. Its 20-byte address is the `agent`.
2. **Fund the account.** Deposit a supported asset from your main wallet — the per-chain deposit contracts are in [`accountingDepositContracts`](post-info.md#accountingdepositcontracts), and the supported assets, their `asset_id`s, and minimums in [`assets`](post-info.md#assets) / [`accountingWithdrawTokens`](post-info.md#accountingwithdrawtokens). Your trading account is created on the first deposit, which pays a one-time activation fee. [Deposit & Withdraw](../deposit-withdraw/README.md) has the full flow for both directions.
3. **Approve the agent.** Your main wallet signs one `approveAgent` under `auth_scheme:"eip712"` and you `POST /trade` it:

```json
{
  "action": { "type": "approveAgent", "slot_id": "0", "agent": "0x…agentAddress" },
  "nonce": "1717000000007",
  "auth_scheme": "eip712",
  "signature": "0x…"
}
```

The typed-data fields are in [Transaction signing](transaction-signing.md#eip-712-signing-auth_scheme-eip712); the action reference is [`approveAgent`](post-trade.md#approveagent). After approval, read the slot's **`epoch`** from [`userAgents`](post-info.md#useragents) — each entry is `{slot_id, agent, epoch}` — and send that value as the `agent_epoch` envelope field on agent-signed `/trade` writes.

Prefer a UI? The **[Native web app](https://app.native.org/markets/ETH-USDT?agentWallets=agents)** does these same steps for you — connect your main wallet and it generates the agent key, submits the `approveAgent`, and hands you a one-time **connection bundle**. Either way you end up holding the same three values:

```jsonc
{
  "network": "mainnet",       // the network the wallet is bound to
  "accountAddress": "0x…",    // your main wallet — the account the bot trades on
  "agentPrivateKey": "0x…"    // the only signing secret; sign /trade with this
}
```

`accountAddress` is used only locally to identify the account you trade on — it never goes on the wire. the API recovers the signer from the signature. For the API wallet's authority model, replay protection, and safety rules, see:

{% content-ref url="nonces-and-api-wallets.md" %}
[nonces-and-api-wallets.md](nonces-and-api-wallets.md)
{% endcontent-ref %}

## Making requests

Both endpoints take a JSON body over `POST` and return JSON. `POST /info` is unauthenticated; `POST /trade` carries a signed action.

### POST /info (reads)

Dispatch on the top-level `type`. Minimal example — list the tradable markets and their precision:

```bash
curl -sS -X POST "$API_URL/info" \
  -H 'content-type: application/json' \
  -d '{"type":"markets"}'
```

The full per-`type` reference (every query, its parameters, and its response shape) is here:

{% content-ref url="post-info.md" %}
[post-info.md](post-info.md)
{% endcontent-ref %}

### POST /trade (writes)

One signed action per call. Minimal example — a resting limit bid (the `signature` is a secp256k1 signature over the canonical binary payload, not over this JSON text, so you cannot type it by hand — the SDK produces it):

```bash
curl -sS -X POST "$API_URL/trade" \
  -H 'content-type: application/json' \
  -d '{
    "action": {
      "type": "order",
      "market_id": "2",
      "side": "bid",
      "order_type": "limit",
      "tif": "gtc",
      "price": "3500.00",
      "quantity": "1.0000",
      "cloid": "0x11111111111111111111111111111111"
    },
    "nonce": "1760000000000",
    "signature": "0x..."
  }'
```

The full per-action reference (envelope fields, every action type, and the constraints) is here:

{% content-ref url="post-trade.md" %}
[post-trade.md](post-trade.md)
{% endcontent-ref %}

### WebSocket (streaming)

Connect and subscribe — no authentication, no headers:

```bash
wscat -c wss://api.native.org/ws
> {"method":"subscribe","subscription":{"type":"bbo","coin":"2"}}
```

Nine push channels cover books, trades, mids, fills, order updates, and balances, and a `post` method carries the two request bodies above over the same connection. Every channel, payload, and limit is here:

{% content-ref url="websocket.md" %}
[websocket.md](websocket.md)
{% endcontent-ref %}

## Authentication & signing

Every `/trade` write is a **client-signed transaction**. The API reconstructs a canonical unsigned binary payload from `action`, `nonce`, `agent_epoch`, and `expires_after_ms`, then verifies the recoverable secp256k1 signature over that exact payload — the transaction **authority** is the recovered signer. The owner address is never sent.

* **Trading actions** (`order`, `cancel`, `cancelAll`, `modify`, `batch`) use the default legacy binary scheme (`auth_scheme: "legacy"`) and are signed by the **API wallet** key.
* **Owner `/trade` actions** (`withdraw` / `settle` / `repay`) are **EIP-712** (`auth_scheme: "eip712"`) and are signed by your **main wallet** — not with the API wallet, and not part of a bot's hot path.
* **Agent approval** (`approveAgent` / `revokeAgent`) is also a main-wallet EIP-712 signature — it is how you create or revoke the API wallet. Do it in the [Native web app](https://app.native.org/markets/ETH-USDT?agentWallets=agents) or sign it yourself — both are `/trade` action types the API accepts directly.

The exact payload layout, encoding rules, and EIP-712 typed-data schemes are here:

{% content-ref url="transaction-signing.md" %}
[transaction-signing.md](transaction-signing.md)
{% endcontent-ref %}

## Numbers

Native Core executes on **integers only**. Public `order` / `modify` payloads accept human decimal strings for `price` and `quantity`; the write path converts them to raw atoms using each market's `price_decimals` and `base_quantity_decimals` (`max_price_sig_figs` is enforced at execution, not on the write path). Send prices and sizes as **strings**, never floats, and respect each market's precision — an out-of-precision value is rejected, not silently rounded.

{% content-ref url="decimals-units.md" %}
[decimals-units.md](decimals-units.md)
{% endcontent-ref %}

## Request correlation

`POST /trade` accepts an optional `x-trace-id` request header and **echoes it back** in the response, so you can line a write up with the API's own logs. If you omit it or send an invalid value, the API mints one; a `/trade` response always includes an `x-trace-id`. `POST /info` does not process or return `x-trace-id`.

```bash
curl -sS -X POST "$API_URL/trade" \
  -H 'content-type: application/json' \
  -H 'x-trace-id: client-trace-001' \
  -d '{ "action": { ... }, "nonce": "...", "signature": "0x..." }'
```

## Rate limits & errors

Two limiters apply, both returning HTTP `429` with `error.code: "RateLimited"`:

* **Per IP** — **1 request/second** to `/info` and **1 request/second** to `/trade`, as two independent budgets, so a burst of reads never starves your writes. Each is a token bucket holding one second of allowance. Over-quota reads return `{"error":{"code":"RateLimited","message":"ip rate limit exceeded, retry after <ms>ms"}}`; over-quota writes come back in the trade-response shape with an `error.retry_after_ms` hint.
* **Per signer** — writes are additionally limited to **1000 requests/second**, keyed on the recovered [authority](nonces-and-api-wallets.md) over a 1-second sliding window, so one API wallet is one bucket.

For a single integration the **per-IP budget is the one that binds**: it applies before any signature is checked, and 1000 writes/second per signer is only reachable across many source addresses.

Request bodies over **64 KiB** (`/info`) or **256 KiB** (`/trade`) are rejected with HTTP `413`.

A `post` request sent over the [WebSocket](websocket.md#requests-over-the-socket) charges the same per-IP budget as the equivalent HTTP call and is answered with a `429` string in the reply envelope — the socket is a convenience, not extra quota.

The `/trade` error model keys on the **response body, not the HTTP status** — the API returns the same trade-response shape for HTTP 200 / 400 / 429 / 503 / 504, so a business rejection is **data**, not a transport error. Branch on `submission_status`, never the status line. What each outcome means and how to act on it is the [Handle outcomes & timeouts](handle-timeouts.md) playbook; every code and its fix is in:

{% content-ref url="error-responses.md" %}
[error-responses.md](error-responses.md)
{% endcontent-ref %}

## Next step

The Python SDK wraps both endpoints, signs `/trade` for you, and turns each of the outcomes above into a decision-ready field.

{% content-ref url="python-sdk/README.md" %}
[README.md](python-sdk/README.md)
{% endcontent-ref %}
