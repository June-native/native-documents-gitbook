# FirmQuote Swap APIs

## Overview

The Native Relay FirmQuote Swap API offers a robust and user-friendly toolkit for seamless integration with Native Core liquidity across various blockchain networks, including Ethereum, BNB Chain, Base, Arbitrum, and others coming in near future.&#x20;

Our API enables you to leverage Request-For-Quote (RFQ) pricing from Native Core.

## **Endpoints**

* **Orderbook:** Use `GET /orderbook` to check price levels for all supported pairs. This endpoint is essential for any aggregator or solver.
* **Indicative quote:** Use `GET /indicative-quote` to obtain regular quotes for a given chain and token pair. Refer to the provided code snippet for implementation details.
* **Firm quote:** Use `GET /firm-quote` to obtain transaction data necessary to execute swaps. Detailed code snippets are available for guidance.
