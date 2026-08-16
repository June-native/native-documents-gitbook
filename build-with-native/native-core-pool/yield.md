---
description: How Native Core Pool distributes yield — the daily snapshot, the pro-rata share, and the fields that report both.
---

# Yield

Yield is distributed, not accrued. Balances do not grow continuously; they change when a distribution is applied.

Each distribution runs in four steps:

1. At `snapshot_time_utc` each day, the Pool records every account's `earn_balance` for each asset. The snapshot is the balance at that instant, not a daily average.
2. Operations decides the amount to distribute for that asset.
3. Each account receives `distribution × its snapshot balance ÷ the sum of all snapshot balances`, using integer division. The undivided remainder stays with the Pool.
4. The share is added to `earn_balance` and written as a row in `yieldHistory`.

Because the share lands in `earn_balance`, it is included in the next snapshot and compounds.

Distributions are triggered rather than scheduled, so the history is a sparse series with gaps, not one row per day.

## What earns

Only `earn_balance` is snapshotted. The other two buckets are not:

| Bucket                    | Holds                                                  | Earns |
| ------------------------- | ------------------------------------------------------ | ----- |
| `earn_balance`            | Available, earning funds                                | Yes   |
| `in_queue_balance`        | The gross amount of a pending scheduled withdrawal      | No    |
| `withdraw_locked_balance` | The gross amount of an instant withdrawal in flight     | No    |

The three are disjoint. Submitting a withdrawal moves the gross amount out of `earn_balance` in the same transaction, so anything withdrawn before a snapshot earns nothing from it. Cancelling a scheduled withdrawal returns the amount to `earn_balance`, where it is eligible again.

Holding time within a day does not matter. Only the balance at `snapshot_time_utc` counts.

## Reported rates

`config` reports two figures per asset.

| Field                 | Meaning                                                                        |
| --------------------- | ------------------------------------------------------------------------------ |
| `projected_apy`       | The most recent distribution, annualized over the interval since the previous one, divided by current TVL |
| `realtime_tvl_amount` | The asset's total earning balance, in that asset's atoms                        |

`projected_apy` is a **ratio, not a percentage**: `"0.12"` is 12%.

{% hint style="info" %}
`projected_apy` returns `"0"` both for a genuine zero and for every case where the figure cannot be computed: fewer than two completed distributions, a non-positive TVL, or a non-positive latest distribution. The two are indistinguishable, so `"0"` cannot be read as a rate.

Both fields are omitted from the response when the backend returns nothing for them, so read them defensively.
{% endhint %}

The Pool has no interest-rate setting. Each distribution's size is decided when it is made, and `projected_apy` is derived from what has already been distributed.

## Cumulative yield

`lifetime_yield` in [`account`](reference.md#account) is the total credited to an address for one asset.

```bash
curl -sS "$POOL_API_URL/api/v3/earn" \
  -H 'content-type: application/json' \
  -d '{"type":"account","user_address":"0x5555…5555"}'
```

`lifetime_yield` is **per asset**. Assets use different precisions, so a sum across assets has no meaning. Report per asset, or convert to a common currency first.

## Distribution history

```bash
curl -sS "$POOL_API_URL/api/v3/earn" \
  -H 'content-type: application/json' \
  -d '{"type":"yieldHistory","user_address":"0x5555…5555","asset_id":2,"limit":20}'
```

```json
{
  "code": 0,
  "data": {
    "items": [
      {
        "id": 12,
        "distribution_id": "distribution:3f9a…c81e",
        "asset_id": 2,
        "snapshot_balance": "800000000",
        "yield_amount": "1600000",
        "applied_at_unix_ms": 1786694400000
      }
    ],
    "next_before_id": null
  },
  "message": "success"
}
```

Each row carries everything needed to render one payout, with no follow-up request:

| Field                | Meaning                                    |
| -------------------- | ------------------------------------------ |
| `snapshot_balance`   | The balance the share was computed from     |
| `yield_amount`       | The amount credited                         |
| `applied_at_unix_ms` | When it was credited                        |

`distribution_id` is an opaque grouping key. It contains no date; use `applied_at_unix_ms` for time.

The list contains applied rows only, so it sums to `lifetime_yield` — but only within one asset. **Pass `asset_id`.** Without it the response mixes every asset the address holds, and comparing that sum against a single `lifetime_yield` will never reconcile.

An account whose pro-rata share rounds down to zero gets no row for that distribution.

`asset_id`, `before_id` and `limit` behave as they do on the other lists; see [Pagination](reference.md#pagination).

## Next steps

{% content-ref url="withdraw.md" %}
[withdraw.md](withdraw.md)
{% endcontent-ref %}

{% content-ref url="reference.md" %}
[reference.md](reference.md)
{% endcontent-ref %}
