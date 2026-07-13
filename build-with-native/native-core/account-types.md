---
description: The two kinds of Native Core trading account — spot (balance) and credit — and which API surface reads each.
---

# Account Types

Native Core has two kinds of trading account. Every owner is exactly one of them, and each has a different read surface and a different risk gate at order time.

| | **Spot account** (balance) | **Credit account** |
| --- | --- | --- |
| How you get one | Created on your **first deposit** — the default account. | **Granted by the protocol.** You do not open one yourself. |
| What backs a trade | Your `available` balance per asset. | A USD credit line. |
| Signed / short positions | No — you can only trade balance you hold. | Yes — positions are signed; `actual_qty` may be negative (short). |
| Risk gate when an order runs | Sufficient `available` balance. | `available_usd >= 0` against the credit line. |
| State | `active` / `frozen` | `active` / `frozen` |

A `frozen` account may only cancel; new orders and modifies are rejected. **Most integrations are spot accounts** — if the protocol has not granted you a credit line, you are a spot account and the credit-account reads below simply report that you have no credit position.

## Reading an account

Point every `POST /info` account query at the **owner** address (not the API-wallet address). Which query you use depends on the account:

**Spot account**

* [`userBalances`](post-info.md#userbalances) — `available` / `locked` per asset
* [`deposits`](post-info.md#deposits) / [`withdraws`](post-info.md#withdraws) — funding history
* [`accountStatus`](post-info.md#accountstatus) — whether the account exists and its freeze state

**Credit account**

* [`spotCreditAccount`](post-info.md#spotcreditaccount) — credit line, `available_usd`, and status
* [`spotCreditPositions`](post-info.md#spotcreditpositions) — signed long/short positions per asset

The credit account's on-chain name is a *spot-credit account*, which is why its query types are `spotCreditAccount` and `spotCreditPositions`.

## Trading and moving between them

`order`, `cancel`, and `modify` are the **same** actions for both account types — Native Core applies balance gating or credit gating automatically from the signer's account kind. There is no separate order endpoint per account type.

Two public [`POST /trade`](post-trade.md) actions bridge the two:

* [`settle`](post-trade.md#settle) — a credit account moves a long position out into a spot account's `available` balance.
* [`repay`](post-trade.md#repay) — a spot account spends its `available` balance to reduce a credit account's short.

{% content-ref url="post-info.md" %}
[post-info.md](post-info.md)
{% endcontent-ref %}

{% content-ref url="post-trade.md" %}
[post-trade.md](post-trade.md)
{% endcontent-ref %}
