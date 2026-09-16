---
description: >-
  The margin model for a Native Core credit account — how positions are held,
  how available_usd_atoms is calculated, and what a resting order consumes.
---

# Credit & Margin

A credit account trades against a USD credit line rather than a per-asset balance. Its margin headroom is `available_usd_atoms`, and an order is admitted only if that value is still `>= 0` once the order is applied. This page specifies the model and the calculation.

It covers **credit accounts** only; a spot account is gated on its per-asset `available` balance, described in [Account Types](account-types.md).

{% hint style="danger" %}
An order that fails this gate at execution leaves the account frozen until an operator unfreezes it. Clients therefore compute the value locally before signing, rather than establishing it by submitting an order.
{% endhint %}

## The position model

A credit account holds one signed position per asset, not a directional position per market.

```
net = pending_exposure_qty + actual_qty
```

Both terms are in that asset's raw balance atoms, not display units; conversion uses `balance_decimals`, described in [Decimals & Units](decimals-units.md).

**The sign states which way the account stands in that asset.** Positive means the account holds it, counting as collateral; negative means it owes it, counting as a liability. This page calls the two cases long and short. A fill moves two positions in opposite directions — the asset received rises, the asset paid falls — so an account buying ETH with USDC is long ETH and short USDC at once.

The sign is never replaced by the magnitude. It carries three consequences:

* only a positive `net` is haircut by LTV;
* the position adds or subtracts through ordinary signed addition;
* division floors, so a negative value rounds away from zero. Valuing a short from `|net|` and subtracting truncates toward zero instead, understating the liability.

### Committed size and settled size

`actual_qty` is settled size; `pending_exposure_qty` is size a resting order has committed and not yet filled. Both count against the credit line on the same terms, and a fill converts one into the other, so the sum counts each unit once.

A resting order commits **one** asset, always as a debit:

| side | asset committed | amount |
| --- | --- | --- |
| `ask` | the market's **base** asset | the resting quantity, in base balance atoms |
| `bid` | the market's **quote** asset | the resting quantity's notional (`price × quantity`) |

A bid commits the asset it would pay with, not the asset being acquired. The opposite leg appears in `actual_qty` only once a fill settles.

## The calculation

```
available_usd_atoms = credit_usd_atoms + sum of value(position)
```

summed over every position the account holds. With a fresh mark, `value(net)` is

```
long  (net > 0)
  floor( net * mark * credit_ltv / (10^balance_decimals * 100) )

short (net < 0)
  floor( net * mark / 10^balance_decimals )
```

`credit_ltv` is an integer percentage, so the `* 100` divisor is part of the expression. **LTV applies to longs only**: a short is carried at full value, so a long and a short of equal notional do not offset. Both expressions floor, which rounds against the account on either side.

### Stale and missing marks

