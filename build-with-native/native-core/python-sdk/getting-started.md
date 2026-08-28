---
description: Install the Native Core Python SDK, create an API wallet, and place your first order.
---

# Getting Started

The Native Core Python SDK (`native-core-python-sdk`, import `native_core`) is a thin, typed, synchronous client over Native Core's two REST endpoints — `POST /info` for reads, `POST /trade` for writes — plus the [WebSocket](../websocket.md) for live data. This page takes you from `pip install` to a resting order you place and cancel yourself.

## Install

Requires **Python 3.10+**. The runtime dependencies are `requests`, `eth-account`, `eth-utils`, and `websocket-client` (no `web3`, no `pydantic`).

```bash
pip install native-core-python-sdk==2.0.0
```

This page documents **2.0.0**. If you are upgrading from 1.x, read [Accepted is not placed](core-concepts.md#accepted-is-not-placed) first: `submission_status: "accepted"` no longer means your order succeeded.

## Get an API wallet

The SDK trades with an **API wallet**: a protocol-level *agent* key scoped only to placing and cancelling orders. It can never move funds — deposits, withdrawals, and the approval itself are signed by your main wallet in the web app, not by the SDK. You create the API wallet once, in the Native web app:

1. **Connect your main wallet.** Open the **[Native web app](https://app.native.org/markets/ETH-USDT?agentWallets=agents)** — that link opens the **API wallets** panel directly — and connect the wallet that will own the trading account.
2. **Deposit to create the account.** Deposit a supported asset from your main wallet. Your trading account is created on the **first deposit**.
3. **Create the API wallet.** Open the API wallets panel and choose *Create API wallet*. Your main wallet signs a single `approveAgent`, and the app returns a one-time **connection bundle** that contains the agent's private key.
4. **Save the bundle.** The private key is shown **once**. Copy the whole bundle and save it to a file — `bundle.json` is what the quickstart below loads.

A leaked API wallet key can trade your balance but can never withdraw or move funds off Native, so it is safe to run in a bot. Revoke or rotate it in the app any time.

{% hint style="info" %}
Exact panel labels in the app may differ from the names here. The web app's **API wallet** is the protocol's **agent**, and the web app's **Account** is the protocol's **owner** (your main wallet). The SDK uses the owner address only locally to resolve the agent — it never goes on the wire; the API recovers the signer from the signature.
{% endhint %}

## The connection bundle

The bundle is a small JSON object. Hand it to the SDK as a file path, a `dict`, or a JSON string.

```jsonc
{
  "network": "mainnet",       // which network to use — "mainnet" or "testnet"
  "accountAddress": "0x…",    // your main wallet (the owner the bot trades on)
  "agentPrivateKey": "0x…",   // the API wallet key the SDK signs with — the ONLY secret
  "agentEpoch": "10"          // optional; the SDK ignores it and re-resolves the epoch live
}
```

`agentPrivateKey` is the only value you must keep secret. `accountAddress` is used only locally to resolve the agent. `agentEpoch` is optional — the SDK re-resolves the live epoch on every construction, so a stale value in the bundle does no harm.

## Quickstart

Load the bundle, confirm the API wallet is approved, place a resting GTC bid well below the market so it does not fill, verify it rests, then cancel it. This runs on **mainnet** against a funded account with a valid `bundle.json`.

```python
from decimal import Decimal

from native_core import Exchange, is_accepted, is_order_failed, leaf_error_code, order_state

BUNDLE = "bundle.json"   # a file path, or the dict / JSON string itself
MARKET = "ETH/USDT"

exchange = Exchange.from_bundle(BUNDLE)   # picks the endpoint from the bundle, loads the key, sets the owner
info = exchange.info                      # Exchange owns an internal Info

# Confirm the API wallet is approved on the owner before trading.
print(exchange.agent_info())              # {'approved': True, 'slot_id': 0, 'epoch': 10}

# Price a bid at half the reference so it rests instead of matching. Pass
# str/Decimal for price and size — NEVER float; the SDK never silently rounds.
book = info.l2_book(MARKET, depth=1)
if not book.get("found"):                 # found=false omits bids/asks entirely
    raise SystemExit("no book published for this market yet")
reference = book.get("asks") or book.get("bids")   # whichever side has liquidity
px = info.snap_price(MARKET, Decimal(reference[0]["price"]) / 2)  # -> str, snapped to the tick
sz = info.min_order_size(MARKET, px)                             # -> str, smallest valid size at px

# place() submits, takes the oid off the write itself, and reads order_status
# (via wait_for_open for gtc/alo) only when the response has not already settled
# the order. Returns {cloid, submission, status, state, oid}.
order = exchange.place(MARKET, is_buy=True, sz=sz, limit_px=px, tif="gtc")

# "accepted" is a verdict on the TRANSACTION, not on your order. A tick, lotsize
# or balance failure comes back accepted with the reason on a leaf — check it.
assert is_accepted(order["submission"])
if is_order_failed(order["submission"]):
    raise SystemExit(f"order failed: {leaf_error_code(order['submission'])}")

print(order["cloid"], order["oid"], order["state"])   # 0x…  1234  open

# Read the resting state by hand: order_status by (user, market, cloid).
snapshot = info.order_status(user=exchange.effective_account, market=MARKET, cloid=order["cloid"])
print(order_state(snapshot))              # open

# Cancel by cloid. A cancel whose target is already gone is ALSO accepted, with a
# "missingorder" leaf, so check the order and not just the submission status.
cancel = exchange.cancel_by_cloid(MARKET, order["cloid"])
assert is_accepted(cancel) and not is_order_failed(cancel)

# Confirm it left the book. A cancel gives the order a terminal state, so
# wait_for_order is right here (a resting order has none).
final = info.wait_for_order(exchange.effective_account, MARKET, order["cloid"])
print(order_state(final))                 # cancelled
```

{% hint style="warning" %}
**Accepted is not placed.** `submission_status: "accepted"` means the **transaction** landed and executed. Your order can still have failed inside it — `tick`, `lotsize`, `badalopx`, `insufficientspotbalance`, `mintradespotntl`, `missingorder` all arrive that way, on a leaf of the `response` envelope. Read `is_order_failed` / `leaf_error_code`, or branch on `next_action`. A failed order is never written to `/info` at all, so polling or reconciling its `cloid` can only time out.

**An uncertain write is not resubmitted.** If a write times out the SDK raises `SubmissionUncertain` (carrying `.cloid` and `.nonce`) or returns `submission_status: "timeout"`: reconcile by `cloid` and never resubmit, or the order may land twice. The one exception is a timeout where `is_safe_to_resend(resp)` is true — the `Handoff*` family never reached a node, so back off by `retry_after_ms` and send the same `cloid` again. A `RateLimited` rejection is likewise safe to resend.

Use **one** `Exchange` per API wallet and share it across threads; two instances on the same key collide nonces.
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
