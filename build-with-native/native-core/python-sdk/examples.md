---
description: Runnable scripts that ship with the Native Core Python SDK, plus two self-contained snippets.
---

# Examples

The SDK ships a folder of runnable scripts under `examples/` in the source distribution (the `.tar.gz` on PyPI). All of them except `ws_feeds.py` read `examples/config.json` — a two-field file you copy from the template and fill in with your API wallet key:

```json
{ "secret_key": "0x<agentPrivateKey>", "account_address": "0x<accountAddress>" }
```

`secret_key` is the `agentPrivateKey` from your connection bundle; `account_address` is your main wallet (leave it blank to derive it from the key and trade in direct-owner mode). For how to create the API wallet and set this up, see [Getting Started](getting-started.md).

```bash
cp examples/config.json.example examples/config.json   # then paste your key
```

{% hint style="info" %}
The read-only scripts under `examples/info/` place no orders, but they still read `secret_key` from `config.json` (the account address is derived from the key when `account_address` is blank), so a valid key must be present. `ws_feeds.py` is the exception: it needs no key and takes an optional endpoint URL as its argument instead.
{% endhint %}

## Get the examples

`pip install` resolves the wheel, which contains only the `native_core` package — no `examples/`. The scripts ship in the **source distribution**, so take the `.tar.gz`:

```bash
pip download --no-binary :all: native-core-python-sdk==2.0.0
tar xzf native_core_python_sdk-2.0.0.tar.gz
cd native_core_python_sdk-2.0.0
cp examples/config.json.example examples/config.json   # paste your key (see above)
python examples/basic_order.py                          # run from the package root
```

## Preflight: `setup()`

Every shipped script opens with the same call — run it first:

```python
from example_utils import setup

address, info, exchange = setup()   # reads examples/config.json
```

`setup()` reads `examples/config.json`, builds the API wallet from `secret_key`, resolves agent-vs-owner (a blank `account_address` derives the address from the key and trades in **direct-owner** mode; a filled one signs as an **agent** on that owner), prints the effective account and gateway, then — before returning — reads `user_balances` for the **owner** and **hard-fails** if the account holds nothing:

```
connected
  account (owner) : 0x1234…
  api wallet      : 0xabcd…  (agent mode)
  gateway         : https://api-test.native.org
```

It turns the two misconfigs that would otherwise surface as empty query results or opaque rejections — a **wrong owner** address, or an **unfunded account** — into one clear up-front error (`RuntimeError: account 0x… has no balances … fund it before running`). Pass `require_balance=False` to skip that gate. Read-only scripts call `read_only()` instead: same config, but no balance gate and no signing wallet, so it runs even when the key is not yet an approved agent.

`example_utils` also exports **`write_succeeded(result)`**, which is what every trading script gates on:

```python
if not example_utils.write_succeeded(result):   # is_accepted AND not is_order_failed
    raise SystemExit(...)
```

