---
description: Runnable scripts that ship with the Native Core Python SDK, plus two self-contained snippets.
---

# Examples

The SDK ships a folder of runnable scripts under `examples/` in the source distribution (the `.tar.gz` on PyPI). They all read `examples/config.json` — a two-field file you copy from the template and fill in with your API wallet key:

```json
{ "secret_key": "0x<agentPrivateKey>", "account_address": "0x<accountAddress>" }
```

`secret_key` is the `agentPrivateKey` from your connection bundle; `account_address` is your main wallet (leave it blank to derive it from the key and trade in direct-owner mode). For how to create the API wallet and set this up, see [getting-started.md](getting-started.md).

```bash
cp examples/config.json.example examples/config.json   # then paste your key
```

{% hint style="info" %}
The read-only scripts under `examples/info/` place no orders, but they still read `secret_key` from `config.json` (the account address is derived from the key when `account_address` is blank), so a valid key must be present.
{% endhint %}

## Get the examples

The runnable scripts live in `examples/`. `pip install native-core-python-sdk==1.0.0` ships most of them inside the source distribution, but the complete folder — including `market_maker_bot.py` — lives only in the repo, which is excluded from the published PyPI package. Clone it from the [Native GitHub org](https://github.com/Native-org) to get everything, set up the config as above, and run any script from the repo root:

```bash
git clone https://github.com/Native-org/native-core-python-sdk.git
cd native-core-python-sdk
cp examples/config.json.example examples/config.json   # paste your key (see above)
python examples/basic_order.py                          # runs from the repo root
```

## Preflight: `setup()`

Every shipped script opens with the same call — run it first:

```python
from example_utils import setup

address, info, exchange = setup()   # reads examples/config.json
```

`setup()` reads `examples/config.json`, builds the API wallet from `secret_key`, resolves agent-vs-owner (a blank `account_address` derives the address from the key and trades in **direct-owner** mode; a filled one signs as an **agent** on that owner), prints the effective account and endpoint, then — before returning — reads `user_balances` for the **owner** and **hard-fails** if the account holds nothing:

```
connected
  account (owner) : 0x1234…
  api wallet      : 0xabcd…  (agent mode)
  endpoint        : https://api-test.native.org
```

It turns the two misconfigs that would otherwise surface as empty query results or opaque rejections — a **wrong owner** address, or an **unfunded account** — into one clear up-front error (`RuntimeError: account 0x… has no balances … fund it before running`). Read-only scripts call `read_only()` instead: same config, but no balance gate and no signing wallet, so it runs even when the key is not yet an approved agent.

## Shipped scripts

| File | What it shows |
| --- | --- |
| `basic_order.py` | A resting `gtc` limit order end to end: place with a `cloid`, poll `order_status` until it rests (`open`), `cancel_by_cloid`, then `wait_for_order` to confirm the terminal `cancelled` state. |
| `basic_market_order.py` | A protected `market_order` with an explicit `protection_px` (the worst price you accept — this example passes it directly rather than deriving it from `slippage_bps`). Buys a minimum-notional clip 2% through the best ask, then market-sells the fill to flatten. |
| `basic_batch.py` | A mixed `batch` under one nonce: two resting bids in one call, then a second batch that `modify`s the first order and `cancel`s the second — atomically. Per-leg outcomes are reconciled with one `order_status` lookup per leg. |
| `info/query_markets_info.py` | Read-only: list the tradable markets and their precision (`markets()`). |
| `info/query_orderbook_info.py` | Read-only: the L2 order book for one market (`l2_book`). |
| `info/query_balances_info.py` | Read-only: your spot balances, available and locked per asset (`user_balances`). |
| `info/query_open_order_info.py` | Read-only: your resting orders in one market (`open_orders`). |

The trading scripts default to testnet (the base-url default in `example_utils.setup()`) and to `ETH/USDT`. Pass `setup(base_url=constants.MAINNET_API_URL)` to run on mainnet, or edit the `MARKET` constant for another market. Run them from the repo root:

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

Self-contained read-only script. `Info` wraps [`POST /info`](../post-info.md); reads need only the endpoint, so no order is ever signed here. See [Decimals & Units](../decimals-units.md) for what `price_decimals` and `base_quantity_decimals` mean.

```python
from native_core import Info

BUNDLE = "bundle.json"   # your saved connection bundle (see getting-started.md)
MARKET = "ETH/USDT"

info = Info.from_bundle(BUNDLE)   # picks the endpoint from the bundle's network

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

exchange = Exchange.from_bundle(BUNDLE)   # picks the endpoint, loads the key, sets owner
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
`accepted` is not `filled`. A raw `order()` / `market_order()` returning `submission_status: "accepted"` means the transaction **landed and executed** — not that the order rested or filled; read the real state by `cloid` (`wait_for_open` for a resting order, `wait_for_order` for a terminal one, or `info.reconcile_by_cloid(...)`). If a write times out on the wire the SDK raises `SubmissionUncertain` (carrying `.cloid` and `.nonce`); reconcile by `cloid` and **never** resubmit under a fresh nonce — that risks a double-fill. `place()` runs the submit and the matching wait for you.
{% endhint %}

## Rounding & precision

A price or size from your own model rarely lands on the market's grid. Snap both to what the market accepts **before** signing: `info.snap_price(market, price)` rounds to the market's `price_decimals` / `max_price_sig_figs`, and `info.min_order_size(market, px)` returns the smallest size that clears the quote asset's minimum notional at that price. Both return wire-ready strings; **floats are rejected** — pass `str` or `Decimal` or the SDK raises `LocalValidationError` at validation rather than round you down silently.

```python
from decimal import Decimal

from native_core import Exchange

exchange = Exchange.from_bundle("bundle.json")
info = exchange.info
MARKET = "ETH/USDT"

raw_price = 3512.987654        # a float off your model — too many sig-figs for the grid

px = info.snap_price(MARKET, Decimal(str(raw_price)))   # -> str, snapped to price_decimals / max_price_sig_figs
sz = info.min_order_size(MARKET, px)                    # -> str, smallest size clearing min-notional at px

exchange.place(MARKET, is_buy=True, sz=sz, limit_px=px, tif="gtc")
# Passing raw_price straight in (a float, or a too-precise str) raises LocalValidationError.
```

`snap_price` takes a `rounding` argument (`ROUND_DOWN` by default; pass `ROUND_UP` for an ask so it never snaps below your target), and `min_order_size` a `margin` (default `"1.1"`) for headroom over the bare minimum. See [Decimals & Units](../decimals-units.md) for the raw/atom conversion that makes floats unsafe, and [notation](../notation.md) for the field-level number rules.

## Cancel on disconnect (dead-man's switch)

There is **no server-side scheduled cancel**: if your process dies, its resting orders stay on the book until they fill or you cancel them. Build the dead-man's switch client-side — flatten everything on the way out, on every exit path. `exchange.cancel_open()` finds where the effective account actually rests (one read) and issues a `cancelAll` on each of those markets; cancels are always admitted, even for a frozen account.

```python
import signal
import time
from decimal import Decimal

from native_core import Exchange

BUNDLE = "bundle.json"
MARKET = "ETH/USDT"
REFRESH = 5.0             # re-quote cadence, seconds
HEARTBEAT_TIMEOUT = 30.0  # a round slower than this = degraded endpoint -> flatten and stop

exchange = Exchange.from_bundle(BUNDLE)
info = exchange.info


def flatten() -> None:
    # Cancel every resting order for the owner, across markets — this is the switch.
    for market_id, resp in exchange.cancel_open().items():
        print("flattened", market_id, resp["submission_status"])


def _stop(*_):
    raise KeyboardInterrupt   # SIGTERM (docker stop / k8s / `kill`) -> unwind into finally

signal.signal(signal.SIGTERM, _stop)

try:
    while True:
        started = time.monotonic()
        book = info.l2_book(MARKET, depth=1)
        ref = book["asks"] or book["bids"]
        px = info.snap_price(MARKET, Decimal(ref[0]["price"]) / 2)   # rests well below the market
        sz = info.min_order_size(MARKET, px)

        exchange.cancel_open([MARKET])                               # drop last round's quote
        exchange.place(MARKET, is_buy=True, sz=sz, limit_px=px, tif="gtc")

        if time.monotonic() - started > HEARTBEAT_TIMEOUT:
            raise TimeoutError("round overran the heartbeat")        # endpoint degraded -> bail
        time.sleep(REFRESH)
except (KeyboardInterrupt, Exception) as exc:                        # Ctrl-C, SIGTERM, heartbeat, or any bug
    print(f"stopping: {exc!r}")
finally:
    flatten()   # runs on every exit path — crash, signal, or clean stop
```

`market_maker_bot.py` ships this pattern in full: it re-quotes as post-only (`alo`) orders and, in its `finally`, calls `cancel_all` and reconciles per handle so nothing is left resting after the process exits.

## See also

* [getting-started.md](getting-started.md) — create an API wallet, save the bundle, and run your first trade.
* [PyPI project](https://pypi.org/project/native-core-python-sdk/) — `pip install native-core-python-sdk==1.0.0`.
* [Native GitHub org](https://github.com/Native-org) — clone the SDK repo for the full `examples/` folder, including `market_maker_bot.py`.
* Wire references: [`POST /trade`](../post-trade.md), [`POST /info`](../post-info.md), [Transaction signing](../transaction-signing.md), [Decimals & Units](../decimals-units.md).
