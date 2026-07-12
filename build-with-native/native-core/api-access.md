---
description: How to reach the Native Core gateway — endpoints, environments, API wallets, signing, and limits.
---

# Gateway API Access

The Native Core gateway is a single HTTP front door with exactly two REST endpoints:

* `POST /info` — **reads**. One endpoint for all public reads, dispatched by a top-level `type` field (market metadata, order books, balances, order status, fills).
* `POST /trade` — **writes**. One client-signed action per call (`order`, `cancel`, `cancelAll`, `modify`, `batch`, and the owner-signed `withdraw` / `settle` / `repay`).

There is no WebSocket or streaming feed — reads are poll-based over `POST /info`. This page is the front door for anyone integrating **directly** with the gateway: market makers, trading bots, AI-agent builders, and aggregators.

{% hint style="warning" %}
**Testnet only.** The Native Core gateway is currently available on **testnet only**, at `https://api-test.native.org`. Everything on this page targets testnet.
{% endhint %}

There are two ways to integrate:

* **Call the gateway directly** — the two REST endpoints documented on this page. Full control, any language.
* **Use the [Native Core Python SDK](python-sdk/README.md)** — a thin, typed client that wraps both endpoints and handles `/trade` signing, nonces, and accepted-vs-filled reconciliation for you.

Signing `/trade` by hand (a canonical binary payload with recoverable secp256k1 signatures and per-signer monotonic nonces) is involved, so the SDK is the faster path — but the wire format below is fully specified if you need to build your own client.

## Environment

The gateway is served over HTTPS. The examples below use:

```bash
API_URL=https://api-test.native.org
```

You fund a trading account by depositing **Arbitrum Sepolia** testnet assets from your main wallet in the web app (there is no faucet — bring your own). See the [access model](#access-model-api-wallets) below for the deposit-and-mint flow.

## Access model — API wallets

Writes are authorized by an **API wallet**: a protocol-level *agent* key scoped only to placing and cancelling orders. An API wallet can trade your account's balance, but it can **never** move funds off Native — deposits, withdrawals, and agent approval all require your main wallet. That scoping (and the fact that it is revocable) is what makes it safe to run in an unattended bot.

You mint one in the **[Native web app](https://app-uat.native.org/markets/ETH-USDT?network=testnet&agentWallets=agents)** (testnet), not through the gateway — that link opens the **API wallets** panel directly:

1. Connect your **main wallet** and deposit **Arbitrum Sepolia** testnet assets. There is no faucet — bring your own. Your trading account is created on the first deposit.
2. In the **API wallets** panel, click **Create API wallet**. Your main wallet signs one `approveAgent`, and the app shows a one-time **connection bundle** containing the agent key. Copy it now; it is shown once.
3. Revoke or rotate the API wallet in the app at any time.

The connection bundle is a small JSON object:

```jsonc
{
  "network": "testnet",       // which gateway to use
  "accountAddress": "0x…",    // your main wallet — the account the bot trades on
  "agentPrivateKey": "0x…"    // the only signing secret; the SDK signs /trade with this
}
```

`accountAddress` is used only locally to identify the account you trade on — it never goes on the wire. The gateway recovers the signer from the signature. For the API wallet's authority model, replay protection, and safety rules, see:

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

## Authentication & signing

Every `/trade` write is a **client-signed transaction**. The gateway reconstructs a canonical unsigned binary payload from `action`, `nonce`, `agent_epoch`, and `expires_after_ms`, then verifies the recoverable secp256k1 signature over that exact payload — the transaction **authority** is the recovered signer. The owner address is never sent.

* **Trading actions** (`order`, `cancel`, `cancelAll`, `modify`, `batch`) use the default legacy binary scheme (`auth_scheme: "legacy"`) and are signed by the **API wallet** key.
* **Owner `/trade` actions** (`withdraw` / `settle` / `repay`) are **EIP-712** (`auth_scheme: "eip712"`) and are signed by your **main wallet** — not with the API wallet, and not part of a bot's hot path.
* **Agent approval** (`approveAgent` / `revokeAgent`) is also a main-wallet EIP-712 signature, but it happens **in the Native web app**, not as a gateway `/trade` action — it is how you mint or revoke the API wallet in the first place.

The exact payload layout, encoding rules, and EIP-712 typed-data schemes are here:

{% content-ref url="transaction-signing.md" %}
[transaction-signing.md](transaction-signing.md)
{% endcontent-ref %}

## Numbers

Native Core executes on **integers only**. Public `order` / `modify` payloads accept human decimal strings for `price` and `quantity`; the write path converts them to raw atoms using each market's `price_decimals`, `max_price_sig_figs`, and `base_quantity_decimals`. Send prices and sizes as **strings**, never floats, and respect each market's precision — an out-of-precision value is rejected, not silently rounded.

{% content-ref url="decimals-units.md" %}
[decimals-units.md](decimals-units.md)
{% endcontent-ref %}

## Request correlation

Both endpoints accept an optional `x-trace-id` request header and **echo it back** in the response, so you can line a request up with the gateway's own logs. If you omit it or send an invalid value, the gateway generates one; the response always includes an `x-trace-id`.

```bash
curl -sS -X POST "$API_URL/info" \
  -H 'content-type: application/json' \
  -H 'x-trace-id: client-trace-001' \
  -d '{"type":"queryStatus"}'
```

## Rate limits & errors

The gateway enforces rate limits. Nonce validation and rate limiting are **authority-scoped** — keyed on the recovered signer — so one API wallet is one bucket (see [Nonces & API Wallets](nonces-and-api-wallets.md)). The exact numeric limits are not exposed on the wire; get them from the platform.

The `/trade` error model keys on the **response body, not the HTTP status**. The gateway returns the same trade-response shape for HTTP 200 / 400 / 429 / 503 / 504, with `submission_status` in `{ accepted, rejected, timeout }`. A business rejection is **data**, not a transport error — read `submission_status` and, on a rejection, `error.code`.

Two outcomes decide whether you may resend: a `rejected` + `RateLimited` (throttled, never admitted) is the **only** safe resend — back off `error.retry_after_ms` and resend the same signed action; a `timeout` is indeterminate and must **never** be resent under a new nonce — reconcile by `cloid` instead. For the full outcome table, the codes you actually hit, and their fixes, see:

{% content-ref url="error-responses.md" %}
[error-responses.md](error-responses.md)
{% endcontent-ref %}

## Next step

The fastest way onto the gateway is the Python SDK — it wraps both endpoints, signs `/trade` for you, and turns each of the outcomes above into a decision-ready field.

{% content-ref url="python-sdk/README.md" %}
[README.md](python-sdk/README.md)
{% endcontent-ref %}
