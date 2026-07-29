---
hidden: true
---

# Pool Rewards

Liquidity providers who deposited in Native Pool receive rewards as their asset is utilise by Native Relay for trade settlements.

### Where is the reward coming from?

Native Pool LP rewards are generated from Native Relay trading fee on Native Core when relying trades from swap aggregators, wallet, and other trading venues.

Reference the fee documents for the Native Core fee rates:

{% content-ref url="swap-fees.md" %}
[swap-fees.md](swap-fees.md)
{% endcontent-ref %}

Note that some markets on Native Core may have special fee rates applied. Fee rate can be viewed from each of the "Market" page: [http://app.native.org/markets/](http://app.native.org/markets/)

<figure><img src="../.gitbook/assets/image (79).png" alt=""><figcaption></figcaption></figure>

### How rewards are calculated?

`reward = bid side volume * taker fee rate * rebate ratio`&#x20;

Rewards are calculated and distributed every 24 hours. Based on the token bid side volume from the last 24 hours window.

Bid side volume: trades that takers are swapping **to** to the token. For example, if the token is WETH, its volume is calculated by the trades that are swapping from another token **to** WETH.
