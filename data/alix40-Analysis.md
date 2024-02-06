## Centralization Risks

In the realm of emerging protocols, a prevalent concern is the initial high degree of centralization. Despite understanding the necessity of this phase, it's crucial to emphasize the inherent centralization risks, urging mitigative action in subsequent development stages. Achieving full decentralization is paramount for the protocol's longevity and trustworthiness.

At the launch phase, our analysis highlights significant centralization risks attributed to the extensive control wielded by the protocol's administrators. The identified critical capabilities of the Admin of the Shrine include:

- Adjustment of the shrine budget Without Yin Transactions: Admins possess the ability to modify the shrine budgets without the requisite of collecting or withdrawing fees through the `adjust_budget()` function. Misuse of this power undermines the protocol's credibility, potentially eroding user trust.
- Unrestricted Yin Minting: The facility for an admin to mint an unlimited quantity of Yin, through the `inject()` function and subsequent adjustment of the debt ceiling, presents a grave risk. This could lead to inflationary pressures and devaluation.
- Arbitrary Yin Burning: The `eject()` function allows admins to burn Yin from users' balances without consent. This unilateral control further centralizes power and raises concerns about user autonomy and security.
- Arbitrary Trove Liquidation: A particularly alarming power is the ability of an administrator to manipulate the yang price to exceedingly low levels (e.g., 1 USD) and subsequently liquidate troves with minimal Yin holdings. This action could result in the loss of collateral for the protocol's users, representing a substantial risk.

Addressing these centralization risks is paramount to developing a trust-based, decentralized ecosystem and avoiding scenarios that could lead to the devaluation or crash of the protocol's assets.

## Systemic Risks and Mitigation Strategies
#### Price Manipulation Attacks

Price manipulation attacks have been identified as one of the top reasons for DeFi hacks in 2023. This risk is particularly pronounced for assets with low trading volume and market cap, whose prices can be easily manipulated on decentralized exchanges. It is of utmost importance for developers to prioritize safeguards against such vulnerabilities.
The current simplistic checks for prices received from the Pragma oracle may present an ideal attack vector for sophisticated adversaries. We recommend implementing additional validation measures for incoming prices, such as verifying against a predefined range or using cumulative prices to assess new price validity.

#### Oracle Manipulation Attacks

The protocol's reliance on Pragma as the sole price source, coupled with a lack of proper error handling mechanisms (e.g., try-catch), positions Pragma as a potential single point of failure at launch. An attacker gaining control over the Pragma oracle could inflict significant losses on the protocol by manipulating asset prices, enabling them to liquidate a majority of the troves.


## Auditor Experience

Dedicating over 80 hours to this contest since January 18, 2024, I navigated my first audit within the Cairo and StarkNet ecosystems. With no prior experience in Cairo, I allocated the initial 8 hours to mastering the language through resources like the Code4rena bootcamp, the StarkNet book, and the Cairo book. My background in Solidity provided a solid foundation, making the transition to auditing Cairo code comparatively straightforward.

Cairo, as a programming language, currently falls short in offering developers essential functionalities such as signed integers, necessitating extensive boilerplate code. This observation underscores the need for Cairo's evolution to address these limitations, enhancing developer efficiency and protocol robustness.


### Time spent:
80 hours