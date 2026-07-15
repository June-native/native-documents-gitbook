---
description: The two kinds of Native Core trading account — spot (balance) and credit — and which API surface reads each.
---

# Account Types

Native Core has two kinds of trading account. Every owner is exactly one of them, and each has a different read surface and a different risk gate at order time.

| | **Spot account** (balance) | **Credit account** |
| --- | --- | --- |
| Provisioning | Created automatically on **first deposit** — the default account. | **Provisioned by the protocol**; not self-service. |
| Collateral | Per-asset `available` balance. | A USD credit line. |
| Short positions | Not supported — trading is limited to held balance. | Supported — positions are signed (`actual_qty` may be negative). |
| Order-time risk gate | Sufficient `available` balance. | `available_usd >= 0` against the credit line. |
| State | `active` / `frozen` | `active` / `frozen` |

A `frozen` account may only cancel — new orders and modifies are rejected. Most integrations use a **spot account**; without a protocol-granted credit line, an owner is a spot account and the credit-account reads below report no credit position.

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