Copy that check, not a bare `is_accepted` — see [Accepted is not placed](core-concepts.md#accepted-is-not-placed).

## Shipped scripts

| File | What it shows |
| --- | --- |
| `basic_order.py` | A resting `gtc` limit order end to end: place with a `cloid`, take the oid straight off the `/trade` response with `order_oid`, read `order_status` once to see it rest (`open`), `cancel_by_cloid`, then `wait_for_order` for the terminal `cancelled` state. |
| `basic_market_order.py` | A protected `market_order` with an explicit `protection_px` (the worst price you accept — this example passes it directly rather than deriving it from `slippage_bps`). Buys a minimum-notional clip 2% through the best ask, then market-sells the fill to flatten. |
| `basic_batch.py` | A mixed `batch` under one nonce: two resting bids in one call, then a second batch that `modify`s the first order and `cancel`s the second — atomically. Every per-leg outcome is read off the batch response with `batch_legs`, so the script spends **no** `/info` reads. |
| `ws_feeds.py` | Streams the book, top of book, trades and mids over the [WebSocket](streaming.md). Needs no key; takes an optional endpoint URL. |
| `info/query_markets_info.py` | Read-only: list the tradable markets and their precision (`markets()`). |
| `info/query_orderbook_info.py` | Read-only: the L2 order book for one market (`l2_book`). |
| `info/query_balances_info.py` | Read-only: your spot balances, available and locked per asset (`user_balances`). |
| `info/query_open_order_info.py` | Read-only: your resting orders in one market (`open_orders`). |

The trading scripts default to testnet (the base-url default in `example_utils.setup()`) and to `ETH/USDT`. Pass `setup(base_url=constants.MAINNET_API_URL)` to run on mainnet, or edit the `MARKET` constant for another market. Run them from the package root:

```bash
python examples/info/query_markets_info.py     # read-only tour, no orders
python examples/ws_feeds.py                    # streaming tour, no key needed
python examples/basic_order.py                 # places a real testnet order
python examples/basic_market_order.py
python examples/basic_batch.py
```

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

# Top of the L2 book for one market (up to 100 levels). found=false omits
# bids/asks entirely, so check it before indexing.
book = info.l2_book(MARKET, depth=5)
if not book.get("found"):
    raise SystemExit("no book published for this market yet")
for level in reversed(book.get("asks") or []):
    print(f"ask  {level['price']:>12}  {level['quantity']}")
for level in book.get("bids") or []:
    print(f"bid  {level['price']:>12}  {level['quantity']}")
```

## Place a resting limit order and cancel it

Self-contained write script. `Exchange` wraps [`POST /trade`](../post-trade.md) and owns an internal `Info` as `exchange.info`. Use **one** `Exchange` per API wallet and share it across threads — the nonce is a per-instance, lock-guarded counter. Pass `sz` and `limit_px` as `str` or `Decimal`, never `float`.

```python
from decimal import Decimal

from native_core import Exchange, is_accepted, is_order_failed, leaf_error_code, order_state

BUNDLE = "bundle.json"
MARKET = "ETH/USDT"

exchange = Exchange.from_bundle(BUNDLE)   # picks the endpoint, loads the key, sets owner
info = exchange.info

# Price a bid well below the market so it rests instead of filling.
book = info.l2_book(MARKET, depth=1)
if not book.get("found"):                                    # found=false omits bids/asks
    raise SystemExit("no book published for this market yet")
reference = book.get("asks") or book.get("bids")             # whichever side has liquidity
if not reference:
    raise SystemExit("book is empty on both sides")
px = info.snap_price(MARKET, Decimal(reference[0]["price"]) / 2)
sz = info.min_order_size(MARKET, px)

# place() submits and waits for the order to rest, unless the response already
# settled it. Returns {cloid, submission, status, state, oid}.
order = exchange.place(MARKET, is_buy=True, sz=sz, limit_px=px, tif="gtc")
if is_order_failed(order["submission"]):                     # accepted != placed
    raise SystemExit(f"order failed: {leaf_error_code(order['submission'])}")
print(order["cloid"], order["oid"], order["state"])          # 0x…  1234  open

# Cancel by cloid, then confirm it left the book. A cancel against an order that is
# already gone is accepted too, with a "missingorder" leaf.
cancel = exchange.cancel_by_cloid(MARKET, order["cloid"])
assert is_accepted(cancel) and not is_order_failed(cancel)
final = info.wait_for_order(exchange.effective_account, MARKET, order["cloid"])
print(order_state(final))                                    # cancelled
```

{% hint style="warning" %}
`accepted` is not `placed`. `submission_status: "accepted"` is a verdict on the **transaction**; your order can still have failed inside it, on a leaf of the `response` envelope. Check `is_order_failed(resp)` and read `leaf_error_code(resp)`, and take the `oid` and fill from `order_oid` / `fill` rather than an `/info` read. A failed order is never written to `/info`, so reconciling its `cloid` can only time out.

If a write times out on the wire the SDK raises `SubmissionUncertain` (carrying `.cloids` and `.nonce`): reconcile by `cloid` and **never** resubmit — that risks a double-fill. A returned `submission_status: "timeout"` follows the same rule, except when `is_safe_to_resend(resp)` is `True` (the `Handoff*` family, which never reached a node), where you back off by `retry_after_ms` and resend the same `cloid`.
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
        ref = (book.get("asks") or book.get("bids")) if book.get("found") else None
        if not ref:
            time.sleep(REFRESH)                                      # no book yet — wait it out
            continue
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

{% hint style="info" %}
A loop like this one re-reads the book every pass, and `/info` allows about **one read per second per client IP**. A bot that runs continuously should take the book and its order updates from the [WebSocket](streaming.md) instead — `info.ws()` gives you `l2Book`, `bbo` and `orderUpdates` pushed for free against the read budget — and keep HTTP reads for reconciliation. `examples/ws_feeds.py` is the runnable tour.
{% endhint %}

## See also

* [Getting Started](getting-started.md) — create an API wallet, save the bundle, and run your first trade.
* [Streaming](streaming.md) — take feeds off the WebSocket instead of polling.
* [PyPI project](https://pypi.org/project/native-core-python-sdk/) — `pip install native-core-python-sdk==2.0.0`.
* Wire references: [`POST /trade`](../post-trade.md), [`POST /info`](../post-info.md), [Transaction signing](../transaction-signing.md), [Decimals & Units](../decimals-units.md).
