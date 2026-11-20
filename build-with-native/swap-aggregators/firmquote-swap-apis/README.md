# FirmQuote Swap APIs

## Overview

The Native Relay FirmQuote Swap API offers a robust and user-friendly toolkit for seamless integration with Native Core liquidity across various blockchain networks, including Ethereum, BNB Chain, Base, Arbitrum, and others coming in the near future.&#x20;

Our API enables you to leverage Request-For-Quote (RFQ) pricing from Native Core.

## Base URL

Keyed requests use `https://v2.api.native.org/swap-api-v2` and send the key in the `apiKey` header. Fill out the [contact form](https://forms.gle/12LaH1JQyhZqBHqw9) to request a key.

{% hint style="info" %}
For public, no-key access, use `https://v2-pub.api.native.org` as the base URL. Call the same paths and leave off the `apiKey` header, for example `https://v2-pub.api.native.org/swap-api-v2/v1/orderbook`. A rate limit applies. For a higher QPS, use the [contact form](https://forms.gle/12LaH1JQyhZqBHqw9).
{% endhint %}

## **Endpoints**

* **Orderbook:** Use `GET /orderbook` to check price levels for all supported pairs. This endpoint is essential for any aggregator or solver.
* **Indicative quote:** Use `GET /indicative-quote` to obtain regular quotes for a given chain and token pair. Refer to the provided code snippet for implementation details.
* **Firm quote:** Use `GET /firm-quote` to obtain transaction data necessary to execute swaps. Detailed code snippets are available for guidance.
