---
description: Push reject codes, typical fixes, and the default size and rate limits.
layout:
  width: wide
---

# Reject reasons & limits

## Reject reasons

| Code | Meaning | Typical fix |
| --- | --- | --- |
| `stale_nonce` (request-level) | `nonce` not above the last **applied** one; nothing was touched | Retry with a higher nonce |
| `atomic_abort` (status `skipped`) | This book was valid but was not applied, because another book in the same request was rejected | Fix the rejected book and re-push the whole request |
| `invalid_field` (request- or book-level) | A required field is missing, empty or malformed (bad address hex, non-lowercase address, empty `book_id`, non-integer `nonce`, missing symbol, …) | Fix the named field; nothing was touched |
| `symbol_mismatch` | The base or quote symbol does not exactly match the registered symbol for that address (case-sensitive) | Use the exact symbols from the mapping |
| `pair_not_found` | The (chain, base, quote) combination is not a listed pair | Check addresses and chain id |
| `pair_not_owned` | The pair is not assigned to you | Ask ops |
| `orientation_mismatch` | Tokens pushed in the reversed direction | Push in the assigned direction |
| `duplicate_pair` | The same pair appears twice in one request | Send one book per pair |
| `duplicate_book_id` | The same `book_id` appears twice in one request | Use unique ids — results are correlated by `book_id` |
| `expire_time_below_min` | Less than 5 s of remaining lifetime at receipt | Push a later `expire_at_ms`; check clock sync |
| `invalid_level` | Non-positive or unparseable price or qty, or a decimal string longer than 64 chars | Fix the level |
| `levels_out_of_order` | Bids not descending, asks not ascending, or equal prices on a side | Sort and merge |
| `crossed_book` | Best bid ≥ best ask | Re-center the book |
| `too_many_levels` | More levels on a side than the cap | Trim your ladder |
| `level_below_min_size` | A level under the pair's minimum size | Top up or drop the level |

## Limits

| Limit | Default |
| --- | --- |
| Push rate | 10 requests/s per maker (HTTP + WS combined) |
| Orderbooks per request | 20 |
| Levels per side | 100 |
| Request body | 1 MiB |
| WS frame | 64 KiB |
| WS connections per maker | 3 |
| WS idle timeout | 60 s |
| Minimum remaining book lifetime | 5 s |
| WS inbound messages per connection | 25 msg/s |
| Max decimal string length | 64 chars |
| Query rate (GET) | 5 requests/s per maker |

Defaults may be tuned per maker or per pair before and during the trial; your onboarding sheet is authoritative.
