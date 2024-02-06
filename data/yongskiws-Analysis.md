

# Opus Audit Analysis
![images (2)](https://github.com/yongsxyz/Audit-Analysis/assets/39027215/429d1bb0-32fa-42b4-8b79-1a5a5ff9dff2) 

The Opus Protocol is a cross-margin credit protocol with autonomous monetary policy. It consists of modules such as Gate, Sentinel, Purger, Absorber, Controller, Equalizer, Seer, Flash Mint, and Caretaker. Each module plays a specific role in maintaining protocol stability, from managing collateral tokens to automatic interest rate adjustments and handling liquidation of unhealthy troves. With the combination of these modules, Opus creates a dynamic credit ecosystem with adjustable risk parameters.

# Overview

Structuring the code is crucial for conducting a thorough analysis of the code base. This enables auditors to predict the level of contract difficulty, identify critical contracts, and determine security-critical features such as payment functions or assembly usage. Additionally, this approach accurately measures the time required to perform a detailed audit and helps determine the cost of a project audit. To achieve efficiency, clarity, and improvements, it is essential to have a comprehensive view of the entire architecture. This enables the identification of the basic structure, relationships between , and key design patterns. By understanding the big picture, we can analyze the complexity of the code and reveal its strengths and potential for improvement with confidence.


# Approach to Auditing

- We began by thoroughly reviewing and read docs Opus procotol the provided documentations, as well as reviewing the video provided. Here, we made note of important information, noted down unclear parts and overall tried to fully understand how the protocol should function. We also took notes of integrated and inherited protocols and mapped out possible risk areas.

- While we were studying the docs, we had ran static analyzers and linters through the contracts to detect basic vulnerabilities and other non critical errors. The result of our static analysis was then compared to the provided bot reports and those deemed known were removed.

- After the documentation review and static analysis, we started the process of manual code inspection. We noted how the contracts were divided into different  and we inspected the contracts in each carefully. We stuidied each of the contracts, tested how each function behaves and compared that to how they're expected to behave. We then tried out various attack ideas, including the ones provided by the sponsors. We noted the dependencies, inheritancies, and possible vulnerabilities that can arise from them. We made comparisons to issues found in protocols of the same kind (including older commits) to see if the same mistakes were repeated and how effective the provided fixes were.

- After this was done, we made notes on the issues we found and prepared the analysis report.

# Codebase quality and Code Review

Overall, I consider the quality of the Opus codebase to be excellent. The code appears to be very mature and well-developed. We have noticed the implementation of various standards adhere to appropriately. Details are explained below:


| Code Review Feedback                                                | 
|---------------------------------------------------------------------|
| **1. Conversion Functions:**                                        |                                                            |
| The `convert_to_shares` and `convert_to_yin` functions could benefit from optimization. They perform division operations which can be computationally expensive. Consider using bitwise shifting for division by powers of 2 or investigating faster division algorithms. Caching the results of expensive computations may also improve performance. |
| **2. Recursive Functions:**                                         |                                                            |
| The `get_recent_asset_absorption_error` function uses recursion, which may lead to stack overflow for large inputs. Consider refactoring to use an iterative approach or tail recursion optimization to avoid excessive stack usage. |
| **3. Storage Access:**                                              |                                                            |
| Multiple functions access storage repeatedly, such as `self.total_shares.read()` and `self.absorptions_count.read()`. Minimize storage reads by caching values locally when possible, especially within loops or recursive functions. |
| **4. Loop Optimization:**                                           |                                                            |
| Loops in functions like `update_absorbed_asset` iterate over all assets and absorptions, resulting in potentially high time complexity. Investigate ways to optimize these loops, such as early termination if no further processing is needed for a certain condition. Consider using parallelization if applicable. |
| **5. Event Emission:**                                              |                                                            |
| Emitting events (`self.emit(...)`) within loops or nested conditions can impact performance, especially if the event emission logic is complex or involves storage access. Consider batching or aggregating events to reduce overhead. |
| **6. Data Structure Usage:**                                        |                                                            |
| Review the choice of data structures such as arrays and maps. Ensure they are efficient for the intended operations (e.g., lookups, inserts, deletes). Consider using specialized data structures like sets or hash maps for improved performance. |
| **7. External Function Calls:**                                     |                                                            |
| External function calls, such as `self.yin_erc20().balance_of(absorber)`, may introduce latency due to network or contract interactions. Minimize external calls and optimize them where possible, such as batching multiple calls into a single transaction. |
| **8. Code Duplication:**                                            |                                                            |
| Identify and eliminate code duplication to improve maintainability and reduce the risk of performance issues caused by inconsistent implementations. Refactor repetitive code into reusable functions or . |
| **9. Resource Cleanup:**                                           |                                                            |
| Ensure proper resource cleanup to prevent memory leaks or other resource exhaustion issues. Review functions that allocate resources like storage or memory and verify they release resources appropriately, especially in error or exceptional cases. |
| **10. Algorithmic Complexity:**                                     |                                                            |
| Analyze the overall algorithmic complexity of the codebase. Identify areas with high time or space complexity and consider alternative algorithms or optimizations to reduce complexity and improve performance. |


| Issue Description                                                                                   | Suggestion                                                                                   |
|----------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------|
| Potential inefficiency in nested loops. Inner loop seems to iterate over all yangs multiple times. | Consider optimizing nested loops to reduce redundant iterations.                            |
| Possible memory leak due to creation of new `updated_trove_yang_balances` without proper cleanup. | Ensure proper memory management by deallocating resources or using a garbage collector.    |
| Rounding of debt may lead to precision loss.                                                      | Implement a more precise rounding mechanism to minimize precision loss.                      |
| Potential optimization in `pull_redistributed_debt_and_yangs` function by avoiding nested loops. | Refactor the function to use a more efficient algorithm to reduce computational complexity. |
| Lack of error handling for invalid input values.                                                   | Implement robust input validation to prevent unexpected behavior or crashes.                 |
| Redundant variable assignments and computations.                                                   | Eliminate unnecessary calculations and streamline variable assignments for better efficiency. |
| Lack of comments explaining the purpose of certain calculations or decisions.                      | Add descriptive comments to clarify the rationale behind specific code segments.             |
`get_max_close_amount_internal` function could be simplified. | Instead of calling `min` twice, calculate the maximum amount of debt that can be paid off in a given liquidation and return it.
`preview_liquidate_internal` function could be simplified. | Instead of returning `Option::Some((penalty, max_close_amt))`, return `Option::Some(get_max_close_amount_internal(threshold, trove_value, trove_debt, penalty))` directly.




### Strengths :

Exemplary unit test coverage: The codebase shines with its meticulous unit testing strategy. The comprehensive suite of unit tests ensures the reliability and resilience of the system's core functionality, providing confidence in its performance under various scenarios.


Artful integration of Natspec: A commendable strength is the thoughtful integration of Natspec. The use of this documentation format not only improves code readability, but also exemplifies a commitment to providing clear and human-friendly documentation that fosters a more collaborative and accessible development environment.

  

### Weaknesses :

Opportunities to improve documentation: While the codebase stands out in many ways, there is an opportunity for refinement in the documentation. Addressing this area of improvement can increase the overall clarity of the codebase, make it more accessible, and facilitate seamless collaboration among developers. In essence, the codebase demonstrates proficiency in testing methodologies and documentation practices. Strengthening the documentation further would strengthen the codebase's comprehensibility and contribute to an even more polished and developer-friendly ecosystem.


# Approach we followed when reviewing the code

Troves: This is the core of the protocol, allowing users to open, close, and manage troves, which are leveraged positions in the system. Troves enable users to borrow yin with collateral tokens as security.

Gate: Acts as an adapter and custodian for collateral tokens. This receives collateral tokens from users and returns the equivalent yang. Its goal is to ensure accurate conversion between collateral tokens and yang.

Sentinel : Acts as the internal interface for other  to interact with Gates. Facilitates actions involving collateral and enforces access control between other  and Gates.

Purger : Acts as the liquidator for unhealthy troves, protecting the protocol's health. This allows users to liquidate unhealthy troves with their own yin or using yin from the Absorber.

Absorber : Implements a stability pool that allows yin holders to participate in liquidations as a consolidated pool. Provides rewards and income distribution to yin providers.

Controller : Adjusts the global interest rate multiplier for troves based on the deviation of the spot market price from the peg price. Aims to minimize peg error by influencing trove owners' behavior.

Equalizer : Balances the Shrine's budget by allowing the budget to be reset to zero through the production of debt surpluses or the payment of debt deficits.

Seer : Acts as the coordinator of individual oracle , reading the price of underlying collateral tokens from oracle adapter  and submitting them to the Shrine.

Flash Mint : Implements EIP-3156, allowing users to borrow and repay yin in the same transaction without fees.

Caretaker : Responsible for gracefully deprecating the entire protocol by giving yin holders the opportunity to claim collateral backing their yin.

Abbot:

- Acts as the primary user interface to open and manage troves.
- Facilitates deposit, withdrawal, forging, and repayment of trove debt.

Gate :

- Acts as an adapter and custodian for collateral tokens.
- Manages entering and exiting collateral tokens as well as their conversion to and from "yang".

Sentinel:

- Acts as the internal router for interaction with Gates.
- Simplifies access control between other  and Gates.

Purger:

- Responsible for liquidating unhealthy troves.
- Facilitates liquidation using its own assets or using assets from the Absorber.

Equalizer:

- Balances the protocol's budget by minting debt surpluses or paying down debt deficits.
- Facilitates printing and repayment of debt to maintain balance.

Seer :

- Acts as the arbiter of collateral prices by coordinating individual oracle .
- Fetches prices from various sources and updates them in the Shrine.

Flash Mint :

- Implementation of EIP-3156 allowing flash loans.
- Enables borrowing and repaying yin in a single transaction without additional fees.

Caretaker:

- Responsible for gracefully shutting down the entire protocol by facilitating asset claims and collateral withdrawals.

Equalizer:

- Ensures the protocol's budget is balanced by printing excess debt or paying down deficits.

Seer :

- Fetches collateral prices from oracles and calculates the corresponding "yang" price.

Flash Mint :

- Enables users to borrow and repay yin in a single transaction without additional fees.


  

# Analysis of the code base and diagram architecture

  

## Shrine
![shrine](https://github.com/yongsxyz/Audit-Analysis/assets/39027215/0dfc9905-f720-4237-bac8-e63711d910cb)

## Sentinel
![sentinel](https://github.com/yongsxyz/Audit-Analysis/assets/39027215/8ca11e96-e792-4a5f-99b9-4b62ff891284)

## Seer
![seer](https://github.com/yongsxyz/Audit-Analysis/assets/39027215/dae739bc-1342-4cc2-b084-941c858ec7fb)

## Purger
![purger](https://github.com/yongsxyz/Audit-Analysis/assets/39027215/a315756f-c1de-45bd-a8b8-1b589c0fe4de)

## Pragma
![pragma](https://github.com/yongsxyz/Audit-Analysis/assets/39027215/fd64a04f-5393-4f82-b522-268ab8f30e56)

## Gate
![gate](https://github.com/yongsxyz/Audit-Analysis/assets/39027215/2931fe30-a375-4857-b992-27d206678a75)

## Flash_mint
![flash_mint](https://github.com/yongsxyz/Audit-Analysis/assets/39027215/a6c76162-280c-4066-8f30-c890aa540b94)

## Equalizer
![equalizer](https://github.com/yongsxyz/Audit-Analysis/assets/39027215/5a74feb4-4590-4133-a902-bed7276a5403)

## Controller
![controller](https://github.com/yongsxyz/Audit-Analysis/assets/39027215/dec52cfb-400d-4e7b-9ac9-6924a73aaad3)

## Caretaker
![caretaker](https://github.com/yongsxyz/Audit-Analysis/assets/39027215/4056d369-ca33-4a45-8ba4-22d9b073257f)

## Allocator
![allocator](https://github.com/yongsxyz/Audit-Analysis/assets/39027215/4d8db0ee-c68d-4dfa-bba7-83b2c7450db0)

## Absorber
![absorber](https://github.com/yongsxyz/Audit-Analysis/assets/39027215/e27286af-4451-49cb-a7bd-db3153f1cdfa)

## Abbot
![abbot](https://github.com/yongsxyz/Audit-Analysis/assets/39027215/f2484da3-6f27-40fb-a1b2-d39bfccb32a3)


# Risk Assessment:

Debt Surplus/Loss: The protocol's ability to maintain a balance between debt surpluses and deficits is critical for its long-term sustainability. Imbalances could lead to inflation or deflation of the yin supply, impacting user trust and confidence.

Data Feeds: The accuracy and reliability of oracle data feeds are crucial for the proper functioning of the protocol. Manipulation or inaccuracies in oracle data could lead to incorrect pricing or liquidation events on accurate pricing of collateral assets and efficient liquidation mechanisms. Inaccurate pricing or market manipulation could lead to losses and Inadequate liquidity in the system could impact users' ability to enter and exit positions, potentially leading to liquidation events or increased slippage.


# Conclusion

In general, the Opus project exhibits an interesting and well-developed architecture we believe the team has done a good job regarding the code, but the identified risks need to be addressed, and measures should be implemented to protect the protocol from potential malicious use cases. Additionally, it is recommended to improve the documentation and comments in the code to enhance understanding and collaboration among developers. It is also highly recommended that the team continues to invest in security measures such as mitigation reviews, audits, and bug bounty programs to maintain the security and reliability of the project.


### Time spent:
52 hours