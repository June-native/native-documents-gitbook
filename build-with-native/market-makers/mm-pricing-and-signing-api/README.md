---
description: Endpoints for Pricer and Signer
---

# MM Pricing and Signing API

## 1. Authentication

Native requires the `api_key` header to be included in every request to authenticate that the request is coming from trusted users. We also offer flexible authentication methods to meet your needs.

## 2. Integration methods

We allow market makers to integrate using **REST API**

## 3. **Native Credit Pools**

Market makers can utilize the liquidity in Native Credit Pools to facilitate swaps without needing to deploy capital directly on each chain. By providing collateral, market makers receive credit that can be used for trading and quoting. This setup allows quoting sizes to exceed the actual collateral provided.

Native relies on seamless API integration with market makers to ensure accurate off-chain pricing. HTTPS protocol and authentication are required for all endpoints. Responses should be in JSON format, accompanied by appropriate status codes.

## Get Started

To fully integrate with Native and leverage its capabilities, you need to implement the following API endpoints:

1. [**Orderbook**](orderbook.md)**:** This endpoint allows Native to estimate prices for different order sizes.
2. [**Firm-Quote**](firm-quote.md)**:** This endpoint enables Native to obtain a precise price from you for a given token and amount.
3. [**Sign-Quote**](sign-quote.md)**:** This endpoint allows you to validate and sign the order for on-chain execution.
