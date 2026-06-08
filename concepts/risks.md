# Risks

## Smart contract risk

Although our smart contracts have undergone audits by third-party firms, it is theoretically possible for vulnerabilities to exist. Furthermore, integrated third-party projects, including utilities and listed token projects, may also present risks associated with smart contracts.

Mitigation:

* Engaging multiple professional third-party firms to audit smart contracts significantly mitigates the risk of vulnerabilities. Audit reports are available \[here].
* Additionally, we implement a bug bounty program to incentivize individuals to identify vulnerabilities in our live code, serving as an additional measure to filter out potential issues.
* We meticulously evaluate any new tokens or utilities with which we choose to list and utilize, collaborating exclusively with those that have undergone auditing and demonstrated a proven track record of robust security.

## Credit-pool liquidity market risk

{% hint style="info" %}
The risks in this section apply to the credit-pool / RFQ liquidity model, which is now delivered through [Native Relay](../modules/native-relay.md). For background on the underlying mechanics, see the [Legacy (V2)](../legacy-v2/about-native-v2.md) section — including [Total Available Liquidity](../legacy-v2/total-available-liquidity.md), [Health Ratio](../legacy-v2/health-ratio.md), and [Base and Listed Assets](../legacy-v2/base-and-listed-assets.md).
{% endhint %}

### Extremely Pricing Volatility for Base and Listed Tokens

Price volatility is a risk for both the base and listed tokens. Market fluctuations can lead to the following outcomes:

**USD Value Loss of Deposited Assets:** The value of your deposited assets may decline in terms of USD due to the price drops.

**Risk of Bad Debt:** In highly volatile or extreme market conditions, the liquidation mechanisms may struggle to execute swiftly. This lag can occur due to rapidly falling token prices or a congested blockchain network, resulting in uncollateralized positions and potential bad debt within the system.

Mitigation:

* Several automated risk monitoring mechanisms have been implemented throughout the system to maintain operational integrity.
* Conservative collateralization thresholds have been established to provide a sufficient buffer to counteract market volatility.
* Additionally, high-frequency, dynamic adjustments to the lending parameters have been instituted to ensure the system can promptly respond to any changes in market conditions.

### TAL for Listed Tokens Unresponsive to Condition Change

TAL, which establishes the quoting limit in base tokens, may not promptly adapt to fluctuating market conditions. This situation poses a significant risk in instances where the market health of a listed token declines sharply, particularly during periods of rapid liquidity or value deterioration.

Should the adjustments to TAL be delayed or not adequately responsive to these alterations, the protocol might inadvertently overestimate the liquidity of the listed token. Consequently, this increases the probability of PMM positions that cannot be settled without incurring a loss, ultimately resulting in a loss of base tokens.

Mitigation:

* Collateral in the form of liquid tokens is required for all PMMs to ensure that losses are covered at a specified level.
* An automated and rapid assessment of the health ratio of the listed assets has been implemented to adjust TAL parameters in a timely manner.
