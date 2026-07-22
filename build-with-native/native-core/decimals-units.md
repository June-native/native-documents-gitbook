---
description: How Native Core converts human decimal strings to the integer atoms it executes on.
---

# Decimals & Units

Native Core executes on integers only. Public order and modify payloads accept human decimal strings for `price` and `quantity`; the write path converts them to the raw integer atoms encoded in the signed transaction bytes. Integer-only fields such as ids, timestamps, nonces, and accounting amounts remain decimal integer strings. Public query responses use asset and market decimal metadata to return human-readable display strings where that is useful.

## Decimal units

Each asset has one balance decimal field:

* `balance_decimals`: internal balance and settlement atom scale.

Balances use `balance_decimals`:

```
raw_balance_atoms = display_asset_amount * 10^balance_decimals
display_asset_amount = raw_balance_atoms / 10^balance_decimals
```

Order quantities use the market's `base_quantity_decimals`:

```
raw_quantity = display_quantity * 10^market.base_quantity_decimals
display_quantity = raw_quantity / 10^market.base_quantity_decimals
```

For example, ETH has `balance_decimals = 8`, while ETH/USDC uses `base_quantity_decimals = 4`:

```
1 ETH      = 100000000 balance atoms
0.0001 ETH = 1 order quantity atom on ETH/USDC
```

USDC has `balance_decimals = 8`, while USDC/USDT uses `base_quantity_decimals = 2`:

```
1 USDC balance      = 100000000 balance atoms
0.01 USDC order qty = 1 order quantity atom on USDC/USDT
```

`userBalances` returns display amounts for `available` and `locked` using the asset's `balance_decimals`. `spotCreditPositions` returns both raw signed balance atom strings (`pending_exposure_qty`, `actual_qty`) and display strings (`pending_exposure_display`, `actual_display`) using `balance_decimals` when asset metadata is available.

Asset metadata validation:

* `asset_id` is a `u32`.
* `symbol` is 1 to 16 ASCII alphanumeric characters. Display casing is preserved.
* `symbol` must be unique under ASCII case-insensitive matching (`cbBTC`, `CBBTC`, and `cbbtc` collide).
* `asset_id` must be unique.
* `balance_decimals` must be `0..=18`.

## Market decimals

Each market has a base asset and quote asset. Public `quantity` is a display base amount and public `price` is a display quote-per-base price. The write path uses the market's `base_quantity_decimals` and `price_decimals` to convert those display strings into raw integer atoms before assembling signed bytes.

The market response includes:

* `base_quantity_decimals`: market-level order quantity lot scale for the base asset.
* `quote_balance_decimals`: `balance_decimals` of the quote asset.
* `price_decimals`: decimal places for raw prices.
* `max_price_sig_figs`: max significant figures for non-integer display prices, always in `1..=18`.

`base_quantity_decimals`, `price_decimals`, and `max_price_sig_figs` are canonical market metadata. `quote_balance_decimals` is returned on each market so clients can convert quote notional without separately joining against the `assets` response.

Display price means quote asset units per 1 base asset:

```
display_price = raw_price / 10^price_decimals
raw_price = display_price * 10^price_decimals
```

Markets must satisfy:

```
base_quantity_decimals <= base.balance_decimals
price_decimals + base_quantity_decimals <= quote.balance_decimals
```

Quote notional atoms are:

```
quote_atoms =
  raw_price * raw_quantity * 10^(quote.balance.decimals - price_decimals - base_quantity_decimals)
```

For ETH/USDC:

```
ETH/USDC base_quantity_decimals = 4
USDC balance_decimals = 8
price_decimals = 2

display price 3500 USDC / ETH => raw_price = 3500 * 10^2 = 350000
display qty   1 ETH           => raw_quantity = 1 * 10^4 = 10000
raw notional quote atoms      => 350000 * 10000 * 10^2 = 350000000000
display notional              => 350000000000 / 10^8 = 3500 USDC
```

`l2Book`, `openOrders`, `userFills`, and `orderStatus` return display `price` and display quantity fields. Write actions use the same display convention for `price` and `quantity`; the canonical signed payload still contains raw atoms.

Market metadata validation:

* `market_id` is a `u32`.
* `base_asset_id` and `quote_asset_id` must exist.
* `base_asset_id != quote_asset_id`.
* `quote_asset_id` must be enabled in canonical `protocol.quote_assets`.
* `base_quantity_decimals <= base.balance_decimals`.
* `price_decimals + base_quantity_decimals <= quote.balance_decimals`.
* `max_price_sig_figs` must be in `1..=18`.
* A base/quote pair may be opened only once.
* Market ids are unique.

