# pAMM Swap APIs

## Overview

Native Relay pAMM is an onchain-only swap path into Native Core liquidity. There is no offchain quote API and no RFQ signature.

You discover the pair's engine on `NativeRFQPool`, quote with `PropAMMEngine.getQuote`, then execute with `NativeRouter.tradePAMM`.

{% hint style="info" %}
`directSwapEnabled` is the live flag. Quotes still work when it is false; swaps revert with `DirectSwapDisabled`. Always read the flag onchain before sending a swap.
{% endhint %}

pAMM is a **single-hop** path. There is no RFQ fallback if you swap directly with `tradePAMM`. Use [FirmQuote Swap APIs](../firmquote-swap-apis/ "mention") when you need a signed RFQ with fallback and a higher swap success rate.

## Interfaces

* **Address references:** V6 router/pool, pair engines, and how to confirm a pair onchain.
* **Quote and swap:** Discover the engine, call `getQuote`, approve, then `tradePAMM`.
* **Events:** Listen to `PAMMTrade` on the pool. The engine does not emit a fill event.
* **Error handling:** Common onchain reverts and what they mean.

## Test UI

Staging app: [https://native-pamm-staging.vercel.app/](https://native-pamm-staging.vercel.app/)

Connect a wallet and use **BNB Chain**. Quotes and swaps are fully onchain.
