---
description: >-
  How Native Core computes a credit account's available_usd_atoms — the
  valuation formula, per-asset LTV, and mark freshness.
---

# Credit & Margin

`available_usd_atoms` is a credit account's headroom. An order is admitted only if the value is still `>= 0` once the order is applied. This page specifies how it is computed.

It applies to **credit accounts** only. A spot account is gated on its per-asset `available` balance instead; see [Account Types](account-types.md).

{% hint style="danger" %}
An order that fails this gate at execution leaves the account frozen until an operator unfreezes it. Clients therefore compute the value locally before signing, rather than establishing it by submitting an order.
{% endhint %}

## Valuation formula

```
available_usd_atoms = credit_usd_atoms + sum of value(position)
```

over every position the account holds. Each position's net size is

```
net = pending_exposure_qty + actual_qty
```

Both fields are returned by [`spotCreditPositions`](post-info.md#spotcreditpositions) as **raw signed atom strings**, not display amounts. Conversion uses the asset's `balance_decimals`; see [Decimals & Units](decimals-units.md).

`pending_exposure_qty` is size a resting order has committed but not yet filled. It counts against the credit line on the same terms as filled size, and converts into `actual_qty` as the order fills, so the sum counts each unit of size once.

A resting order commits **one** asset, and always as a debit:

| side | asset debited | amount |
| --- | --- | --- |
| `ask` | the market's **base** asset | the resting quantity, in base balance atoms |
| `bid` | the market's **quote** asset | the resting quantity's notional (`price × quantity`) |

A bid therefore commits the quote asset, not the asset being acquired. The opposite leg appears in `actual_qty` only once a fill produces a settlement delta.

With a fresh mark, `value(net)` is

```
long  (net > 0)
  floor( net * mark * credit_ltv / (10^balance_decimals * 100) )

short (net < 0)
  floor( net * mark / 10^balance_decimals )
```

`credit_ltv` is an integer percentage, so the `* 100` divisor is part of the formula. **LTV is applied to longs only**: a short is a liability carried at full value, and a long and a short of equal notional do not offset.

`credit_ltv`, returned by [`assets`](post-info.md#assets), is the effective percentage for that asset and is the value the formula takes. `credit_ltv_setting` is the raw per-asset override and is `null` whenever none is configured. An unconfigured asset still has an effective `credit_ltv`, which the protocol resolves and `assets` reports, so `credit_ltv_setting: null` does not imply a zero LTV.

Both expressions floor, which rounds against the account whether the position is long or short.

## Mark freshness

A mark is **fresh** only when its `updated_height` equals the height the valuation runs at; any earlier update is stale. The applicable height differs by context: in a [`markPrices`](post-info.md#markprices) response it is that response's `query_height`, and at the order gate it is the block the order lands in. Both operands appear in the same `markPrices` response, so the comparison requires no second query.

The valuation of a stale mark depends on the action:

| position | placing an order | `settle` / `repay` |
| --- | --- | --- |
| **Long**, mark stale or missing | contributes `0` | rejected |
| **Short**, mark stale | valued at a marked-up price | rejected |
| **Short**, mark missing | cannot be valued — rejected | rejected |

A stale mark elsewhere in the account therefore does not block trading, but it can only reduce headroom, never increase it. `settle` and `repay` value every position strictly, so the same stale mark that trading tolerates blocks both of them.

For a stale short, the mark is multiplied by a protocol haircut of at least 1.0 and rounded up, so the short is over-stated rather than under-stated. The haircut is a protocol schedule parameter keyed on block height; no query returns it, and it can change at a fork. Its effect scales linearly with the position's full value: on a short worth $38,000, each 0.1 of haircut removes $3,800 of headroom.

A separate check precedes valuation: both assets of the market being traded must have a mark updated in the current block, or the order is rejected with `OracleMarkPriceMissing`.

## Worked example

{% hint style="info" %}
The marks and LTV percentages below are **illustrative** and are not any asset's real values. Live values are returned by [`markPrices`](post-info.md#markprices) and [`assets`](post-info.md#assets).
{% endhint %}

A credit account with a $100,000 line holds two longs and a short. All three assets use `balance_decimals: 8`, so `10^balance_decimals` is `100000000`. All marks are fresh.

| asset | `net` (atoms) | `mark` (`usd_atoms`) | `credit_ltv` |
| --- | ---: | ---: | ---: |
| USDC | 5000000000000 | 100000000 | 95 |
| ETH | 1000000000 | 350000000000 | 85 |
| BTC | -50000000 | 7600000000000 | 85 |

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

In display terms: $100,000.00 of credit, plus $47,500.00 and $29,750.00 of collateral after LTV, less $38,000.00 for the short, giving **$139,250.00** of headroom.

The BTC row shows the long/short asymmetry: at `credit_ltv: 85` the same 0.5 BTC held long would contribute $32,300, while held short it costs the full $38,000.

If the BTC mark goes stale, the short is revalued upward by the haircut and headroom falls with no change in position. An account operating close to its limit can therefore fail the gate, and be frozen, as a result of oracle staleness alone.

## Reproducing the value

| query | fields |
| --- | --- |
| [`spotCreditAccount`](post-info.md#spotcreditaccount) | `credit_usd_atoms`; `available_usd_atoms` for comparison against the locally computed result |
| [`spotCreditPositions`](post-info.md#spotcreditpositions) | `pending_exposure_qty`, `actual_qty` per asset |
| [`markPrices`](post-info.md#markprices) | `usd_atoms` and `updated_height` per asset, plus the response's `query_height` |
| [`assets`](post-info.md#assets) | `credit_ltv`, `balance_decimals` per asset |

The arithmetic must be performed in integers. Floating-point evaluation does not reproduce floor division at atom scale, and the gate is an exact integer comparison. `credit_usd_atoms` is returned as a JSON number while `available_usd_atoms` and `last_known_available_usd_atoms` are strings, so a client that parses JSON natively receives two different types for the same unit.

Evaluating a prospective order means subtracting its committed amount from the debited asset's `net` and recomputing — for an `ask`, the quantity in base balance atoms; for a `bid`, the quote notional. The order is admitted when the result is `>= 0`.

A locally computed value is a snapshot: a mark that is fresh when read is stale one block later unless the oracle republishes. [`spotCreditState`](websocket.md#spotcreditstate) streams the same positions and credit line as an alternative to polling.

## When the value is `null`

`available_usd_atoms` is `null` when the account cannot be valued, which is not the same as zero. It is `null` when the owner has no credit line — the common case, see [`spotCreditAccount`](post-info.md#spotcreditaccount) — when a short position has no mark, and when the valuation overflows. A stale or missing mark on a **long** does not produce `null`; that position contributes `0`.

`last_known_available_usd_atoms` uses the most recently committed mark for every asset regardless of freshness, and is `null` only when a position asset has never had a mark. It is a monitoring value that stays stable while marks are stale. It is never the gate: only `available_usd_atoms` reflects the rules that admit an order.
