# CrossChain Swap API

## Overview

The Native Bridge API enables seamless cross-chain token bridging and swapping across multiple blockchain networks, utilising the same reliable Native Swap Engine and Native Credit Pool for superb pricing and performance.

## Endpoints

Our API enables users to fetch cross-chain bridge quotes, generate transaction data for executing on-chain bridge transactions via Native's infrastructure, and manage their bridging lifecycle. The intelligent routing algorithm ensures users receive the best possible market prices for optimal cross-chain trading outcomes.

* **Indicative quote**: Use `GET /bridge/indicative-quote` to obtain fast, indicative quotes for bridging tokens across chains. This endpoint provides pricing information without generating calldata, making it ideal for displaying estimated rates to users before they commit to a transaction.
* **Firm quote**: Use `GET /bridge/firm-quote` to obtain executable transaction data necessary to initiate cross-chain bridge transactions. This endpoint returns the complete calldata, target contract address, and all necessary parameters for executing the bridge transaction on the source chain.
* **Transaction status**: Use `GET /bridge/tx-status` to check the current status of a bridging transaction. This endpoint allows you to monitor the progress of your bridge request from initiation through completion, refund, or claim.
* **Transaction history**: Use `GET /bridge/tx-history` to retrieve a paginated list of all bridge transactions for a specific user address. This endpoint enables users to view their complete bridging history across different chains.
* **Refund**: Use `GET /bridge/refund` to request a refund for a bridging transaction that has failed or cannot be completed. This endpoint initiates the refund process for tokens that were locked but not successfully bridged.