A mark is **fresh** when its `updated_height` equals the height the valuation runs at. In a [`markPrices`](post-info.md#markprices) response that height is the response's own `query_height`; at the order gate it is the block the order lands in. Both operands are in the same response, so the comparison requires no second query.

Any earlier update is stale, and changes `value(net)`:

| position | `value(net)` |
| --- | --- |
| **Long**, mark stale or missing | `0` |
| **Short**, mark stale | as above, with `mark` replaced by `mark` times a protocol haircut of at least 1.0, rounded up |
| **Short**, mark missing | undefined — the account cannot be valued, and the order is rejected |

A stale mark can therefore only reduce headroom, never increase it. The haircut is a protocol schedule parameter keyed on block height; no query returns it, and it can change at a fork.

This table covers positions the order does not trade. The assets of the market being traded are checked for a fresh mark before valuation, and the order is rejected with `OracleMarkPriceMissing` if either is stale.

## Worked examples

{% hint style="info" %}
The marks and LTV percentages below are **illustrative** and are not any asset's real values. Live values are returned by [`markPrices`](post-info.md#markprices) and [`assets`](post-info.md#assets).
{% endhint %}

Both examples use `balance_decimals: 8` throughout, so `10^balance_decimals` is `100000000`, and a USDC mark of exactly `100000000` (`$1.00`). Fees are omitted.

### A resting bid, from placement to fill

A `$100,000` line, no positions, then a limit **bid** for `10` ETH at `$3,500` — a notional of `$35,000`. ETH has `credit_ltv: 85` and a mark of `350000000000`.

| | `net` USDC | `net` ETH | `available_usd_atoms` | |
| --- | ---: | ---: | ---: | --- |
| before | `0` | `0` | `10000000000000` | $100,000.00 |
| order resting | `-3500000000000` | `0` | `6500000000000` | $65,000.00 |
| fully filled | `-3500000000000` | `1000000000` | `9475000000000` | $94,750.00 |

**Placement** commits the quote asset only. `pending_exposure_qty` for USDC becomes the notional, and a negative `net` is carried at full value with no LTV:

```
floor(-3500000000000 * 100000000 / 100000000)  =  -3500000000000
```

ETH is untouched: its `net` stays `0`, contributes nothing, and needs no mark.

**The fill** releases the pending exposure and books both settlement legs. USDC's pending returns `+3500000000000` and its `actual_qty` takes `-3500000000000`, leaving `net` unchanged — a fill converts committed size into settled size. The ETH leg is new, and being long it is haircut:

```
floor(1000000000 * 350000000000 * 85
      / (100000000 * 100))   =  2975000000000
```

The filled buy therefore costs `$5,250.00` of headroom, or `(100 - 85)%` of notional. **A credit account consumes headroom when it buys, not only when it sells short**: the cost of holding a long is the part of its value that LTV does not return.

### An account with three positions

A `$100,000` line, two longs and a short, all marks fresh.

| asset | `net` (atoms) | `mark` (`usd_atoms`) | `credit_ltv` |
| --- | ---: | ---: | ---: |
| USDC | `5000000000000` | `100000000` | 95 |
| ETH | `1000000000` | `350000000000` | 85 |
| BTC | `-50000000` | `7600000000000` | 85 |

```
credit_usd_atoms                      10000000000000

USDC  floor(5000000000000 * 100000000 * 95
            / (100000000 * 100))       4750000000000
ETH   floor(1000000000 * 350000000000 * 85
            / (100000000 * 100))       2975000000000
BTC   floor(-50000000 * 7600000000000
            / 100000000)              -3800000000000
                                      --------------
available_usd_atoms                   13925000000000
```

That is $100,000.00 of credit, plus $47,500.00 and $29,750.00 of collateral after LTV, less $38,000.00 for the short: **$139,250.00** of headroom.

The BTC row shows the long/short asymmetry. At `credit_ltv: 85` the same `0.5` BTC held long would contribute $32,300; held short it costs the full $38,000.

If the BTC mark goes stale, the short is revalued upward by the haircut and headroom falls with no change in position. An account operating close to its limit can therefore fail the gate, and be frozen, as a result of oracle staleness alone.

## Reading the inputs

Four `POST /info` queries supply every term, each read against the **owner** address.

| query | field | meaning |
| --- | --- | --- |
| [`spotCreditAccount`](post-info.md#spotcreditaccount) | `credit_usd_atoms` | the credit line, in `usd_atoms`, as a JSON number |
| | `available_usd_atoms` | the protocol's own result, as a JSON string, for comparison |
| [`spotCreditPositions`](post-info.md#spotcreditpositions) | `pending_exposure_qty` | committed by resting orders, signed raw atoms |
| | `actual_qty` | settled size, signed raw atoms |
| [`markPrices`](post-info.md#markprices) | `usd_atoms` | mark price scaled by `USD_SCALE = 10^8` |
| | `updated_height` | the block in which that mark was committed |
| | `query_height` | response-level; the height freshness is tested against |
| [`assets`](post-info.md#assets) | `credit_ltv` | effective loan-to-value percentage, an integer |
| | `balance_decimals` | atom scale of that asset's balances |

`credit_ltv` is the effective percentage and is the value the calculation takes. `credit_ltv_setting` is the raw per-asset override, `null` whenever none is configured; an unconfigured asset still has an effective `credit_ltv`, so `credit_ltv_setting: null` does not imply a zero LTV.

The whole calculation is performed in integers: floating-point evaluation does not reproduce floor division at atom scale, and the gate is an exact integer comparison. `credit_usd_atoms` arrives as a JSON number while the two available fields arrive as strings, so a client parsing JSON natively receives two types for the same unit.

Evaluating a prospective order means subtracting its committed amount from the committed asset's `net` and recomputing. The order is admitted when the result is `>= 0`.

A locally computed value is a snapshot: a mark that is fresh when read is stale one block later unless the oracle republishes. [`spotCreditState`](websocket.md#spotcreditstate) streams the same positions and credit line as an alternative to polling.

## When the value is `null`

`available_usd_atoms` is `null` when the account cannot be valued, which is not the same as zero: when the owner has no credit line — the common case, see [`spotCreditAccount`](post-info.md#spotcreditaccount) — when a short position has no mark, and when the valuation overflows. A stale or missing mark on a **long** does not produce `null`; that position contributes `0`.

`last_known_available_usd_atoms` uses the most recently committed mark for every asset regardless of freshness, and is `null` only when a position asset has never had a mark. It is a monitoring value that stays stable while marks are stale. It is never the gate: only `available_usd_atoms` reflects the rules that admit an order.
