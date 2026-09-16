---
description: >-
  How Native Core computes a credit account's available_usd_atoms — the
  valuation formula, per-asset LTV, short valuation, and mark staleness.
---

# Credit & Margin

A credit account's order-time gate is a single condition: **`available_usd_atoms >= 0`** once the order is applied. This page specifies how that value is computed, so a client can reproduce it locally before signing.

{% hint style="danger" %}
**An order that fails this gate freezes the account.** It is rejected with `insufficientspotcredit` and the account remains frozen until an operator unfreezes it; there is no self-service recovery. The gate is not a validator — compute the post-order value locally and keep headroom.
{% endhint %}

This page applies to **credit accounts** only. A spot account is gated on its per-asset `available` balance; see [Account Types](account-types.md).

## Valuation formula

```
available_usd_atoms = credit_usd_atoms + Σ value(position)
```

summed over every position the account holds. Each position's net size is

```
net = pending_exposure_qty + actual_qty
```

Both terms are returned by [`spotCreditPositions`](post-info.md#spotcreditpositions) as signed decimal strings. A position whose `net` is zero contributes nothing.

`pending_exposure_qty` is size committed by a resting order but not yet filled. It consumes the credit line from the moment the order rests: cancelling the order releases it, and a fill converts it into `actual_qty`. Headroom computed from filled positions alone is therefore an over-estimate.

All values are in `usd_atoms` (`USD_SCALE = 10^8`). Every division rounds down, so the sum is never optimistic.

## Position valuation

`value(net)` depends on the sign of `net` and on whether the asset's mark is fresh.

| | mark fresh | mark stale | no mark |
| --- | --- | --- | --- |
| **Long** (`net > 0`) | `net × mark × credit_ltv% / 10^balance_decimals` | contributes `0` | contributes `0` |
| **Short** (`net < 0`) | `net × mark / 10^balance_decimals` — LTV is not applied | `net × ceil(mark × 13 / 10) / 10^balance_decimals` | order rejected with `OracleMarkPriceMissing` |

Consequences of that table:

* **LTV applies to long positions only.** A short is a liability and is carried at full value. A long and a short of equal notional do not offset.
* **An unpriceable long contributes no collateral; an unpriceable short still carries its full liability.** Collateral that cannot be priced is never credited, and a debt that cannot be priced is never discounted. A stale short is marked up, over-stating the liability.
* **A short with no mark at all rejects the order** rather than being valued conservatively, because the liability has no bound.

A mark is **fresh** only when it was updated at the height the valuation runs at; any earlier update is stale. Mark values are the `usd_atoms` field of [`markPrices`](post-info.md#markprices).

`credit_ltv`, returned by [`assets`](post-info.md#assets), is the effective percentage and is used directly. `credit_ltv_setting` is the explicit configuration and is `null` for an asset that has never been configured; `credit_ltv` resolves that case. An asset at `credit_ltv: 0` contributes no collateral at any size, while still carrying full weight as a short.

## Worked example

A credit account with a **$100,000** line holds two long positions and one short. All three assets use `balance_decimals: 8`, so `10^balance_decimals = 10^8`.

| asset | `net` | mark | `credit_ltv` | side |
| --- | ---: | ---: | ---: | --- |
| USDC | `+50,000` | `$1.00` | 95 | long |
| ETH | `+10` | `$3,500.00` | 85 | long |
| BTC | `−0.5` | `$76,000.00` | 85 | short |

With all three marks fresh:

```
credit_usd_atoms             10,000,000,000,000     $100,000.00
USDC   +50,000 × 1.00 × 95%  +4,750,000,000,000     +$47,500.00
ETH        +10 × 3,500 × 85% +2,975,000,000,000     +$29,750.00
BTC       −0.5 × 76,000      −3,800,000,000,000     −$38,000.00
                             ────────────────────────────────────
available_usd_atoms          13,925,000,000,000     $139,250.00
```

The BTC row is the asymmetry in the table above: at 85% LTV a long of 0.5 BTC would contribute $32,300, while the same size held short costs the full $38,000.

With the BTC mark stale and the other two unchanged:

```
BTC  −0.5 × ceil(76,000 × 13/10) = −0.5 × 98,800   −4,940,000,000,000   −$49,400.00
                                                   ────────────────────────────────────
available_usd_atoms                                12,785,000,000,000   $127,850.00
```

A single stale mark reduces headroom by **$11,400** with no change in position. An account operating close to its limit can therefore fail the gate, and be frozen, as a result of oracle staleness alone.

## Reproducing the value

Four queries supply every input:

| query | fields |
| --- | --- |
| [`spotCreditAccount`](post-info.md#spotcreditaccount) | `credit_usd_atoms`; `available_usd_atoms` for comparison against the locally computed result |
| [`spotCreditPositions`](post-info.md#spotcreditpositions) | `pending_exposure_qty`, `actual_qty` per asset |
| [`markPrices`](post-info.md#markprices) | `usd_atoms`, `updated_height` per asset |
| [`assets`](post-info.md#assets) | `credit_ltv`, `balance_decimals` per asset |

The arithmetic must be performed in integers. Floating-point evaluation does not reproduce floor division at atom scale, and the gate is an exact integer comparison.

To evaluate an order before submitting it, apply its size to the relevant asset's `net` and recompute. The order is admitted when the result is `>= 0`.

## Null and monitoring values

`available_usd_atoms` is `null` when a short position has no mark — the same condition that rejects an order. It means the account cannot currently be valued, and is not equivalent to zero.

`last_known_available_usd_atoms` uses the most recently committed mark for every asset regardless of freshness. It is a monitoring value that remains stable while marks are stale, and it is never the gate. Only `available_usd_atoms` determines whether an order is admitted.
