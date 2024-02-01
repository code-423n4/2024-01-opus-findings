# Opus Protocol - Audit Analysis

- [Opus Protocol - Audit Analysis](#opus-protocol---audit-analysis)
  - [Project Description](#project-description)
    - [**Opus Protocol**](#opus-protocol)
  - [**Key Smart Contract**:](#key-smart-contract)
    - [**Absorber Module**](#absorber-module)
      - [Overview](#overview)
      - [Key Functions](#key-functions)
      - [Liquidity Provision Mechanisms](#liquidity-provision-mechanisms)
      - [Rewards Distribution](#rewards-distribution)
      - [Operational Requirements](#operational-requirements)
      - [Emergency Mechanism](#emergency-mechanism)
      - [Risk Analysis and Security Considerations](#risk-analysis-and-security-considerations)
      - [System Design Security](#system-design-security)
    - [**Purger Module**](#purger-module)
      - [Overview](#overview-1)
      - [Key Functions](#key-functions-1)
      - [Liquidation Mechanics](#liquidation-mechanics)
      - [Liquidation Penalty](#liquidation-penalty)
      - [Compensation](#compensation)
      - [Risk Analysis and Security Considerations](#risk-analysis-and-security-considerations-1)
        - [Economic Risks](#economic-risks)
    - [**Abbot Module**](#abbot-module)
      - [Overview](#overview-2)
      - [Key Concepts](#key-concepts)
      - [Key Functions](#key-functions-2)
      - [Operational Mechanics](#operational-mechanics)
        - [Opening a Trove](#opening-a-trove)
        - [Depositing Collateral](#depositing-collateral)
        - [Withdrawing Collateral](#withdrawing-collateral)
        - [Forging Synthetic](#forging-synthetic)
        - [Melting Synthetic](#melting-synthetic)
      - [Risk Analysis and Security Considerations](#risk-analysis-and-security-considerations-2)
    - [**Flash Mint Module**](#flash-mint-module)
      - [Overview](#overview-3)
      - [EIP-3156 Flash Loans](#eip-3156-flash-loans)
      - [Features](#features)
      - [Interaction with the Debt Ceiling](#interaction-with-the-debt-ceiling)
      - [Risk Analysis and Security Considerations](#risk-analysis-and-security-considerations-3)
    - [**Sentinel Module**](#sentinel-module)
      - [Overview](#overview-4)
      - [Key Functionalities](#key-functionalities)
      - [Gatekeeper Role](#gatekeeper-role)
      - [Emergency Mechanisms](#emergency-mechanisms)
      - [Risk Analysis and Security Considerations](#risk-analysis-and-security-considerations-4)
    - [**Gate Module**](#gate-module)
      - [Overview](#overview-5)
      - [Key Functions](#key-functions-3)
      - [Conversion Mechanism](#conversion-mechanism)
      - [Mitigation of Front-Running Vulnerabilities](#mitigation-of-front-running-vulnerabilities)
      - [Rounding Policies](#rounding-policies)
      - [Properties and Safeguards](#properties-and-safeguards)
    - [**Shrine Module**](#shrine-module)
      - [Overview](#overview-6)
      - [Key Functions](#key-functions-4)
      - [Timekeeping and Trove Management](#timekeeping-and-trove-management)
      - [Collateral and Synthetic Management](#collateral-and-synthetic-management)
      - [Interest Rates and Forge Fees](#interest-rates-and-forge-fees)
      - [Liquidations and Redistributions](#liquidations-and-redistributions)
      - [Recovery Mode and Emergency Mechanisms](#recovery-mode-and-emergency-mechanisms)
    - [**Equalizer Module**](#equalizer-module)
      - [Overview](#overview-7)
      - [Key Functions](#key-functions-5)
      - [Interaction with the Debt Ceiling](#interaction-with-the-debt-ceiling-1)
      - [Distribution of Income](#distribution-of-income)
    - [**Caretaker Module**](#caretaker-module)
      - [Overview](#overview-8)
      - [Key Functions](#key-functions-6)
      - [Shutdown Mechanism](#shutdown-mechanism)
      - [Redeeming Yin for Collateral](#redeeming-yin-for-collateral)
      - [Withdrawing Collateral from Troves](#withdrawing-collateral-from-troves)
  - [**Key Smart Contract**:](#key-smart-contract-1)
    - [**Controller Module**](#controller-module)
      - [Overview](#overview-9)
      - [Operational Dynamics](#operational-dynamics)
      - [Control Mechanism](#control-mechanism)
    - [**Seer Module**](#seer-module)
      - [Overview](#overview-10)
      - [Oracle Integration](#oracle-integration)
      - [Key Functions](#key-functions-7)
      - [Price Calculation for Yang](#price-calculation-for-yang)
      - [Price Update Triggers](#price-update-triggers)
      - [Supported Oracles](#supported-oracles)
  - [Approach](#approach)
  - [Audit Practices for this Project](#audit-practices-for-this-project)
      - [Codebase Quality](#codebase-quality)
  - [Systemic \& Centralization Risks](#systemic--centralization-risks)
  - [Recommendations](#recommendations)
  - [Final Thoughts](#final-thoughts)



## Project Description
### **Opus Protocol**
Opus Protocol is a cross-margin autonomous credit protocol that offers dynamic loan management based on each user's collateral profile. The whole diagram for the smart contracts is: ![image](https://4180423341-files.gitbook.io/~/files/v0/b/gitbook-x-prod.appspot.com/o/spaces%2FBTWxg1bdHQ15qxTg6SOE%2Fuploads%2FfsrC6vc3McLsuAx73WsC%2FOpus%20Architecture-Interactions.png?alt=media&token=0b6a66d4-4715-4c5d-8ea9-0eab8a90d99f)

Key features and highlights:
- **Dynamic Interest Rates**: Interest rates adjust based on market conditions.
- **Multi-Layer Liquidation**: A multi-layered liquidation system enhances protocol solvency.
- **Minimal Human Intervention**: Rates and loan-to-value ratios are autonomously adjusted.

## **Key Smart Contract**:
### **Absorber Module**

#### Overview

The Absorber is a component of the Opus protocol, designed as a stability pool that enables yin holders to contribute to liquidation processes, termed as "absorptions." By participating in the Absorber, users can:

- Share in the rewards from absorptions.
- Earn yield from debt surpluses like interest and forge fees in the form of yin.
- Receive additional rewards distributed by the protocol or external entities.

The contract introduces specific terminologies like "provide" and "remove" for depositing and withdrawing yin to avoid confusion with similar operations in other parts of the protocol.

#### Key Functions

- **Provide**: Users deposit yin to receive internal shares, representing claims on remaining yin, entitlement to absorbed assets, and rewards.
- **Remove**: Allows withdrawal of yin under certain conditions, after a removal request has been made and approved.
- **Request**: Enables users to initiate a withdrawal process for their provided yin.
- **Reap**: Permits the withdrawal of absorbed assets and rewards without removing the provided yin.
- **Update**: Updates the Absorber on new absorptions, adjusting the accounting for absorbed assets.

#### Liquidity Provision Mechanisms

- Epochs and Shares: Tracks liquidity in epochs, allocating shares based on provided yin. Epoch transitions are triggered under specific conditions.
- Initial Shares: Mitigates first depositor front-running by "minting into oblivion" a minimum number of shares at the start of each epoch.

#### Rewards Distribution

Supports the distribution of whitelisted rewards, vested based on internal shares, and distributed during user interactions or absorptions.

#### Operational Requirements

- Minimum Shares: Ensures the Absorber operates with sufficient liquidity for liquidations, setting a minimum share threshold to mitigate overflow issues.
- Removing Liquidity: Enforces preconditions for liquidity removal to maintain stability and prevent exploitation.

#### Emergency Mechanism

Includes a kill function to pause liquidity provision and reward distribution in emergencies, while still allowing liquidity removal.

#### Risk Analysis and Security Considerations

#### System Design Security

- **Front-Running Mitigation**: The mechanism of "minting into oblivion" initial shares at the start of each epoch deters potential front-running, increasing the cost of such attacks.
- **Epoch Management**: The epoch-based system ensures that liquidity provision and rewards distribution are managed effectively, reducing the risk of manipulation.
- **Overflow Prevention**: By setting a minimum threshold for operational shares, the system guards against potential overflows in asset and reward distribution.


### **Purger Module**

#### Overview

The `Purger`  module serves as a crucial interface within the Opus protocol's multi-layered liquidation system. It enables the liquidation of unhealthy troves to maintain the protocol's solvency, utilizing either the user's own yin or the yin pooled in the Absorber. This dual-method approach ensures flexibility and efficiency in handling liquidations. Liquidation occurs when a trove's Loan-to-Value (LTV) ratio is higher than a certain threshold, indicating it is undercollateralized, and absorption is another form of liquidation where surplus assets are distributed to the stability pool or other troves.

The contract includes several components for access control and reentrancy protection, interfaces for interacting with other related modules (`Shrine`, `Sentinel`, `Absorber`, `Seer`), and a set of parameters that define liquidation behavior, penalties, and compensation for users who perform liquidations.

#### Key Functions

- **Liquidate**: Allows the caller to use their own yin to pay down the debt of an unhealthy trove, receiving the trove's collateral and a liquidation penalty as a reward.

![image](https://4180423341-files.gitbook.io/~/files/v0/b/gitbook-x-prod.appspot.com/o/spaces%2FBTWxg1bdHQ15qxTg6SOE%2Fuploads%2FIdXrRIxIRAOmECG9FEvJ%2Fimage.png?alt=media&token=50fc8413-1ad8-442c-a0f6-d575ed8574c4)

- **Absorb**: Facilitates the use of the Absorber's yin for liquidation, with the Absorber receiving the collateral and penalty, and the caller compensated for initiating the process.

![image](https://4180423341-files.gitbook.io/~/files/v0/b/gitbook-x-prod.appspot.com/o/spaces%2FBTWxg1bdHQ15qxTg6SOE%2Fuploads%2FtegWQDFczQlw7VNfYEW1%2Fimage.png?alt=media&token=cfcce67c-6004-488e-9b4b-72699d1a0ba8)


Both functions are designed to revert if the Shrine component of the protocol is not live, ensuring operations are halted during critical failures or maintenance periods.

#### Liquidation Mechanics

- **Priority**: Liquidation can be initiated once a trove's Loan-to-Value (LTV) ratio breaches its threshold. The penalty increases with the LTV's deviation from this threshold, capped at 12.5% to ensure debt coverage.
- **Absorption Conditions**: Absorption is specific to scenarios where the LTV exceeds a certain limit, allowing for a controlled and phased liquidation process.
- **Parameters**: Dynamic factors such as close amount, liquidation penalty, and compensation are key to the liquidation process, incentivizing participation while ensuring fairness and efficiency.

#### Liquidation Penalty

The penalty structure is critical in balancing the risk and reward for participants, with distinct calculations for the `liquidate` and `absorb` functions. A penalty scalar is introduced in the absorption process to adjust the incentivization dynamically, based on the protocol's needs and the prevailing market conditions.

#### Compensation

Compensation mechanisms are in place to encourage users to initiate the `absorb` process, set to the lower of 3% of the trove's collateral value or a fixed USD amount. This ensures that there is always an incentive to support the protocol's stability, even in varying market conditions.

#### Risk Analysis and Security Considerations

##### Economic Risks

- **Manipulation**: The dual liquidation approach could be exploited through market manipulation, where actors artificially trigger liquidation conditions.
- **Incentive Misalignment**: Overly generous compensation or penalties could distort user behavior, leading to undesirable economic outcomes.

### **Abbot Module**

#### Overview

The Abbot module serves as the central interface for users to interact with their troves within the protocol, facilitating the opening, management, and closure of troves. Its design ensures immutability and sequential issuance of trove IDs, enhancing the integrity and traceability of user interactions.

#### Key Concepts

- **Deposit and Withdraw**: These actions pertain to the collateral management within a trove, allowing users to secure or release their assets.
- **Forge and Melt**: These operations are related to the synthetic asset's lifecycle within a trove, including its creation (forge) and elimination (melt).

#### Key Functions

- **Open Trove**: Initializes a trove, assigning a unique ID to the caller, depositing collateral, and optionally minting synthetic debt. It's a foundational action that establishes a user's position within the protocol.
- **Close Trove**: Clears a trove's debt and returns all collateral to the owner, maintaining the trove's ID for potential future use. This function encapsulates the melt and withdraw processes.
- **Deposit**: Enables users to add collateral to a trove, increasing its security and potentially its borrowing capacity.
- **Withdraw**: Allows the trove's owner to retrieve collateral, subject to the maintenance of the trove's health, specifically its loan-to-value ratio.
- **Forge**: Facilitates the minting of synthetic assets against the trove's collateral, increasing the trove's debt.
- **Melt**: Involves the repayment of debt and the burning of the synthetic asset, reducing the trove's debt.

#### Operational Mechanics

##### Opening a Trove

The action of opening a trove is the entry point for users into the protocol's ecosystem, enabling them to deposit collateral and, if desired, mint synthetic assets. This process is critical for establishing the user's stake and capability within the protocol.

##### Depositing Collateral

Collateral depositing is a flexible operation, allowing for the enhancement of a trove's health and borrowing capacity. The protocol's design ensures that collateral management is secure and directly tied to the user's control.

##### Withdrawing Collateral

The withdrawal mechanism is designed with safeguards to ensure the trove's stability and the overall health of the protocol, such as maintaining minimum loan-to-value ratios and collateral values.

##### Forging Synthetic

The forging process introduces synthetic assets into the ecosystem, directly tied to the collateralized troves. This mechanism is central to the protocol's functionality, enabling users to leverage their collateral effectively.

##### Melting Synthetic

Melting represents the reduction of debt within a trove, a crucial aspect of maintaining balance and solvency within the protocol. This function allows for the dynamic management of debt positions, directly impacting the protocol's liquidity and stability.

#### Risk Analysis and Security Considerations

The design of the Abbot module incorporates several security measures to maintain the integrity of troves and the overall protocol. Key considerations include:

- **Sequential Trove IDs**: Enhances traceability and auditability of troves within the protocol.
- **Immutable Design**: Ensures that the core functionalities related to trove management remain consistent and reliable.
- **Collateral and Debt Management**: Incorporates safeguards to prevent unhealthy trove states, such as maintaining minimum loan-to-value ratios and ensuring debt coverage.


### **Flash Mint Module**

#### Overview

The `flash_mint` contract appears to implement a flash loan functionality compliant with EIP-3156 for Starknet. It allows for the minting of a synthetic asset represented by the contract `shrine`. Users can flash borrow up to 5% of the total Yin supply (`FLASH_MINT_AMOUNT_PCT`) within the constraints of the total debt ceiling. A reentrancy guard component is used to prevent reentrancy attacks during the flash loan process.

#### EIP-3156 Flash Loans

Flash loans are an innovative DeFi tool allowing for the borrowing of assets without upfront collateral, under the condition that the liquidity is returned within the same transaction block. The Flash Mint module's adherence to EIP-3156 ensures compatibility and standardization in its flash loan offering.

#### Features

- **Fee-less Transactions**: Unlike many flash loan services that charge a fee, this module allows for fee-less borrowing, reducing the cost barrier for various strategies and operations.
- **Circulating Yin Supply Percentage**: The maximum borrowable amount is pegged to a percentage of the circulating yin supply, introducing a dynamic cap that scales with the ecosystem's liquidity.
- **Adjustable Maximum Percentage**: The cap on the borrowable amount, while set as a constant, can be modified through contract redeployment, providing a mechanism for adaptability to changing protocol needs or governance decisions.

#### Interaction with the Debt Ceiling

The Flash Mint module is designed with a crucial mechanism to interact with the protocol's debt ceiling, ensuring that the flash loan operations do not destabilize the system's solvency.

- **Temporary Debt Ceiling Adjustment**: In cases where a flash loan would breach the existing debt ceiling, the module is capable of temporarily adjusting the ceiling to accommodate the loan, ensuring the protocol's operational continuity.
- **Governance and Protocol Safety**: This interaction underscores the balance between offering flexible financial tools and maintaining the protocol's overall health and security.

#### Risk Analysis and Security Considerations

The introduction of flash loans, while adding utility and efficiency to the protocol, also brings specific risks and considerations:

- **Smart Contract Security**: Given the complexity of flash loan transactions, the module must be robust against reentrancy attacks, transaction ordering vulnerabilities, and other smart contract risks.
- **Systemic Risk Management**: The temporary adjustment of the debt ceiling for flash loans requires careful management to avoid systemic risks, ensuring that the protocol remains solvent and secure even in scenarios of extreme demand.
- **Governance and Adjustment Mechanisms**: The ability to adjust the maximum borrowable percentage through contract redeployment highlights the importance of governance processes in managing risk and adapting to evolving market conditions.

### **Sentinel Module**

#### Overview

The Sentinel module is designed as a central interface within the protocol, enabling other modules to interact with Gates without needing to know each Gate's specific address. It acts as an abstraction layer, simplifying access control and interactions involving collateral assets.

#### Key Functionalities

- **Abstraction of Gate Addresses**: By centralizing interactions through the Sentinel, the protocol enhances its modularity and reduces the complexity of module interactions.
- **Simplified Access Control**: The Sentinel module streamlines access control, requiring only the Sentinel to be approved by each Gate, thereby reducing the overhead for other modules.
- **Function Mimicry**: The Sentinel replicates the functions of Gates, adding an additional parameter for the collateral token's address to facilitate targeted interactions.

#### Gatekeeper Role

- **Collateral Token Management**: The Sentinel is responsible for adding collateral tokens (referred to as yang) to the Shrine, ensuring that a corresponding Gate has been deployed for each token.
- **Initial Minimum Deposit Enforcement**: To mitigate the risk of first depositor front-running, the Sentinel ensures an initial minimum deposit is made by the protocol for each new collateral token.
- **Supply Cap Enforcement**: The Sentinel monitors and ensures that the total amount of collateral tokens for a yang does not surpass its predefined supply cap, maintaining balance and preventing over-collateralization.

#### Emergency Mechanisms

- **Kill Gate Functionality**: The Sentinel is equipped with a `kill_gate()` function, allowing for the immediate halt of further deposits (`enter` function) for a specific Gate through the Sentinel. This measure is critical in emergency scenarios where halting new deposits while allowing withdrawals (`exit` function) is necessary for user protection and protocol stability.

#### Risk Analysis and Security Considerations

The Sentinel module's design incorporates several security measures to safeguard the protocol's integrity:

- **Centralized Interaction Point**: By serving as the primary interface for module-to-Gate interactions, the Sentinel reduces the attack surface associated with direct Gate access, centralizing and streamlining security efforts.
- **Front-Running Mitigation**: The enforcement of an initial minimum deposit for new collateral tokens addresses the risk of first depositor front-running, enhancing the fairness and security of the protocol's operations.
- **Supply Cap Compliance**: Monitoring and enforcing supply caps for collateral tokens prevent the risk of over-collateralization, maintaining the protocol's economic balance and stability.

### **Gate Module**

#### Overview

The Gate smart contract serves as an interface for users to deposit and withdraw assets in exchange for Yang, a credit system used within the Opus Autonomous Credit Protocol. The contract utilizes functions such as `enter` and `exit` to manage the asset and Yang balances. A Sentinel address is authorized to handle specific contract functions, while the Shrine address is associated with the credit system. This report provides an audit of the Gate smart contract, focusing on security, efficiency, and adherence to best practices.


#### Key Functions

- **Enter**: This function is responsible for accepting collateral tokens from users, transferring them to the Gate, and in return, issuing the corresponding amount of yang. The process encapsulates the conversion of collateral tokens to yang, reflecting the current valuation within the protocol.
- **Exit**: Conversely, the exit function enables users to convert their yang back into the original collateral tokens. The Gate module returns the collateral to the user, deducting the corresponding amount of yang from their balance.

#### Conversion Mechanism

The Gate module's approach to asset conversion does not rely on a fixed rate. Instead, it dynamically calculates the conversion rate based on the prevailing balance of collateral tokens within the Gate and the total amount of yang. This design allows for efficient redistribution of collateral via rebasing, where the value of each yang unit in terms of the underlying collateral can increase over time.

#### Mitigation of Front-Running Vulnerabilities

A notable aspect of the Gate module is its defense against the first depositor front-running vulnerability, common in mechanisms like ERC-4626. Rather than penalizing the first depositor, the protocol itself contributes a small initial deposit of collateral, which is converted to yang and dedicated to the Shrine. This preemptive measure shifts the burden from users to the protocol, ensuring fairness and security in the initiation of new collateral types.

#### Rounding Policies

Both the enter and exit functions implement rounding down as a default policy, favoring the protocol. This means that when users deposit collateral for yang, or withdraw collateral against their yang, the amounts are rounded down to the nearest whole number, ensuring minimal discrepancies in favor of the protocol's reserves.

#### Properties and Safeguards

The Gate module upholds several key properties to ensure consistency and reliability:

- **Monotonic Conversion Rate**: The conversion rate from yang to its underlying collateral is designed to be monotonic while the Shrine is operational, ensuring that it does not decrease over time.
- **Stable Conversion Rate**: The conversion rate is maintained during both enter and exit transactions, provided there is a non-zero balance of the collateral asset before and after the transactions, respectively.


### **Shrine Module**

#### Overview

The Shrine module serves as the central accounting hub for the synthetic ecosystem, managing the intricate balance of deposited collateral (yang) and minted synthetic (yin) across all user debt positions (troves) and the protocol itself. It ensures the accurate calculation and application of interest on each trove, maintains the valuation of collateral types (yangs), and integrates the multiplier values from the Controller module. Additionally, the Shrine module adheres to the ERC-20 standard for the synthetic it represents, reinforcing its role in the ecosystem's liquidity and exchange mechanisms.

#### Key Functions

- **Deposit**: Increases the yang balance for a specific trove, enhancing its collateralization.
- **Withdraw**: Decreases the yang balance for a trove, subject to maintaining the trove's health and collateralization requirements.
- **Forge**: Augments the debt of a trove while minting an equivalent amount of yin, ensuring the trove's continued health.
- **Melt**: Reduces a trove's debt by burning the corresponding amount of yin.
- **Seize**: Used in liquidation processes to adjust the yang balance of a trove without the need for it to remain healthy.
- **Redistribute**: In liquidation scenarios, this function equitably reallocates the debt and yang of an unhealthy trove among healthy troves.
- **Inject/Eject**: These functions manage the minting and burning of yin to and from addresses outside of the standard trove operations, typically used by other modules within the ecosystem.

#### Timekeeping and Trove Management

The Shrine module employs a discrete timekeeping system, defining intervals based on a constant time period. This system underpins the interest calculation mechanism, ensuring consistency and predictability in the accrual of debt over time. Troves are identified by unique IDs and are tracked through the Trove struct, which records their debt and interest accrual starting points.

#### Collateral and Synthetic Management

Yang represents the internal abstraction of collateral tokens, normalized to a precision of 18 decimal places, allowing for flexible and efficient collateral management. The Shrine's design ensures that once a collateral type is accepted and integrated as yang, it becomes a permanent fixture of the ecosystem, though it can be suspended under specific conditions to manage risk and maintain protocol health.

#### Interest Rates and Forge Fees

Interest on troves is calculated dynamically, combining the base rates of deposited yangs with multiplier values over time. This system allows for the nuanced management of debt costs, reflecting the varying risk profiles and market conditions of different collateral types. The forge fee mechanism serves as a stabilizing force, discouraging excessive minting of yin during periods of volatility or devaluation, thus protecting the protocol's stability.

#### Liquidations and Redistributions

The Shrine module outlines a multi-tiered liquidation approach, including searcher liquidations, absorptions, and redistributions, to manage trove health and protocol solvency. Redistributions play a key role in maintaining the ecosystem's equilibrium, reallocating collateral and debt among troves to ensure a fair and balanced distribution of risk and reward.

#### Recovery Mode and Emergency Mechanisms

In scenarios where the protocol's aggregate loan-to-value ratio breaches critical thresholds, the Shrine can enter recovery mode, adjusting operational parameters to safeguard solvency and encourage corrective user actions. The module also includes an emergency kill function, allowing for the decisive shutdown of user-facing operations in extreme circumstances.

### **Equalizer Module**

#### Overview

The Equalizer plays a pivotal role in maintaining the financial equilibrium of the Shrine by addressing the budget variances through debt surplus minting or deficit reduction. This module ensures the synthetic ecosystem remains sustainable, avoiding the pitfalls of perpetual debt cycles by aligning the total debt with the circulating synthetic currency (yin).

#### Key Functions

- **Equalize**: This crucial function addresses the Shrine's positive budget scenarios by minting debt surpluses as yin, which are held by the Equalizer. This process is vital for ensuring the total amount of minted yin is reflective of the actual debt within the Shrine, thereby maintaining the system's balance.
- **Normalize**: In cases where the Shrine experiences a debt deficit, this function allows for the reduction of such deficits by burning the caller's yin, directly reducing the overall debt within the system.
- **Allocate**: Central to the distribution of income, this function disperses the yin held by the Equalizer to predefined recipients, as dictated by the Allocator module. This mechanism facilitates the fair and efficient distribution of surplus yin generated from debt surpluses.

#### Interaction with the Debt Ceiling

A critical aspect of the Equalizer's operation is its ability to temporarily adjust the debt ceiling to accommodate the minting of debt surpluses. This ensures that the system can continue to mint yin as needed, without being hindered by a pre-existing debt ceiling, thus maintaining the protocol's liquidity and operational fluidity.

#### Distribution of Income

The Equalizer not only addresses the Shrine's budgetary needs but also plays a significant role in the distribution of income within the ecosystem. By minting yin to its address and allowing for its subsequent allocation, the Equalizer ensures that the benefits of debt surpluses are equitably shared among designated recipients, as per the Allocator module's directives.

### **Caretaker Module**

#### Overview

The Caretaker module is designed to manage the protocol's deprecation, especially the Shrine, ensuring a structured and equitable process for yin holders to claim collateral corresponding to their holdings. It embodies the protocol's emergency shutdown mechanism, safeguarding the interests of participants by allowing for the redemption of yin and the withdrawal of collateral from troves post-shutdown.

#### Key Functions

- **Shut**: This function initiates the protocol's shutdown, permanently disabling user actions within the Shrine. It transfers a predetermined percentage of the collateral backing the total debt of all troves to the Caretaker, setting the stage for collateral claims by yin holders.
- **Release**: Post-shutdown, this function enables trove owners to withdraw any remaining collateral from their troves, accounting for the system-wide redistribution that occurs during the shutdown process.
- **Reclaim**: Allows yin holders to burn their yin in exchange for a proportionate share of the collateral held by the Caretaker, up to the total debt of all troves at the time of shutdown.

#### Shutdown Mechanism

The execution of the `shut` function marks the beginning of the protocol's orderly deprecation. All debts within troves become irreparable, and the necessary collateral to back the total forged debt is moved to the Caretaker. This system-wide redistribution effectively finalizes each trove's collateral position, aligning it with the protocol's overall liquidity state at the time of shutdown.
![image](https://4180423341-files.gitbook.io/~/files/v0/b/gitbook-x-prod.appspot.com/o/spaces%2FBTWxg1bdHQ15qxTg6SOE%2Fuploads%2FRPkcCXm8GXcXKDQdx4Vn%2Fimage.png?alt=media&token=c57bf1bd-ee0f-4a1d-9313-45a2f02205ca)

#### Redeeming Yin for Collateral

Following the shutdown, yin holders are entitled to exchange their yin for a share of the collateral now under the Caretaker's management. This process is governed by the total debt at shutdown, with each yin's redemption value being proportional to the remaining reclaimable yin supply.

#### Withdrawing Collateral from Troves

Trove owners are permitted to withdraw the residual collateral from their troves post-shutdown. Given the equitable redistribution of collateral during shutdown, the remaining collateral available for withdrawal will reflect the trove's share of the total collateral post-redistribution.


## **Key Smart Contract**:

### **Controller Module**

#### Overview

The Controller module serves as an autonomous regulatory mechanism within the synthetic ecosystem, tasked with adjusting a global interest rate multiplier to minimize the deviation of yin's market price from its peg. By modulating the multiplier, the Controller indirectly influences trove owners' decisions regarding debt creation and repayment, aiming to stabilize the synthetic's value.

#### Operational Dynamics

The core principle of the Controller is to respond to fluctuations in yin's market price with adjustments to the interest rate multiplier:

- **Above Peg**: When yin's price exceeds the peg, the multiplier decreases below 1, making debt cheaper and encouraging the forging of new yin. This expansion of yin supply exerts downward pressure on its market price.
- **Below Peg**: Conversely, if yin's price falls below the peg, the multiplier increases above 1, discouraging new debt creation and incentivizing debt repayment. This contraction of yin supply aims to elevate its market price.

#### Control Mechanism

The Controller employs a Proportional-Integral (PI) control strategy, augmented with a nonlinear function to fine-tune its responsiveness to price deviations. This nonlinearity ensures that minor fluctuations are met with proportionate adjustments, while significant deviations prompt more substantial interventions to preemptively counteract potential market imbalances.

### **Seer Module**

#### Overview

The `seer` module operates as the linchpin in aggregating and validating price data for the synthetic ecosystem, interfacing with various oracle modules to ascertain the current market prices of collateral tokens (yangs). By standardizing the interaction through adapter modules that conform to the IOracle interface, the Seer ensures compatibility and reliability in price data acquisition.

#### Oracle Integration

To incorporate an oracle into the ecosystem, an adapter module adhering to the IOracle interface must be implemented. This approach standardizes the diverse functionalities and outputs of different oracles, allowing the Seer to seamlessly obtain price data from multiple sources without being encumbered by the intricacies of individual oracle implementations.

#### Key Functions

- **Update Prices**: This critical function collates the most recent and valid prices for each yang, updating the Shrine with the latest data. The process involves iterating through the configured oracles according to their priority and selecting the earliest valid price available.

#### Price Calculation for Yang

It is crucial to distinguish between the price of a yang and its underlying collateral token. The Seer is tasked with adjusting the raw oracle price data to reflect the yang's value, taking into account the conversion rate between a yang and its underlying collateral. This ensures that the price data accurately represents the yang's market value within the protocol's context.

#### Price Update Triggers

Price updates within the Seer module can be initiated under two conditions:

1. **Time-Based**: A predefined duration elapses since the last price update attempt, ensuring regular price refreshment regardless of market activity.
2. **Access-Based**: Specific entities granted the privilege can invoke price updates, circumventing the time-based restriction. This feature is particularly relevant during redistributions, where immediate price adjustments are necessary to reflect changes in yang conversion rates following trove rebalances.

#### Supported Oracles

At its inception, the protocol will primarily utilize the Pragma oracle, with plans to integrate additional fallback oracles as the Starknet ecosystem expands. This strategy ensures redundancy and reliability in price data, safeguarding against single points of failure and providing a robust foundation for the protocol's valuation mechanisms.


## Approach
The audit focused on reviewing and analyzing the core functionality and security of the Opus Protocol's smart contracts. Specific areas of focus included:
1. **Contract Logic and State Management**
2. **Interest Rate Adjustment Mechanism**
3. **Liquidation System and Stability Pool**

---

## Audit Practices for this Project
- **Automated Scanning**: Leveraged automated tools to identify common vulnerabilities.
- **Manual Review**: Conducted line-by-line review of smart contracts to understand the system's logic and potential edge cases.

#### Codebase Quality
The codebase demonstrates a coherent structure and systematic modularization. Each module encapsulates specific protocol functions, promoting clear delineation of responsibilities and readability.

## Systemic & Centralization Risks
The protocol relies on an honest admin for the integrity of its access control mechanisms, posing a centralization risk.

## Recommendations
1. **Fallback Oracle**: Implement a fallback oracle to ensure the robustness of the price data feed.
2. **Interest Accrual**: Consider optimizing the interest accrual process to account for the delay in interest application during shutdown.



## Final Thoughts
Overall, the Opus Protocol's smart contract suite is well-organized, featuring distinct components that handle specific aspects of the protocol logic. The autonomous nature of adjustments in response to market conditions shows significant promise for decentralized finance. To enhance trust and protocol resilience, a movement towards decentralized governance structures is recommended, coupled with the implementation of a fallback oracle system for improved data reliability.



### Time spent:
32 hours