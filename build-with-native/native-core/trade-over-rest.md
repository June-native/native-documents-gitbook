---
description: Zero to a live order over REST — the fastest path to your first Native Core trade.
---

# Trade over REST

One signed request per call, and the call blocks for the outcome — no socket to manage. This page takes you from nothing to a working order.

The examples below use mainnet, `API_URL=https://api.native.org`. To integrate against testnet, swap in `https://api-test.native.org` and sign with the testnet chain id — see [environments](api-access.md#environments).

## 1. Get an API wallet

An API wallet is an agent keypair you generate locally, authorized by one owner-signed `approveAgent`. Provision it directly against the API — see [creating an API wallet](api-access.md#access-model-api-wallets) for the full flow:

1. Generate an agent keypair locally; its 20-byte address is the `agent`.
2. Deposit the quote asset you'll trade from your main wallet — your account is created on the first deposit.
3. Your main wallet signs one `approveAgent` (EIP-712) authorizing the agent. The **[Native web app](https://app-uat.native.org/markets/ETH-USDT?agentWallets=agents)** can do this for you and return the same values as a one-time connection bundle:

```jsonc
{
  "network": "mainnet",
  "accountAddress": "0x…",   // your account (owner) — you read and sign-as-agent against this
  "agentPrivateKey": "0x…"   // the only signing secret
}
```

The API wallet places and cancels orders; it can **never** move funds. See [Account Types](account-types.md) if you were granted a credit account.

## 2. Find your market

Markets and their precision are public. You need the `market_id` and its decimals to build an order.

```bash
curl -sS -X POST "$API_URL/info" -H 'content-type: application/json' \
  -d '{"type":"markets"}'
```

Note the `market_id`, `price_decimals`, and `base_quantity_decimals` for the pair you want to trade.

## 3. Get your agent_epoch

Every trading write signed by an API wallet must carry `agent_epoch` — the current generation of your wallet's approval. Read it once at startup:

```bash
curl -sS -X POST "$API_URL/info" -H 'content-type: application/json' \
  -d '{"type":"userAgents","user":"<accountAddress>"}'
```

Use the `epoch` of the slot whose `agent` matches your API-wallet address. Omit `agent_epoch` and the write is rejected with `DirectSignerIsActiveAgent`.

## 4. Sign and place an order

`signature` signs a canonical **binary** payload built from `action` + `nonce` + `agent_epoch` — not the JSON text. Two ways to produce it:

* **[Python SDK](python-sdk/README.md)** — signs, manages nonces, and reconciles for you. Recommended.
* **By hand** — the full byte layout and a TypeScript signer are in [Transaction Signing](transaction-signing.md).

The assembled request:

```bash
curl -sS -X POST "$API_URL/trade" -H 'content-type: application/json' \
  -d '{
    "action": {
      "type": "order", "market_id": "2",
      "side": "bid", "order_type": "limit", "tif": "gtc",
      "price": "3500.00", "quantity": "1.0000",
      "cloid": "0x11111111111111111111111111111111"
    },
    "nonce": "1760000000000",
    "agent_epoch": "3",
    "signature": "0x…"
  }'
```

Always set a `cloid` — it is how you reconcile a `timeout` (step 5). Send `price` and `quantity` as strings at the market's precision.

## 5. Read the outcome

`/trade` is synchronous: the call blocks (~1s) and returns the result in `submission_status`.

| `submission_status` | Meaning | Do next |
| --- | --- | --- |
| `accepted` | The order landed and executed. | To see whether it rested or filled, read [`orderStatus`](post-info.md#orderstatus). |
| `rejected` | Refused, or failed at execution. `error.code` says why. | Fix the cause and submit a **fresh** action. If `RateLimited`, back off `error.retry_after_ms` and resend the same one. |
| `timeout` | The outcome was not observed in time. **May still land.** | Reconcile by `cloid`; never resubmit under a new nonce. |

```json
{ "submission_status": "accepted", "tx_hash": "0x…" }
```

That is a full round trip. The order is working — list it with [`openOrders`](post-info.md#openorders), and cancel with a [`cancel`](post-trade.md#cancel) action.

## Next steps

* [POST /trade](post-trade.md) — every action and envelope field
* [POST /info](post-info.md) — every read
* [Transaction Signing](transaction-signing.md) — sign it yourself
* [Error responses](error-responses.md) — every code and what to do about it
