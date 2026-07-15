---
description: A WebSocket streaming API for live market data and order updates is on the roadmap.
---

# WebSocket

{% hint style="info" %}
**Coming soon.** Streaming is not available yet.
{% endhint %}

Today, all reads are poll-based over [POST /info](post-info.md), and [POST /trade](post-trade.md) returns each write's outcome synchronously. A WebSocket streaming API — live order books, trades, and marks, plus account and order updates — is on the roadmap. This page will document the connection, subscription, and message formats when it ships.

Building now? Poll [POST /info](post-info.md); the [Read market data](read-market-data.md) guide shows which query to use for each feed.