The genesis quote allowlist is USDC asset `1` with `min_quantity="10"`, USDT asset `2` with `min_quantity="10"`, ETH asset `3` with `min_quantity="0.01"`, and BNB asset `4` with `min_quantity="0.02"`. Read the live per-quote-asset minimums from [`POST /info` `quoteAssets`](post-info.md#quoteassets).

## Write-time numeric validation

`POST /trade` validates signed action numbers before reconstructing the canonical signed payload:

* Integer fields accept a base-10 integer string or an unsigned JSON integer (no decimal point, no sign, no exponent notation, no commas); the string form is preferred for values above 2^53.
* Order and modify `price` / `quantity` are human decimal strings: no sign, no exponent notation, no commas, and no more fractional digits than the market's `price_decimals` / `base_quantity_decimals`. A value that overflows `u64` after conversion is rejected with `invalid_price_overflow` / `invalid_quantity_overflow`.
* `market_id` parses as `u32`.
* `oid`, `nonce`, `agent_epoch`, and `expires_after_ms` parse as `u64`.
* `quantity` greater than zero is enforced at **execution**, not at this admission stage — a zero quantity passes shaping and is rejected when the order runs.
* `price` must be present for both limit orders and protected market orders in the current protocol encoding.
* `market` orders are valid only with `ioc` or `fok`.
* `gtc` and `alo` are valid only for limit orders.
* non-integer display prices may have at most `max_price_sig_figs` significant figures (enforced at execution, not admission — see [Valid / invalid examples](#valid-invalid-examples)); integer display prices are always allowed.
* quote notional is computed with widened arithmetic and must fit into `u64` quote balance atoms.
* If the configured minimum spot notional is nonzero, the raw quote notional must be at least that minimum.

## Valid / invalid examples

The pairs below use an **illustrative** market with `price_decimals = 2`, `max_price_sig_figs = 5`, and `base_quantity_decimals = 4`. These are not any specific market's real values — read each market's precision from [`POST /info` `markets`](post-info.md#markets). Send the display string exactly; nothing rounds on the wire.

Two independent gates apply to a `price`, and they fail at **different layers**:

* **Fractional digits `>` `price_decimals`** — rejected at **admission**: `POST /trade` returns `submission_status: "rejected"` with `invalid_price_precision` (or `invalid_quantity_precision` for `quantity`).
* **Non-integer price with more than `max_price_sig_figs` significant figures** — clears admission, then **fails at execution** as [`tick`](error-responses.md#execution-level-failures). Because `/trade` is synchronous, that failure comes back on the `/trade` response itself (`submission_status: "rejected"`, `error.code: "tick"`), and is also readable afterward on [`orderStatus`](post-info.md#orderstatus). The [Python SDK](python-sdk/README.md) saves you the round trip by rejecting it locally (`LocalValidationError: … significant figures`) before it ever signs.

`price`:

| Value | Result | Why |
| ------------ | ------------------------------------- | ------------------------------------------------------------------------------------------------- |
| `3500.00`    | valid                                 | raw `350000` is an integer price; the significant-figures gate is skipped.                         |
| `3500.1`     | valid                                 | 1 fractional digit `<=` `price_decimals`; 5 significant figures — exactly at `max_price_sig_figs`. |
| `3500.12`    | admitted, then rejected `tick`        | 2 fractional digits clear admission, but 6 significant figures `>` `max_price_sig_figs` (5); returned on `/trade` as `submission_status: "rejected"`, `error.code: "tick"`. The SDK rejects it locally first. |
| `3500.123`   | rejected `invalid_price_precision`    | 3 fractional digits `>` `price_decimals` (2) — rejected at admission.                              |
| `3500.120`   | rejected `invalid_price_precision`    | trailing zeros count — still 3 fractional digits `>` 2.                                            |

`quantity` — only the fractional-digit gate applies; `quantity` has no significant-figures cap:

| Value     | Result                                | Why                                                        |
| --------- | ------------------------------------- | ---------------------------------------------------------- |
| `1.0000`  | valid                                 | 4 fractional digits `<=` `base_quantity_decimals`.         |
| `0.0001`  | valid                                 | smallest lot — 1 order quantity atom.                      |
| `1.2345`  | valid                                 | 4 fractional digits.                                       |
| `1.12345` | rejected `invalid_quantity_precision` | 5 fractional digits `>` `base_quantity_decimals` (4).      |
| `1.10000` | rejected `invalid_quantity_precision` | trailing zeros count — still 5 fractional digits `>` 4.    |

Fractional-digit precision (`price_decimals` / `base_quantity_decimals`) is a `POST /trade` **admission** rejection — see [`/trade` error codes](transaction-signing.md#trade-error-codes). Significant figures (`max_price_sig_figs`) apply only to non-integer prices and are enforced one layer deeper, at **execution**, as a [`tick`](error-responses.md#execution-level-failures) failure — returned synchronously on the `/trade` response (`submission_status: "rejected"`, `error.code: "tick"`) and also readable on `orderStatus`; an integer display price such as `3500` skips that gate entirely.
