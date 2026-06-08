# Risks

## Smart contract risk

Although our smart contracts have undergone audits by third-party firms, it is theoretically possible for vulnerabilities to exist. Furthermore, integrated third-party projects, including utilities and listed token projects, may also present risks associated with smart contracts.

Mitigation:

* Engaging multiple professional third-party firms to audit smart contracts significantly mitigates the risk of vulnerabilities. Audit reports are available \[here].
* Additionally, we implement a bug bounty program to incentivize individuals to identify vulnerabilities in our live code, serving as an additional measure to filter out potential issues.
* We meticulously evaluate any new tokens or utilities with which we choose to list and utilize, collaborating exclusively with those that have undergone auditing and demonstrated a proven track record of robust security.

## Pool liquidity market risk

Price volatility is a risk for Native Pool tokens. Market fluctuations can lead to the following outcomes:

**USD Value Loss of Deposited Assets:** The value of your deposited assets may decline in terms of USD due to the price drops.

**Risk of Bad Debt:** In highly volatile or extreme market conditions, the liquidation mechanisms may struggle to execute swiftly. This lag can occur due to rapidly falling token prices or a congested blockchain network, resulting in uncollateralized positions and potential bad debt within the system.

Mitigation:

* Several automated risk monitoring mechanisms have been implemented throughout the system to maintain operational integrity.
* Conservative collateralization thresholds have been established to provide a sufficient buffer to counteract market volatility.
* Additionally, high-frequency, dynamic adjustments to the lending parameters have been instituted to ensure the system can promptly respond to any changes in market conditions.
