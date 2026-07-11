---
description: Runnable scripts that ship with the Native Core Python SDK, plus two self-contained snippets.
---

# Examples

{% hint style="warning" %}
**Testnet only · pre-1.0.** The Native Core Python SDK is `v0.1.0` (alpha) and currently runs on **testnet only**. The API may change before 1.0; pin an exact version: `pip install native-core-python-sdk==0.1.0`.
{% endhint %}

The SDK ships a folder of runnable scripts under `examples/` in the source distribution (the `.tar.gz` on PyPI). They all read `examples/config.json` — a two-field file you copy from the template and fill in with your API wallet key:

```json
{ "secret_key": "0x<agentPrivateKey>", "account_address": "0x<accountAddress>" }
```

`secret_key` is the `agentPrivateKey` from your connection bundle; `account_address` is your main wallet (leave it blank to derive it from the key and trade in direct-owner mode). For how to mint the bundle and set this up, see [getting-started.md](getting-started.md).

```bash
cp examples/config.json.example examples/config.json   # then paste your key
```

{% hint style="info" %}
The read-only scripts under `examples/info/` place no orders, but they still read `secret_key` from `config.json` (the account address is derived from the key when `account_address` is blank), so a valid key must be present.
{% endhint %}

## Shipped scripts

| File | What it shows |
| --- | --- |
| `basic_order.py` | A resting `gtc` limit order end to end: place with a `cloid`, poll `order_status` until it rests (`open`), `cancel_by_cloid`, then `wait_for_order` to confirm the terminal `cancelled` state. |
| `basic_market_order.py` | A protected `market_order` with an explicit `protection_px` (the worst price you accept — this example passes it directly rather than deriving it from `slippage_bps`). Buys a minimum-notional clip 2% through the best ask, then market-sells the fill to flatten. |
| `basic_batch.py` | A mixed `batch` under one nonce: two resting bids in one call, then a second batch that `modify`s the first order and `cancel`s the second — atomically. Per-leg outcomes are reconciled with `batch_order_status`. |
| `info/query_markets_info.py` | Read-only: list the tradable markets and their precision (`markets()`). |
| `info/query_orderbook_info.py` | Read-only: the L2 order book for one market (`l2_book`). |
| `info/query_balances_info.py` | Read-only: your spot balances, available and locked per asset (`user_balances`). |
| `info/query_open_order_info.py` | Read-only: your resting orders in one market (`open_orders`). |

The trading scripts default to testnet and `ETH/USDT` — edit the `MARKET` constant to trade a different market. Run them from the repo root:

```bash
python examples/info/query_markets_info.py     # read-only tour, no orders
python examples/basic_order.py                 # places a real testnet order
python examples/basic_market_order.py
python examples/basic_batch.py
```

{% hint style="info" %}
`examples/market_maker_bot.py` is an illustrative cancel-and-replace market-making loop — it re-quotes both sides as post-only (`alo`) orders every few seconds and caps inventory. It is **excluded from the published PyPI package**, so `pip install` does not ship it; clone the repo from the [Native GitHub org](https://github.com/Native-org) to run it.
{% endhint %}

## Read markets and the order book

Self-contained read-only script. `Info` wraps [`POST /info`](../post-info.md); reads need only the gateway, so no order is ever signed here. See [Decimal Units](../decimals-units.md) for what `price_decimals` and `base_quantity_decimals` mean.

```python
from native_core import Info

BUNDLE = "bundle.json"   # your saved connection bundle (see getting-started.md)
MARKET = "ETH/USDT"

info = Info.from_bundle(BUNDLE)   # picks the testnet gateway from the bundle's network

# List the tradable markets and their precision.
for m in info.markets()["markets"]:
    symbol = f"{m['base_symbol']}/{m['quote_symbol']}"
    print(f"{m['market_id']:>4}  {symbol:<12}  price {m['price_decimals']}dp  size {m['base_quantity_decimals']}dp")

# Top of the L2 book for one market (up to 100 levels).
book = info.l2_book(MARKET, depth=5)
for level in reversed(book["asks"]):
    print(f"ask  {level['price']:>12}  {level['quantity']}")
for level in book["bids"]:
    print(f"bid  {level['price']:>12}  {level['quantity']}")
```

## Place a resting limit order and cancel it

Self-contained write script. `Exchange` wraps [`POST /trade`](../post-trade.md) and owns an internal `Info` as `exchange.info`. Use **one** `Exchange` per API wallet and share it across threads — the nonce is a per-instance, lock-guarded counter. Pass `sz` and `limit_px` as `str` or `Decimal`, never `float`.

```python
from decimal import Decimal

from native_core import Exchange, is_accepted, order_state

BUNDLE = "bundle.json"
MARKET = "ETH/USDT"

exchange = Exchange.from_bundle(BUNDLE)   # picks the gateway, loads the key, sets owner
info = exchange.info

# Price a bid well below the market so it rests instead of filling.
book = info.l2_book(MARKET, depth=1)
reference = book["asks"] or book["bids"]                     # whichever side has liquidity
px = info.snap_price(MARKET, Decimal(reference[0]["price"]) / 2)
sz = info.min_order_size(MARKET, px)

# place() submits the order and waits for it to rest; it returns the handle and state.
order = exchange.place(MARKET, is_buy=True, sz=sz, limit_px=px, tif="gtc")
print(order["cloid"], order["state"])                        # 0x…  open

# Cancel by cloid, then confirm it left the book.
cancel = exchange.cancel_by_cloid(MARKET, order["cloid"])
assert is_accepted(cancel)
final = info.wait_for_order(exchange.effective_account, MARKET, order["cloid"])
print(order_state(final))                                    # cancelled
```

{% hint style="warning" %}
`accepted` is not `filled`. A raw `order()` / `market_order()` returning `submission_status: "accepted"` means only that the transaction entered the pipeline — resolve the real outcome by `cloid` (`wait_for_open` for a resting order, `wait_for_order` for a terminal one, or `info.reconcile_by_cloid(...)`). If a write times out on the wire the SDK raises `SubmissionUncertain` (carrying `.cloid` and `.nonce`); reconcile by `cloid` and **never** resubmit under a fresh nonce — that risks a double-fill. `place()` runs the submit and the matching wait for you.
{% endhint %}

## See also

* [getting-started.md](getting-started.md) — mint an API wallet, save the bundle, and run your first trade.
* [PyPI project](https://pypi.org/project/native-core-python-sdk/) — `pip install native-core-python-sdk==0.1.0`.
* [Native GitHub org](https://github.com/Native-org) — clone the SDK repo for the full `examples/` folder, including `market_maker_bot.py`.
* Wire references: [`POST /trade`](../post-trade.md), [`POST /info`](../post-info.md), [Transaction signing](../transaction-signing.md), [Decimal Units](../decimals-units.md).
