---
description: EVM Vaults That Custody Assets and Enforce Settlement
---

# Native Pool

**Native Pool** is the asset layer of the Native stack: a set of distributed smart contracts (EVM vaults) deployed across chains that **custody user assets and enforce settlement**. While [Native Core](native-core.md) finalizes pricing and matching, Native Pool anchors the actual funds and bridges them to and from the chain where trading happens.

It is what gives users a **single, unified balance** to trade with, regardless of where their assets originated.

### What It Does

* **Custody & settlement:** vault contracts (e.g. `DepositWithdrawVault`) accept deposits, lock assets, and release them on withdrawal — enforcing settlement on each supported chain.
* **Unified capital experience:** one balance to trade across all markets; no per-pair or per-chain fragmentation.
* **Multi-chain deposits:** deposit from supported networks (Ethereum first, expanding to BNB Chain, Base, Arbitrum, and later non-EVM chains) and receive credit on Native Core.
* **Loan-based liquidity:** the pool model is designed so liquidity providers face minimal market risk while their capital is actively utilized.

### Deposit & Withdraw Pipeline

Native Pool works with an accounting service and an independent **signer network** to keep on-chain balances in sync with deposits and withdrawals:

* **Deposit:** funds are locked in the vault on the source chain; the accounting service provisionally credits the user on Native Core, then finalizes once the source chain reaches safe finality (rolling back if confirmation fails).
* **Withdraw:** the user's balance is locked on Native Core; signers co-sign the release on the destination chain; the balance is deducted once the release is confirmed (with admin fallback if the release fails).

### Related Concepts

* [Unified Balance](../concepts/unified-balance.md)
* [Deposit & Withdraw](../concepts/deposit-and-withdraw.md)
