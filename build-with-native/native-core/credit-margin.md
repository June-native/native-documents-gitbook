---
description: >-
  How a credit account's available_usd_atoms is computed — the formula, the
  per-asset LTV, how shorts are valued, and what a stale mark does to it.
---

# Credit & Margin

A credit account's order-time gate is one number: **`available_usd_atoms >= 0`** after the order is applied. This page is how that number is built, so you can compute it yourself before you sign.

{% hint style="danger" %}
**An order that fails this gate freezes the account.** It is rejected with `insufficientspotcredit` and the account stays frozen until an operator unfreezes it — there is no self-service recovery. Compute the post-order value locally and leave headroom; do not use the gate as a validator.
{% endhint %}

This page applies only to **credit accounts**. A spot account is gated on its per-asset `available` balance and none of the below applies — see [Account Types](account-types.md).

## The formula

```
available_usd_atoms = credit_usd_atoms + Σ value(position)
```

summed over every position the account holds, where each position's net size is

```
net = pending_exposure_qty + actual_qty
```

Both come from [`spotCreditPositions`](post-info.md#spotcreditpositions) as signed decimal strings. A position whose `net` is zero contributes nothing and is skipped.

**`pending_exposure_qty` is the part most integrations miss.** Size that is committed by a resting order but not yet filled already counts against the credit line. Cancelling the order releases it; the fill converts it into `actual_qty`. You cannot compute your headroom from filled positions alone.

Everything is in `usd_atoms` (`USD_SCALE = 10^8`), and every division **rounds down**, so the sum is never optimistic.

## Valuing one position

`value(net)` depends on the sign and on whether the asset's mark is fresh.

| | mark fresh | mark stale | no mark |
| --- | --- | --- | --- |
| **Long** (`net > 0`) | `net × mark × credit_ltv% / 10^balance_decimals` | contributes `0` | contributes `0` |
| **Short** (`net < 0`) | `net × mark / 10^balance_decimals` — **no LTV** | `net × ceil(mark × 13 / 10) / 10^balance_decimals` | order rejected, `OracleMarkPriceMissing` |

Three things follow from that table, and each one surprises somebody:

* **LTV applies to longs only.** A short is a liability and is carried at full value. The same notional long and short do not offset.
* **An unpriceable long is worth nothing, an unpriceable short still costs you.** The protocol never credits collateral it cannot price, and never discounts a debt it cannot price. A stale short is marked *up* — the multiplier over-states the liability on purpose.
* **A missing short mark is fatal to the order,** not merely conservative: the liability cannot be bounded at all, so the order is rejected rather than valued.

`mark` is `usd_atoms` from [`markPrices`](post-info.md#markprices). A mark is **fresh** only when it was updated at the height the valuation runs at; anything older is stale.

`credit_ltv` comes from [`assets`](post-info.md#assets) and is already the effective percentage — use it directly. (`credit_ltv_setting` is the explicit configuration and is `null` when the asset has never been configured; `credit_ltv` resolves that for you.) An asset at `credit_ltv: 0` contributes **nothing as collateral** no matter how much of it you hold, while still carrying full weight as a short.

## Worked example

A credit account with a **$100,000** line, holding two longs and one short. All decimals are 8, so `10^balance_decimals = 10^8`.

| asset | `net` | mark | `credit_ltv` | |
| --- | ---: | ---: | ---: | --- |
| USDC | `+50,000` | `$1.00` | 95 | long |
| ETH | `+10` | `$3,500.00` | 85 | long |
| BTC | `−0.5` | `$76,000.00` | 85 | short |

**All marks fresh:**

```
credit_usd_atoms            10,000,000,000,000     $100,000.00
USDC   +50,000 × 1.00 × 95%  +4,750,000,000,000    +$47,500.00
ETH        +10 × 3,500 × 85% +2,975,000,000,000    +$29,750.00
BTC       −0.5 × 76,000      −3,800,000,000,000    −$38,000.00   ← full value, no LTV
                             ───────────────────────────────────
available_usd_atoms          13,925,000,000,000    $139,250.00
```

Note the BTC row: at 85% LTV a *long* 0.5 BTC would have been worth $32,300, but as a short it costs the full $38,000.

**Same account, BTC's mark now stale:**

```
BTC  −0.5 × ceil(76,000 × 13/10) = −0.5 × 98,800   −4,940,000,000,000   −$49,400.00
                                                   ───────────────────────────────────
available_usd_atoms                                12,785,000,000,000   $127,850.00
```

One stale mark removed **$11,400** of headroom without a single trade. An account sized close to its limit can fail the gate — and freeze — purely because an oracle went quiet.

## Computing it yourself

Three reads, and you can reproduce the number exactly:

| | |
| --- | --- |
| [`spotCreditAccount`](post-info.md#spotcreditaccount) | `credit_usd_atoms`, plus `available_usd_atoms` to check your arithmetic against ours |
| [`spotCreditPositions`](post-info.md#spotcreditpositions) | `pending_exposure_qty` and `actual_qty` per asset |
| [`markPrices`](post-info.md#markprices) | `usd_atoms` and `updated_height` per asset |
| [`assets`](post-info.md#assets) | `credit_ltv` and `balance_decimals` per asset |

Do the arithmetic in integers. Floating point will not reproduce the floor-division at the atom level, and the gate is an exact integer comparison.

To check an order before sending it, apply its size to the relevant asset's `net` and recompute. The order is admitted when the result is `>= 0`.

## Two related fields

`available_usd_atoms` is `null` when a short position has no mark at all — the same condition that rejects an order. Treat `null` as "cannot be valued", not as zero.

`last_known_available_usd_atoms` uses the most recently committed mark for every asset **regardless of freshness**. It is a monitoring value: it tells you roughly where the account stands when marks are stale, and it is the number that does *not* move when an oracle goes quiet. It is never the gate. Only `available_usd_atoms` decides whether an order is admitted.
