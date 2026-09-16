---
description: >-
  The order-time checks a credit account passes, and how Native Core computes
  available_usd_atoms — the valuation formula, per-asset LTV, and mark freshness.
---

# Credit & Margin

A credit account order passes five checks. Four are permission and data-availability checks that leave the account untouched; the fifth is the credit gate, and it is the only one that freezes the account. This page specifies all five, and gives the arithmetic behind the gate so a client can reproduce it before signing.

This page applies to **credit accounts** only. A spot account is gated on its per-asset `available` balance; see [Account Types](account-types.md).

## Order-time checks

Applied in this order. The first failure ends the action.

| # | Check | Rejection | Account left |
| --- | --- | --- | --- |
| 1 | Account is not frozen | `SpotCreditAccountFrozen` | unchanged |
| 2 | Oracle status is available | `OracleUnavailable` | unchanged |
| 3 | Both assets of the **submitted market** have a mark updated in the current block | `OracleMarkPriceMissing` | unchanged |
| 4 | The market is permitted for credit trading | `ActionNotAllowedForSpotCreditAccount` | unchanged |
| 5 | `available_usd_atoms >= 0` once the order is applied | `InsufficientSpotCredit` | **frozen** |

{% hint style="danger" %}
**Only check 5 freezes the account.** The order is rolled back, the account is set to `frozen`, and it stays frozen until an operator unfreezes it — there is no self-service recovery. Checks 1–4 reject the action and leave the account `active`, so a rejected order is not by itself a sign of trouble.

Because a failed gate costs a freeze, it is not usable as a validator. Compute the post-order value locally and keep headroom.
{% endhint %}

