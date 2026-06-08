---
hidden: true
---

# API Guide

## 1. Authentication

Native requires the `api_key` header to be included in every request to authenticate that the request is coming from trusted users. We also offer flexible authentication methods to meet your needs.

## 2. Integration methods

We allow market makers to integrate using [**REST API**](../mm-pricing-and-signing-api/)

## 3. Liquidity sources

### **(Recommended) Native Credit Pools**

Market makers can utilize the liquidity in Native Credit Pools to facilitate swaps without needing to deploy capital directly on each chain. By providing collateral, market makers receive credit that can be used for trading and quoting. This setup allows quoting sizes to exceed the actual collateral provided.

### (Coming Soon) Market maker's own inventory

Native also allows market makers to use their own inventory to facilitate swaps. This requires setting up a dedicated pool smart contract.

To do this, please provide the following information to the Native team:

* **Pool name**: The name you'd like to give your pool.
* **Chain and list of pairs**: The chain and the pairs to quote, along with the corresponding token addresses.
* **Owner address**: The owner of the pool where ownership will be transferred.
* **Treasury address**: The address where the fund is stored.
* **Signer Address**: The key you'll use to sign quotes off-chain for your market maker. Provide the public key (address) and ensure the private key is stored securely. You'll need to programmatically sign quotes with the private key later on (in the `signQuote` API).

After the pool is created, the Native team will provide an API key and the pool address. To fund the market maker pool, you will need to grant token allowance to the pool from the treasury address. Quoting size can be more than the actual collateral being put in.
