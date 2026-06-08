# FirmQuote Swap APIs

## Overview

The Native FirmQuote Swap API offers a robust and user-friendly toolkit for seamless integration with Native Swap across various blockchain networks, including Ethereum, and others coming in near future. Our API enables you to leverage Request-For-Quote (RFQ) pricing from all authorized professional market makers on Native Swap Engine and place orders without incurring additional network (gas) fees.

## **Endpoints**

Our API enables users to fetch token swap prices and generate transaction data for executing on-chain swaps via Native's infrastructure. The intelligent routing algorithm ensures users receive the best possible market prices for optimal trading outcomes.

* **Orderbook:** Use `GET /orderbook` to check Native price levels for all supported pairs. This endpoint is essential for any aggregator or solver.
* **Indicative quote:** Use `GET /indicative-quote` to obtain regular quotes for a given chain and token pair. Refer to the provided code snippet for implementation details.
* **Firm quote:** Use `GET /firm-quote` to obtain transaction data necessary to execute swaps. Detailed code snippets are available for guidance.
