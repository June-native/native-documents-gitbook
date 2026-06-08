---
layout:
  width: wide
  title:
    visible: true
  description:
    visible: true
  tableOfContents:
    visible: true
  outline:
    visible: true
  pagination:
    visible: true
  metadata:
    visible: true
  tags:
    visible: true
  actions:
    visible: true
---

# Native Core (Market Makers)

{% hint style="info" %}
_Last updated: 2026-05-30 (_&#x76;0.&#x34;_)_
{% endhint %}

This document describes the Native Core business API. Clients submit signed trading actions with `POST /trade` and read business state with `POST /info`.

Operational, recovery, and node-private interfaces are intentionally not part of this contract.

## Conventions

* `API_URL` examples use `https://api-test.native.org`.
* Hex values are `0x`-prefixed lowercase strings unless noted otherwise.
* Protocol numeric fields in signed actions are decimal strings.
* Business query responses include `query_height` and `app_hash` when a query view is available. `query_height` is the execution height represented by the published read view.
* `POST /trade` returns admission success or rejection. Public clients confirm effects through business queries such as `orderStatus`, `openOrders`, `userFills`, `userBalances`, `spotCreditPositions`, `spotCreditAccount`, `quoteAssets`, `oracleStatus`, and `markPrices`.

## Transaction Signing

Native Core transactions require signing. Read more:

{% content-ref url="transaction-signing.md" %}
[transaction-signing.md](transaction-signing.md)
{% endcontent-ref %}

## Decimal Units

Native Core executes on integers only. Read more:

{% content-ref url="decimals-units.md" %}
[decimals-units.md](decimals-units.md)
{% endcontent-ref %}

## POST /trade

{% content-ref url="post-trade.md" %}
[post-trade.md](post-trade.md)
{% endcontent-ref %}

## POST /info

{% content-ref url="post-info.md" %}
[post-info.md](post-info.md)
{% endcontent-ref %}

## Changelog

```
2026-05-30: v0.4 - add cancelAll action, trading-fee fields on userFills, orderStatus pending/null-OID overlay, assets.issuer, expanded /trade error codes, and signing example refresh.
2026-05-13: v0.3 - admission precheck and margin controls, migrate quantity precision to markets, and add quote-asset minimum/allowlist admin and query APIs (plus related fixes).
2026-05-13: v0.2 - add decimals related, put sections into subpages
2026-05-12: v0.1 - initial base, with tx signing
```
