---
description: wNLP – Native's LP Token
---

# wNLP

{% hint style="info" %}
wNLP is a legacy LP token used for users who supplied single-sided liquidity directly on public networks.
{% endhint %}

#### wNLP is:

* An ERC-20 wrapper, on top of NTLP, the legacy rebasing NativeLP tokens.
* Replacing NTLP, being the ONLY public facing, Native Liquidity Provider tokens.
* Fully ERC-20 compatible, meaning:
  * Free to transfer.
  * No deposit cooldown.
  * No early withdraw fee.
  * No yield rebasing. Instead, it is yield-bearing.
* Free to deposit/mint, no depositing fee, no performance fee, no withdraw fee.

#### When redeem/burn:

* **Instant Redeem**
  * LP may redeem up to the amount of real-time available inventory (usually at 5–10% reserve but may vary based on market volatility).
  * The inventory can be found in the CreditVault contracts. There is only one single CreditVault on each network.
  * Instant redeem comes with a small discount (0.1% \~ 2% depending on the asset, default 1%).
  * The discount ratio can be found in the wNLP contracts.
* **Delayed Redeem**
  * LP may submit a redeem request, and wait for a predefined time (1–8 days depending on the asset, default 3 days), then claim the redeemed asset.
  * The waiting time can be found in the wNLP contracts.
  * The redeem queuing contract address can be found in the wNLP contracts.

### Interface Document

{% file src="../../.gitbook/assets/wNLP-docs.pdf" %}

### Asset Pricing

wNLP's USD price can be calculated with:

* Getting the underlying asset using: `wNLP.underlying() -> address`
* Getting the underlying asset amount using LP amount: `wNLP.getNlpByWnlp(uint256 wNLPAmount) -> uint256`

Additionally, exchange rate with 1e18 precision can be calculated via:

```
wNLP.getNlpByWnlp(1e18) / 1e18
```

### Major Flow

#### Deposit and Mint

* **Name:** `depositAndWrap (0x57aa3d60)`
* **Input:**
  * `to (address)` — deposit recipient, useful when composed in external contracts.
  * `amount (uint256)` — the amount of underlying token to deposit.
* **Notes:**
  * Need to approve underlying token to wNLP contract before the deposit.
  * Once deposited, you will receive wNLP according to the real-time exchange rate, which can be fetched using `tokensPerNlp`.

#### Burn and Redeem — Instant

* **Name:** `instantRedeem (0x22928208)`
* **Input:**
  * `wNLPAmount (uint256)` — the amount of wNLP token to redeem.
  * `to (address)` — redeem recipient, useful when composed in external contracts.
* **Notes:**
  * Once redeemed, you will receive underlying token according to the real-time exchange rate, which can be fetched using `tokensPerNlp`.
  * Instant redeem comes with discounts, rate can be checked using `instantRedeemFeeBips`.
  * Some assets may not support instant redeem, please check `instantRedeemEnabled`.

#### Burn and Redeem — Queuing

> ℹ️ Use `withdrawQueue` to check the withdrawing contract address, the contract to call for requesting redeem. Each wNLP has their own queuing contract.

**Request**

* **Name:** `requestWithdrawal (0x9ee679e8)`
* **Input:**
  * `wNLPAmount (uint256)` — the amount of wNLP token to redeem.
* **Notes:**
  * You will need to approve wNLP to the queuing contract first.
  * Once requested, your wNLP will be transferred to the queuing contract.
  * If you wish to update the request amount, you need to first cancel the current withdraw request. Cancelling withdrawal will return the previously submitted wNLP.
  * The queuing time can be checked using `withdrawalWindow`.
  * As yield distribution is global, wNLP in the queue will still accumulate yield. However, the withdrawQueue contract is designed to calculate and record the amount of underlying tokens a user shall receive at the time of the request. Any additional yield generated during the queue will be collected to the Native official fee wallet.
    * To check the receiving amount:
      * **Before submitting:** use `wNLPAmount * exchangeRate` to calculate and preview in realtime based on user input.
      * **After submitting:** use the `withdrawalRequests(address _userAddress)` view function to query:
        * `wNLPAmount (uint256)` — the amount of wNLP in queue
        * `claimableTime (uint256)` — earliest claimable timestamp
        * `underlyingSnapshot (uint256)` — the receiving amount of underlying token

**Claim**

> ℹ️ wNLP in queue will not receive any yield. The receiving amount in underlying token will be calculated and recorded during the withdraw request. In claim, only the recorded amount will be transferred to the user; any excess amount created by subsequent exchange rate updates will be consolidated to fee treasury for future distributions.

* **Name:** `claimWithdrawal (0x6e66d84a)` or `claimWithdrawalTo (0x2aafb1f0)`
* **Input:**
  * `to (address)` — redeem recipient, useful when composed in external contracts.
* **Notes:**
  * You may only claim the full requested amount of wNLP submitted in queue.
  * You will receive the amount of underlying token with the exchange rate at the time of `requestWithdrawal`; any additional rebase after the request will not contribute to more claimable tokens.
    * The amount can be fetched using `withdrawalRequests(address _userAddress).underlyingSnapshot`.

#### Burn and Redeem — Instant & No Fee

* **Name:** `unwrapAndRedeem (0x93ab4051)`
* **Input:**
  * `wNLPAmount (uint256)` — the amount of wNLP token to redeem.
  * `to (address)` — redeem recipient, useful when composed in external contracts.
* **Notes:**
  * ⚠️ This is a privileged, no-fee instant redeem designed for specific use and requires pre-communication and whitelisting.
  * Instant redeem is subject to the limit of real-time cash inventory. The inventory can be found in the CreditVault contracts. There is only one single CreditVault on each network.
