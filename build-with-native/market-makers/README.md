---
description: Push orderbooks for assigned pairs. Native prices and signs every firm quote from your latest valid book.
layout:
  width: wide
---

# External Market Maker Orderbook Push API

{% hint style="info" %}
Version 0.2 · 13 Nov 2026. Defaults in [Limits](errors-and-limits.md#limits) are provisional and finalized with your onboarding sheet. A [Chinese counterpart](zh.md) is maintained in sync; if the two differ, the English pages win.
{% endhint %}

Native's external market maker integration lets you make markets on Native by **pushing orderbooks** for the pairs assigned to you.

* **You push, we quote.** You push your bids and asks. Native prices every firm quote from your latest valid book, applies its own fee, and signs the quote itself.
* **Inbound only.** Native never calls you on the quote path — no price fetch, no last look, no sign request.
* **You are the only price source.** For a pair assigned to you, your book is the book. If your book is missing, expired or withdrawn, the pair simply does not quote — nothing else fills in.
* **Two transports, one semantics.** Push each update as one HTTPS request, or stream updates over a WebSocket. Same book format, same validation, same ordering.

{% hint style="warning" %}
**No last look — a signed quote is final.** A firm quote is priced from your book at request time and stays executable for its full validity (this window follows the upstream's requirement and is set globally by Native: you cannot choose it per pair; ask Native for a global change). A newer push, an expired book, a withdrawal or a pause stops *new* quotes, but never cancels a quote already signed. You accept fills at the signed price until that quote's deadline. Price your books and choose your expiry with this in mind.
{% endhint %}

{% content-ref url="http-api.md" %}
[http-api.md](http-api.md)
{% endcontent-ref %}

{% content-ref url="websocket.md" %}
[websocket.md](websocket.md)
{% endcontent-ref %}

{% content-ref url="errors-and-limits.md" %}
[errors-and-limits.md](errors-and-limits.md)
{% endcontent-ref %}

{% content-ref url="zh.md" %}
[zh.md](zh.md)
{% endcontent-ref %}

## Before you start

Native ops gives you, per maker account:

| Item | Description |
| --- | --- |
| API key | The value for the `Apikey` header |
| IP allowlist | The source IPs allowed to use the key |
| Assigned pairs | For each pair: chain, base token, quote token, and the direction you must push |
| Pair limits | Minimum level size, level cap, and fee rate for your pairs |
| Token mapping | For each pair: the contract addresses and the exact symbol strings (case-sensitive) for base and quote |

All traffic is TLS (HTTPS / WSS). This document uses `https://api.native.org` and `wss://api.native.org` as example URLs — use the exact URLs from your onboarding sheet.

## Core concepts

### Book

A book is your bids and asks for one pair.

| Side | Meaning | Order |
| --- | --- | --- |
| `bids` | Levels where you **buy base** (pay quote) | Price descending, best first |
| `asks` | Levels where you **sell base** (receive quote) | Price ascending, best first |

A level is:

```json
{ "price": "0.98", "qty": "12000" }
```

* `price` — quote per 1 base.
* `qty` — level size, in base. **Per level, not cumulative.**
* Both are decimal strings, so precision survives the wire.
* Push your own prices. Native adds its fee on top when quoting; you add nothing for Native.
* The best bid must be strictly below the best ask — a crossed book is rejected.

### Pair and direction

A book identifies its pair with `chain_id` plus the two tokens. **Every field is required and validated.** Addresses must be **lowercase** EVM hex — checksummed or mixed-case forms are rejected (`invalid_field`). Symbols are matched **case-sensitively** and must equal the registered symbol exactly (`MEME` ≠ `meme`); the address → symbol mapping for your pairs is in your onboarding sheet, and `GET /v1/orderbook` echoes it for every pair with a live book. Push in exactly the direction assigned to you — a reversed book is rejected (`orientation_mismatch`), a wrong symbol is rejected (`symbol_mismatch`). Nothing is silently inverted or corrected for you.

### Replace, not patch

Each accepted book replaces your previous book for that pair **in full**:

| You push | Effect |
| --- | --- |
| Both sides non-empty | The pair's book is replaced entirely |
| One side empty (`"bids": []`) | Replaces the book; that side stops quoting |
| Both sides empty | The pair is withdrawn |
| A pair your request does not name | Its book is left as it was |

No deltas, no level ids, no partial updates — every push carries the complete book.

Pushing an empty side or an empty book is how you **cancel**. A one-side-empty book cancels that side while the other keeps quoting; a both-sides-empty book cancels the pair entirely. Withdrawal is immediate and **has no expiry** — it lasts until your next push for that pair; the `expire_at_ms` sent with an empty book is ignored.

### All-or-nothing requests

Every push is a complete set of books, and one request is one complete unit of work:

* A request may carry books for **different chains and different pairs** — each book names its own `chain_id`.
* A request is **atomic**: either every book passes validation and all of them are applied, or nothing is applied at all. There is no partial state in between.
* The nonce is consumed **only when the request is applied**. A request that fails has zero side effects — nothing changes, not even the nonce. Retry with the same or a higher nonce.
* When the request applies, the response is a bare confirmation — no per-book results. When it fails, `results` lists **every** book as `rejected` (with a reason) or `skipped` (this book was fine, but was not applied because the request as a whole failed).
* Pushing a single pair is the same protocol with a one-element `orderbooks` array. Partial acceptance inside one request may be enabled later **without any schema change** — parse `results[].status` per book and you are already future-proof.

### Expiry: `expire_at_ms`

* An absolute Unix timestamp in milliseconds — the moment your book stops pricing new quotes.
* Native reads it on Native's clock. Keep your hosts NTP-synced; every ack carries `server_time_ms` so you can watch drift.
* Remaining lifetime at receipt must be **5 seconds or more**. There is no maximum — a longer expiry only extends your own staleness risk.
* A newer push replaces the current book immediately, whatever the old book's expiry said.

### Nonce

Every request carries an integer `nonce`:

* It must be **strictly greater** than the nonce of the last **applied** request of your maker account.
* One counter is shared by HTTP and **all** your WebSocket connections. Concurrent pushes race on it — the losers get `stale_nonce`. Multiple connections for HA are fine; if they push simultaneously, expect races.
* A delayed push with an old nonce is rejected (`stale_nonce`); it can never replace a newer book.
* The counter advances only when a request is **applied**. A request that fails validation changes nothing at all — fix it and push again with the same or a higher nonce.

### Trimming

Native caps each side of your book by the position room left for your pair, and removes a side entirely when its room reaches zero (fills on the other side can bring it back). The push response carries no trim data — check the depth still quotable for each pair via `GET /v1/orderbook`; the binding trim happens at quote time with the room of that moment. If trimming leaves a side below the pair's minimum size, that side is effectively removed.

## Good to know

* **Cadence.** A book prices quotes until its `expire_at_ms`. If you stop pushing, the pair goes dark when the last book expires — expiry is your staleness control.
* **You will not receive fills on this API.** Follow your fills in the pool's on-chain `RFQTrade` events, filtered by your pairs' tokens; your account statement from Native ops covers positions, limits and fees.
* **`book_id` pays off.** It flows into your trade statements, so a per-push id makes every fill traceable to the exact book you pushed.
* **Clocks.** `expire_at_ms` is read on Native's clock. Sync with NTP and watch `server_time_ms` in the acks.
* **Cancel by pushing less.** An empty side cancels that side; an empty book cancels the pair entirely. Withdrawal is immediate, has no expiry, and lasts until your next push — unambiguous, unlike widening prices.
* **Atomic retries.** `applied: false` means zero side effects — fix the failing book and push the whole request again, with the same or a higher nonce.