Check 4 is documented with the `credit_trading` per-market gate under [`markets`](post-info.md#markets); an account's exceptions are in `credit_trading_whitelisted_market_ids` on [`spotCreditAccount`](post-info.md#spotcreditaccount). The Python SDK combines both halves in `credit_trading_allowed()`.

A frozen account may only cancel. See [Account Types](account-types.md) for what else it blocks.

## Valuation formula

```
available_usd_atoms = credit_usd_atoms + sum of value(position)
```

over every position the account holds. Each position's net size is

```
net = pending_exposure_qty + actual_qty
```

Both fields come from [`spotCreditPositions`](post-info.md#spotcreditpositions) as **raw signed atom strings** — not display amounts. Convert with the asset's `balance_decimals`; see [Decimals & Units](decimals-units.md). A position whose `net` is zero contributes nothing and needs no mark at all, so flattening a position also removes its mark dependency.

`pending_exposure_qty` is size committed by a resting order but not yet filled. It consumes the credit line from the moment the order rests: cancelling the order releases it, and a fill converts it into `actual_qty`. Headroom computed from filled positions alone is therefore an over-estimate. Resting orders are single-leg — a resting ask debits the base asset only, a resting bid debits the quote asset only.

With a fresh mark, `value(net)` is

```
long  (net > 0)   floor( net * mark * credit_ltv / (10^balance_decimals * 100) )
short (net < 0)   floor( net * mark / 10^balance_decimals )
```

`credit_ltv` is an integer percentage, so the `* 100` divisor is part of the formula. **LTV is applied to longs only**: a short is a liability carried at full value, and a long and a short of equal notional do not offset.

`credit_ltv`, returned by [`assets`](post-info.md#assets), is the effective percentage and is used directly. `credit_ltv_setting` echoes the per-asset override and is `null` when none is configured. An asset at `credit_ltv: 0` contributes no collateral at any size, while still carrying full weight as a short.

Long valuation floors. The stale-short haircut below rounds up. Both round against the account, so the sum is never optimistic.

## Mark freshness

A mark is **fresh** only when its `updated_height` equals the height the valuation runs at. Any earlier update is stale. Mark values and `updated_height` are returned by [`markPrices`](post-info.md#markprices), which also returns the `query_height` to compare against.

Freshness is not one rule. Which assets tolerate a stale mark depends on the action and on whether the asset is one the order is trading:

| | Assets of the submitted market | The account's other positions | `settle` / `repay` |
| --- | --- | --- | --- |
| **Long**, mark stale or missing | rejected | contributes `0` | rejected |
| **Short**, mark stale | rejected | valued at a marked-up price | rejected |
| **Short**, mark missing | rejected | rejected | rejected |

Three consequences:

* **The market being traded is always strict.** A stale mark on the order's own base or quote asset rejects it with `OracleMarkPriceMissing` — it is not valued conservatively.
* **A stale mark elsewhere in the account no longer blocks trading**, but it is not free: an unpriceable long contributes no collateral, while an unpriceable short still carries its full liability, marked up.
* **`settle` and `repay` are strict for every position.** An account that can trade may still be unable to settle.

For a stale short, the mark is multiplied by a protocol haircut of at least 1.0 and rounded up, so the liability is over-stated rather than under-stated. The haircut is a protocol schedule parameter keyed on block height; it is not returned by any query and can change at a fork. Its effect scales linearly with the position's full value — on a short worth $38,000, each 0.1 of haircut removes $3,800 of headroom.

## Worked example

{% hint style="info" %}
The marks and LTV percentages below are **illustrative**. They are not any asset's real values — read the live ones from [`markPrices`](post-info.md#markprices) and [`assets`](post-info.md#assets).
{% endhint %}

A credit account with a $100,000 line holds two longs and a short. All three assets use `balance_decimals: 8`, so `10^balance_decimals` is `100000000`. All marks are fresh, and the order being placed is on a market whose own marks are fresh.

| asset | `net` (atoms) | `mark` (`usd_atoms`) | `credit_ltv` |
| --- | ---: | ---: | ---: |
| USDC | 5000000000000 | 100000000 | 95 |
| ETH | 1000000000 | 350000000000 | 85 |
| BTC | -50000000 | 7600000000000 | 85 |

```
credit_usd_atoms                                   10000000000000    $100,000.00
USDC   floor(5000000000000 * 100000000 * 95
             / (100000000 * 100))                   4750000000000     +$47,500.00
ETH    floor(1000000000 * 350000000000 * 85
             / (100000000 * 100))                   2975000000000     +$29,750.00
BTC    floor(-50000000 * 7600000000000
             / 100000000)                          -3800000000000     -$38,000.00
                                                   --------------------------------
available_usd_atoms                                13925000000000    $139,250.00
```

The BTC row shows the long/short asymmetry: at `credit_ltv: 85` the same 0.5 BTC held long would contribute $32,300, while held short it costs the full $38,000.

If the BTC mark goes stale while the account holds no BTC order, the short is revalued upward by the haircut and headroom falls accordingly — with no change in position. An account operating close to its limit can fail check 5, and be frozen, as a result of oracle staleness alone. If instead the stale mark is on the market the order is trading, check 3 rejects it first and the account stays active.

## Reproducing the value

| query | fields |
| --- | --- |
| [`spotCreditAccount`](post-info.md#spotcreditaccount) | `credit_usd_atoms`, `status`, `oracle_status`; `available_usd_atoms` to compare against the locally computed result |
| [`spotCreditPositions`](post-info.md#spotcreditpositions) | `pending_exposure_qty`, `actual_qty` per asset |
| [`markPrices`](post-info.md#markprices) | `usd_atoms` and `updated_height` per asset, plus the response's `query_height` |
| [`assets`](post-info.md#assets) | `credit_ltv`, `balance_decimals` per asset |

The arithmetic must be performed in integers. Floating-point evaluation does not reproduce floor division at atom scale, and the gate is an exact integer comparison. Note that `credit_usd_atoms` is returned as a JSON number while `available_usd_atoms` and `last_known_available_usd_atoms` are strings; a client that parses JSON natively gets two different types for the same unit.

To evaluate an order before submitting it, apply its size to the debited asset's `net` — base for an ask, quote for a bid — and recompute. The order clears check 5 when the result is `>= 0`.

A locally computed value is a projection, not a guarantee. The query evaluates freshness at the last published block; the gate evaluates it at the block the order lands in. A mark that is fresh when read is stale one block later unless the oracle republishes, so leave headroom rather than sizing to the boundary. [`spotCreditState`](websocket.md#spotcreditstate) streams the same positions and credit line for integrations that would rather not poll.

## `available_usd_atoms` and `last_known_available_usd_atoms`

`available_usd_atoms` is `null` when the account cannot be valued, which is not the same as zero. It is `null` when the owner has no credit line (the common case — see [`spotCreditAccount`](post-info.md#spotcreditaccount)), when a short position has no mark, and when the valuation overflows. A stale or missing mark on a **long** does not produce `null`; that position simply contributes `0`.

`last_known_available_usd_atoms` uses the most recently committed mark for every asset regardless of freshness, and is `null` only when a position asset has never had a mark. It is a monitoring value that stays stable while marks are stale. It is never the gate — only `available_usd_atoms` reflects the rules that admit an order.
