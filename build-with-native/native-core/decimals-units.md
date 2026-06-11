# Decimals Units

Native Core executes on integers only. Public order and modify payloads accept human decimal strings for `price` and `quantity`; the write path converts them to the raw integer atoms encoded in the signed transaction bytes. Integer-only fields such as ids, timestamps, nonces, and accounting amounts remain decimal integer strings. Public query responses use asset and market decimal metadata to return human-readable display strings where that is useful.

### Decimal Units

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

### Market Decimals

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

The checked-in testnet/mainnet genesis quote allowlist is USDC asset `1` with `min_quantity="10"`, USDT asset `2` with `min_quantity="10"`, ETH asset `3` with `min_quantity="0.01"`, and BNB asset `4` with `min_quantity="0.02"`. Operators manage quote-asset minimums through the internal API documented in `docs/internal-api.md`.

### Write-Time Numeric Validation

`POST /trade` validates signed action numbers before reconstructing the canonical signed payload:

* Integer fields must be base-10 integer strings: no decimal point, no sign, no exponent notation, no commas.
* Order and modify `price` / `quantity` are human decimal strings: no sign, no exponent notation, no commas, and no more fractional digits than the market's `price_decimals` / `base_quantity_decimals`.
* `market_id` parses as `u32`.
* `oid`, `nonce`, `agent_epoch`, and `expires_after_ms` parse as `u64`.
* `quantity` must be greater than zero.
* `price` must be present for both limit orders and protected market orders in the current protocol encoding.
* `market` orders are valid only with `ioc` or `fok`.
* `gtc` and `alo` are valid only for limit orders.
* non-integer display prices may have at most `max_price_sig_figs` significant figures; integer display prices are always allowed.
* quote notional is computed with widened arithmetic and must fit into `u64` quote balance atoms.
* If the configured minimum spot notional is nonzero, the raw quote notional must be at least that minimum.
