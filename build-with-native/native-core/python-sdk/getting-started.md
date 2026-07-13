---
description: Install the Native Core Python SDK, create an API wallet, and place your first order on testnet.
---

# Getting Started

The Native Core Python SDK (`native-core-python-sdk`, import `native_core`) is a thin, typed, synchronous client over the gateway's two REST endpoints: `POST /info` for reads and `POST /trade` for writes. This page takes you from `pip install` to a resting order you place and cancel yourself.

{% hint style="warning" %}
**Testnet only · pre-1.0.** The Native Core Python SDK is `v0.1.0` (alpha) and currently runs on **testnet only**. The API may change before 1.0; pin an exact version: `pip install native-core-python-sdk==0.1.0`.
{% endhint %}

## Install

Requires **Python 3.10+**. The runtime dependencies are `requests`, `eth-account`, and `eth-utils` (no `web3`, no `pydantic`).

```bash
pip install native-core-python-sdk
```

Because the SDK is pre-1.0, any release may change the API. Pin an exact version and review the changes before upgrading:

```bash
pip install native-core-python-sdk==0.1.0
```

## Get an API wallet

The SDK trades with an **API wallet**: a protocol-level *agent* key scoped only to placing and cancelling orders. It can never move funds — deposits, withdrawals, and the approval itself are signed by your main wallet in the web app, not by the SDK. You create the API wallet once, in the Native web app:

1. **Connect your main wallet.** Open the **[Native web app](https://app-uat.native.org/markets/ETH-USDT?network=testnet&agentWallets=agents)** (testnet) — that link opens the **API wallets** panel directly — and connect the wallet that will own the trading account.
2. **Deposit to create the account.** There is no faucet — bring your own **Arbitrum Sepolia** testnet assets and deposit them from your main wallet. Your trading account is created on the **first deposit**.
3. **Create the API wallet.** Open the API wallets panel and choose *Create API wallet*. Your main wallet signs a single `approveAgent`, and the app returns a one-time **connection bundle** that contains the agent's private key.
4. **Save the bundle.** The private key is shown **once**. Copy the whole bundle and save it to a file — `bundle.json` is what the quickstart below loads.

A leaked API wallet key can trade your balance but can never withdraw or move funds off Native, so it is safe to run in a bot. Revoke or rotate it in the app any time.

{% hint style="info" %}
Exact panel labels in the app may differ from the names here. The web app's **API wallet** is the protocol's **agent**, and the web app's **Account** is the protocol's **owner** (your main wallet). The SDK uses the owner address only locally to resolve the agent — it never goes on the wire; the gateway recovers the signer from the signature.
{% endhint %}

## The connection bundle

The bundle is a small JSON object. Hand it to the SDK as a file path, a `dict`, or a JSON string.

```jsonc
{
  "network": "testnet",       // which gateway to use — must be "testnet"
  "accountAddress": "0x…",    // your main wallet (the owner the bot trades on)
  "agentPrivateKey": "0x…",   // the API wallet key the SDK signs with — the ONLY secret
  "agentEpoch": "10"          // optional; the SDK ignores it and re-resolves the epoch live
}
```

`agentPrivateKey` is the only value you must keep secret. `accountAddress` is used only locally to resolve the agent. `agentEpoch` is optional — the SDK re-resolves the live epoch on every construction, so a stale value in the bundle does no harm.

## Quickstart

Load the bundle, confirm the API wallet is approved, place a resting GTC bid well below the market so it does not fill, verify it rests, then cancel it. This runs on **testnet** against a funded account with a valid `bundle.json`.

```python
from decimal import Decimal

from native_core import Exchange, is_accepted, order_state

BUNDLE = "bundle.json"   # a file path, or the dict / JSON string itself
MARKET = "ETH/USDT"

exchange = Exchange.from_bundle(BUNDLE)   # picks the testnet gateway, loads the key, sets the owner
info = exchange.info                      # Exchange owns an internal Info

# Confirm the API wallet is approved on the owner before trading.
print(exchange.agent_info())              # {'approved': True, 'slot_id': 0, 'epoch': 10}

# Price a bid at half the reference so it rests instead of matching. Pass
# str/Decimal for price and size — NEVER float; the SDK never silently rounds.
book = info.l2_book(MARKET, depth=1)
reference = book["asks"] or book["bids"]  # whichever side has liquidity (books can be one-sided)
px = info.snap_price(MARKET, Decimal(reference[0]["price"]) / 2)  # -> str, snapped to the tick
sz = info.min_order_size(MARKET, px)                             # -> str, smallest valid size at px

# place() submits the order and polls info.order_status (via wait_for_open) until
# it rests. "accepted" alone means only admitted to the pipeline — NOT filled;
# place() resolves the real state for you and returns it.
order = exchange.place(MARKET, is_buy=True, sz=sz, limit_px=px, tif="gtc")
assert is_accepted(order["submission"])
print(order["cloid"], order["state"])     # 0x…  open

# The same read place() did for you, by hand: order_status by (user, market, cloid).
snapshot = info.order_status(user=exchange.effective_account, market=MARKET, cloid=order["cloid"])
print(order_state(snapshot))              # open

# Cancel by cloid, then confirm it left the book. A cancel gives the order a
# terminal state, so wait_for_order is safe here (a resting order has none).
cancel = exchange.cancel_by_cloid(MARKET, order["cloid"])
assert is_accepted(cancel)
final = info.wait_for_order(exchange.effective_account, MARKET, order["cloid"])
print(order_state(final))                 # cancelled
```

{% hint style="warning" %}
**Accepted is not filled, and an uncertain write is never resubmitted.** A raw `submission_status` of `accepted` means only that the order entered the pipeline — resolve the real outcome by `cloid`. If a write times out on the wire the SDK raises `SubmissionUncertain` (carrying `.cloid` and `.nonce`) or returns `submission_status: "timeout"`; **reconcile by cloid — never resubmit under a fresh nonce**, or the order may land twice. Only a `RateLimited` rejection is safe to resend. Use **one** `Exchange` per API wallet and share it across threads; two instances on the same key collide nonces.
{% endhint %}

The wire fields behind `POST /trade` and `POST /info`, transaction signing, and decimal/unit rules are documented in the API reference: [../post-trade.md](../post-trade.md), [../post-info.md](../post-info.md), [../transaction-signing.md](../transaction-signing.md), and [../decimals-units.md](../decimals-units.md).

## Next steps

{% content-ref url="core-concepts.md" %}
[core-concepts.md](core-concepts.md)
{% endcontent-ref %}

{% content-ref url="api-reference.md" %}
[api-reference.md](api-reference.md)
{% endcontent-ref %}

{% content-ref url="examples.md" %}
[examples.md](examples.md)
{% endcontent-ref %}
