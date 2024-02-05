#  **Advanced Analysis Report for OPUS** 

[Audit approach](#1-audit-approach)

[Brief Overview](#2-brief-overview)

[Scope and Architecture Overview](#3-scope-and-architecture-overview)

[Codebase Overview](#4-codebase-overview)

[Centralization Risks](#5-centralization-risks)

[Systemic Risks](#6-systemic-risks)

[Recommendations](#7-recommendations)

[Conclusions](#8-conclusions)

[Resources](#9-resources)


## **1. Audit approach**

- **Ecosystem analysis**: A study of the starknet's ecosystem using its documentation as well as learning the cairo's language syntax to provide a base for understanding of the protocol's implementation.
 
- **Documentation Dive**: A comprehensive analysis of provided docs was conducted to understand protocol functionality, key points were noted, ambiguities were discussed with the dev, and possible risk areas were mapped.

- **Code Inspection**: Manual review of each contract within defined sections was conducted, testing function behavior against expectations, and working out potential attack vectors. Vulnerabilities related to dependencies and  inheritances, were assessed. Comparisons with similar protocols (including older commits) was also performed to identify recurring issues and evaluate fix effectiveness.

- **Report Compilation**: Identified issues were generated into a comprehensive audit report.
***

## **2. Brief Overview**

- Opus is an autonomous credit protocol that operates without significant human intervention.
- Users can borrow against a diversified range of assets, including yield-generating ones.
- Interest rates, loan-to-value ratios, and liquidation thresholds are dynamically adjusted based on each user's collateral profile.
***
## **3. Scope and Architecture Overview**

<p align="center">
    <img width= auto src="https://gist.github.com/assets/112232336/aa3c70af-c329-4a2b-b547-7152d0125953" alt="contract architecture overview.sol">
</p>

For the audit scope, the contracts are sectioned into three different sections.

### **[3.1. Core modules](https://github.com/code-423n4/2024-01-opus/tree/main/src/core)**

These are the modules that perform the most important functionalities of the protocol. They're mostly isolated from direct external interactions from users for security reasons. 

#### **[Shrine.cairo](https://github.com/code-423n4/2024-01-opus/blob/main/src/core/shrine.cairo)**

- The shrine functions as the accounting engine of the protocol. It is isolated from any direct calls from users, and is not intended to hold any tokens. It holds the balance of the yang against the yin to calculate the total debt accrued by users, calculates the interest charged on the debt, stores/updates the price of the yang and the multiplier value.

**Yang deposit and withdrawal** - The yang for a debt position can be increased or decreased upon calling the `deposit` and `withdraw` functions. These calls are made by the users via the Abbot, unless the shrine has been killed in which yang can no longer be deposited, but withdrawn via the Caretaker contract instead.

**Yang seizure** - During liquidations and trove shutdowns, yangs can be seized by calling the `seize` fucntion to withdraw a specific amount of the yang and transfer it to the liquidator.

**Debt redistribution** - Upon liquidation of an unhealthy trove, the `redistribute` function is called to redistribute the trove's yand and debt proportionally to its collateral consumption.

- The shrine contract also holds the Yin's erc20 contract that functions as the protocol's synthetic stablecoin. It implements basic ERC20 functions including approvals and transfers.

**Yin forging and melting** - Through the `forge` and `melt` functions, yin is minted and burned for the borrower. Minting the yin increases the user's debt position, burning decreases the debt position.

**Yin injection and ejection** - Yin can be forged or melted to a user without actually increasing the user's debt position.  

```
                                        +---------------------------------+
                                        |        Shrine - CLoC1313        |
                                        +---------------------------------+
                                        | - admin                         |
                                        | - name                          |
                                        | - symbol                        |
                                        | - access_control                |
                                        | - troves                        |
                                        | - yin                           |
                                        | - yang_total                    |
                                        | - initial_yang_amts             |
                                        | - yangs_count                   |
                                        | - yang_ids                      |
                                        | - deposits                      |
                                        | - total_troves_debt             |
                                        | - total_yin                     |
                                        | - budget                        |
                                        | - yang_prices                   |
                                        | - yin_spot_price                |
                                        | - minimum_trove_value           |
                                        | - debt_ceiling                  |
                                        | - multiplier                    |
                                        | - rates_latest_era              |
                                        | - rates_intervals               |
                                        | - yang_rates                    |
                                        | - yang_suspension               |
                                        | - thresholds                    |
                                        | - redistributions_count         |
                                        | - trove_redistribution_id       |
                                        | - is_exceptional_redistribution |
                                        | - yang_redistributions          |
                                        | - yang_to_yang_redistribution   |
                                        | - is_live                       |
                                        | - yin_name                      |
                                        | - yin_symbol                    |
                                        | - yin_decimals                  |
                                        | - yin_allowances                |
                                        +---------------------------------+
                                        | + constructor()                 |
                                        | + add_yang()                    |
                                        | + set_threshold()               |
                                        | + suspend_yang()                |
                                        | + unsuspend_yang()              |
                                        | + update_rates()                |
                                        | + advance()                     |
                                        | + set_multiplier()              |
                                        | + set_minimum_trove_value()     |
                                        | + set_debt_ceiling()            |
                                        | + adjust_budget()               |
                                        | + update_yin_spot_price()       |
                                        | + kill()                        |
                                        | + deposit()                     |
                                        | + withdraw()                    |
                                        | + forge()                       |
                                        | + melt()                        |
                                        | + seize()                       |
                                        | + redistribute()                |
                                        | + inject()                      |
                                        | + eject()                       |
                                        | + get_yin()                     |
                                        | + get_total_yin()               |
                                        | + get_yin_spot_price()          |
                                        | + get_yang_total()              |
                                        | + get_initial_yang_amt()        |
                                        | + get_yangs_count()             |
                                        | + get_deposit()                 |
                                        | + get_budget()                  |
                                        | + get_yang_price()              |
                                        | + get_yang_rate()               |
                                        | + get_current_rate_era()        |
                                        | + get_minimum_trove_value()     |
                                        | + get_debt_ceiling()            |
                                        | + get_multiplier()              |
                                        | + get_yang_suspension_status()  |
                                        | + get_yang_threshold()          |
                                        | + get_shrine_health()           |
                                        | + get_redistributions_count()   |
                                        | + get_trove_redistribution_id() |
                                        | + get_redistribution_for_yang() |
                                        | + is_recovery_mode()            |
                                        | + get_live(): bool              |
                                        +---------------------------------+
                                        |           ERC20 Yin             |
                                        +---------------------------------+
                                        | + name()                        |
                                        | + symbol()                      |
                                        | + decimals()                    |
                                        | + total_supply()                |
                                        | + balance_of                    |
                                        | + allowance                     |
                                        | + transfer                      |
                                        | + transfer_from                 |
                                        | + approve                       |
                                        | + supports_interface()          |
                                        +---------------------------------+

```
#### **[Gate.cairo](https://github.com/code-423n4/2024-01-opus/blob/main/src/core/gate.cairo)**

- The Gate acts as the secure escrow for each type of collateral token users deposit. It takes custody of the tokens and keeps them safe. Users don't interact with the Gate directly, instead they deposit tokens into a trove which sends them to the Gate for safekeeping. 

**Entering and exiting a gate** - Through the `enter` and `exit` functions, made from the sentinel, transfers a stipulated amount of assets from the user returning the corresponding amount of yang and vice versa. 

```
                                          +-----------------------------+
                                          |         Gate - CLoC120      |
                                          +-----------------------------+
                                          | - shrine: IShrineDispatcher |
                                          | - asset: IERC20Dispatcher   | 
                                          | - sentinel: ContractAddress |
                                          +-----------------------------+
                                                         ^
                                                         |
                                          +-----------------------------+
                                          | Storage                     |
                                          +-----------------------------+
                                          | - shrine: IShrineDispatcher |
                                          | - asset: IERC20Dispatcher   |
                                          | - sentinel: ContractAddress |
                                          +-----------------------------+
                                                         ^
                                                         |
                                          +-----------------------------+
                                          | Event                       |
                                          +-----------------------------+
                                          | - user: ContractAddress     |
                                          | - trove_id: u64             |
                                          | - asset_amt: u128           |
                                          | - yang_amt: Wad             |
                                          +-----------------------------+
                                          | + Enter                     |
                                          | + Exit                      |
                                          +-----------------------------+
                                                         ^
                                                         |
                                          +-----------------------------+
                                          | IGateImpl                   |
                                          +-----------------------------+
                                          | + get_shrine                |
                                          | + get_sentinel              |
                                          | + get_asset                 |
                                          | + get_total_assets          |
                                          | + get_total_yang            |
                                          | + get_asset_amt_per_yang    |
                                          | + convert_to_yang           |
                                          | + convert_to_assets         |
                                          | + enter                     |
                                          | + exit                      |
                                          +-----------------------------+
                                                         ^
                                                         |
                                          +-----------------------------+
                                          | GateHelpers                 |
                                          +-----------------------------+
                                          | + assert_sentinel           |
                                          | + get_total_yang_helper     |
                                          | + convert_to_assets_helper  |
                                          | + convert_to_yang_helper    | 
                                          +-----------------------------+
                                                         ^
                                                         |
                                          +-----------------------------+
                                          | gate                        |
                                          +-----------------------------+
                                          | # get_total_assets_helper   |
                                          +-----------------------------+

```  

#### **[Sentinel.cairo](https://github.com/code-423n4/2024-01-opus/blob/main/src/core/sentinel.cairo)**

- The Sentinel module serves as an intermediary between other modules and Gates in the system. It routes actions related to a collateral token to its corresponding Gate and enforces access control between modules and Gates. Additionally, it acts as a gatekeeper by ensuring that Gates are deployed correctly and that collateral token deposits follow specific rules.

**Yang addition, suspension and unsuspension** - Through the `add_yang` function, admins can add new yangs to function as collateral in the protocol. As, yangs cannot be removed, there's the `suspend_yang` and `unsuspend_yang` pause and unpause protocols interactions with the yang. Important to note that there's a time period for which a yang can be suspended before which unsuspension becomes impossible. 

**Killing a gate** - The `kill_gate` function freezes a Gate, preventing new deposits but allowing withdrawals. This ensures users can exit when needed while stopping further investments.

``` 
                                           +--------------------------+    
                                           |     Sentinel - CLoC173   |
                                           +--------------------------+     
                                           | -access_control          |    
                                           | -yang_to_gate            |     
                                           | -yang_addresses          |     
                                           | -yang_asset_max          |
                                           | -yang_is_live            |
                                           | -shrine                  |
                                           +--------------------------+
                                                        ^
                                                        |
                                           +--------------------------+    
                                           | ISentinelImpl            |
                                           +--------------------------+     
                                           | +get_gate_address        |     
                                           | +get_gate_live           |     
                                           | +get_yang_addresses      |    
                                           | +get_yang                |     
                                           | +get_yang_asset_max      |   
                                           | +get_asset_amt_per_yang  |
                                           | +convert_to_yang         |     
                                           | +convert_to_assets       |     
                                           | +add_yang                |
                                           | +set_yang_asset_max      |
                                           | +enter                   |
                                           | +exit                    |
                                           | +kill_gate               |
                                           | +suspend_yang            |
                                           | +unsuspend_yang          |
                                           +--------------------------+
                                                         ^
                                                         |
                                           +--------------------------+
                                           | SentinelHelpers          |
                                           +--------------------------+
                                           | #assert_can_enter        |
                                           +--------------------------+

```
### **[3.2. Management Modules](https://github.com/code-423n4/2024-01-opus/tree/main/src/core)**

These contracts perform various management functions of the protocol. Balancing budgets, updating interest multipliers, fetching and updating yang prices from oracles and so on. These modules of the most part free of human interactions to perform their functionalities.

#### **[Caretaker.cairo](https://github.com/code-423n4/2024-01-opus/blob/main/src/core/caretaker.cairo)**
- The Caretaker module acts as a "safety switch" for the protocol, allowing a controlled shutdown process in case of emergency. It focuses on gracefully deprecating the Shrine, which holds the collateral backing yin tokens, and enabling yin holders to reclaim their collateral. This ensures users are not left with locked-up funds if the protocol needs to be shut down.

**Shrine shutdown** - By calling the `shut` function, admins disable all user functions on the shrine and kill it. The function also transfers a portion of the collateral to the contract.

**Collateral release and reclaim** - After the shutdown, users can exchange their yin for a percentage of the collateral assets in the Caretaker. The amount of assets a yin holder is entitled to is proportional to the remaining amount of yin that is reclaimable. They do this by calling the `reclaim` function. The `release` function functions like this, but allows trove owners instead to withdraw collateral from their trove. 

```
                                              +----------------------------+
                                              |    Caretaker - CLoC193     |
                                              |----------------------------|
                                              | - access_control           |
                                              | - reentrancy_guard         |
                                              | - abbot                    |
                                              | - equalizer                |
                                              | - sentinel                 |
                                              | - shrine                   |
                                              | - reclaimable_yin          |
                                              |----------------------------|
                                              | + preview_release()        |
                                              | + preview_reclaim()        |
                                              | + shut()                   |
                                              | + release()                |       
                                              | + reclaim()                |       
                                              +----------------------------+       

```
#### **[Controller.cairo](https://github.com/code-423n4/2024-01-opus/blob/main/src/core/controller.cairo)**

- The controller module makes automatic adjustments to the troves' interest rate to maintain a stable market price for the yin. It raises/reduces the market price of the yin in case it goes below/above peg by adjusting the interest rate multiplier.

**Multiplier update** - The `update_multiplier` function updates the interest rate multiplier based on the standard "Proportional Integral" in which updates are made aggressively or not to changes in current multiplier based perceived signs of mismatch in the yin supply and demand. 

```
                                            +-----------------------------+
                                            |     Controller - CLoC188    |
                                            +-----------------------------+
                                            | - access_control            |
                                            | - shrine                    |
                                            | - yin_previous_price        |
                                            | - yin_price_last_updated    |
                                            | - i_term_last_updated       |
                                            | - i_term                    |
                                            | - p_gain                    |
                                            | - i_gain                    |
                                            | - alpha_p                   |
                                            | - beta_p                    |
                                            | - alpha_i                   |
                                            | - beta_i                    |
                                            +-----------------------------+
                                            | + constructor()             |
                                            | + get_current_multiplier()  |
                                            | + get_p_term()              |
                                            | + get_i_term()              |
                                            | + get_parameters()          |
                                            | + update_multiplier()       |
                                            | + set_p_gain()              |
                                            | + set_i_gain()              |
                                            | + set_alpha_p()             |
                                            | + set_beta_p()              |
                                            | + set_alpha_i()             |
                                            | + set_beta_i()              |
                                            +-----------------------------+

```
#### **[Equalizer.cairo](https://github.com/code-423n4/2024-01-opus/blob/main/src/core/equalizer.cairo)**

- This functions as the budget balancer by minting budget surpluses or paying down budget deficits. It does this to keep the budget balanced and allows reseting to zero. 

**Equalizing the budget** - The equalize function helps maintain a balance between debt and its corresponding surplus in the system. If the Shrine has a positive budget, the function mints new debt surpluses and adds them to the Equalizer. This ensures that the amount of outstanding debt (yin) matches the total surplus in the system, preventing an unsustainable situation where debt keeps growing without ever being repaid.

**Normalizing the budget** - The `normalize` function is called to reduce the budget deficit in the shrine by burning yin from the caller. 

**Allocating yin balance** - To allocate the yin balance, the `allocate` function is called, in which the yin balance from the Equalizer is distributed to different recipients based on their designated percentages. The allocation information is gotten from the Allocator.

```
                                    +-----------------------------------+
                                    |       Equalizer - CLoC120         |
                                    +-----------------------------------+
                                    | - access_control                  |
                                    | - allocator                       |
                                    | - shrine                          |
                                    +-----------------------------------+
                                    | + constructor()                   |
                                    | + get_allocator()                 |
                                    | + set_allocator()                 |
                                    | + equalize()                      |
                                    | + allocate()                      |
                                    | + normalize()                     |
                                    +-----------------------------------+

```
#### **[Allocator.cairo](https://github.com/code-423n4/2024-01-opus/blob/main/src/core/allocator.cairo)**

- This contract holds and update information about allocation recipients and their repective percentage shares of the newly minted surplus debt.

```
                                      +------------------------------+
                                      |      Allocator - CLoC78      |
                                      +------------------------------+
                                      | - access_control             |
                                      | - recipients_count           |
                                      | - recipients                 |
                                      | - percentages                |
                                      +------------------------------+
                                      | + constructor()              |
                                      | + set_allocation()           |
                                      | + get_allocation()           |
                                      +------------------------------+
```
#### **[Seer.cairo](https://github.com/code-423n4/2024-01-opus/blob/main/src/core/seer.cairo)**

- The seer contract gets collateral prices from the protocol's oracles and converts them to yang prices. It then sends these prices over to the shrine to update the yang's price.

**Oracle and price frequenct setup** - The `set_oracles` and `set_update_frequency` functions are called by the admin to update the protocol's oracle address and frequency of update of prices. The current oracle in use is StarkNet's pragma oracle. The frequency of price updates (the minimal time difference for which prices from the oracle are fetched) exists within a range of about 15 seconds and 4 hours.

**Yang price update** - The admin/purger, by calling the `update_prices` function gets the earliest valid price for the yangs and updates their prices in the shrine.

```
                                           +-------------------------------------+
                                           |           Seer - CLoC154            |
                                           +-------------------------------------+
                                           | - access_control                    |
                                           | - shrine                            |
                                           | - sentinel                          |
                                           | - oracles                           |
                                           | - last_update_prices_call_timestamp |
                                           | - update_frequency                  |
                                           +-------------------------------------+
                                           | + constructor()                     |
                                           | + get_oracles()                     |
                                           | + set_oracles()                     |
                                           | + get_update_frequency()            |
                                           | + set_update_frequency()            |
                                           | + update_prices()                   |
                                           +-------------------------------------+
```
### **[3.3. User-facing Modules](https://github.com/code-423n4/2024-01-opus/tree/main/src/core)**

These contracts are the main interfaces through which the users interact with the protocol. Through these contracts, troves can be opened and closed, debt positions open, liquidations performed and so on.

#### **[Abbot.cairo](https://github.com/code-423n4/2024-01-opus/tree/main/src/core/abbot.cairo)**

- User, through the abbot interact with their troves, like opening and managing them. It also ensures each user gets a unique trove ID, starting from 1. This module is designed to be permanent and only deployed once for each synthetic asset.

**Trove creation and closure** - The `open_trove` function creates new troves, deposits yang, and optionally borrows yin. It's like combining "deposit" and "forge" functions. The `close_trove` functions repays all debt and withdraws all collateral from a trove. It's like combining "melt" and "withdraw" functions.

**Yang deposit and withdrawal** - Users can call the `deposit` and `withdraw` functions to deposit and withdraw yangs as collaterals into and from the trove. Any user can deposit yangs into the trove, however only the trove creator/owner can withdraw.

**Yin forge and melting** - For a specific yang amount, the trove owner can mint yin by calling the `forge` function, increasing the trove's debt. To pay off this debt, any user can call the `melt` function to burn their yin and decrease the corresponding debt amount in the trove.

```
                                          Constructor
                                              |
                                              V
                                          Set Shrine and Sentinel
                                              |
                                              V
                                          External Abbot Functions
                                              |
                                              V
                                              -------------------------------
                                              |                             |
                                              V                             V
                                          get_trove_owner()            get_user_trove_ids()
                                              |                             |
                                              V                             V
                                          get_troves_count()           get_trove_asset_balance()
                                              |                             |
                                              V                             V
                                          open_trove()                 close_trove()
                                              |                             |
                                              V                             V
                                          deposit()                   withdraw()
                                              |                             |
                                              V                             V
                                          forge()                         melt()
                                              |                             
                                              V                             
                                          Internal Abbot Functions
                                              |
                                              V
                                          assert_trove_owner()
                                              |
                                              V
                                          deposit_helper()
                                              |
                                              V
                                          withdraw_helper()

```
#### **[Purger.cairo](https://github.com/code-423n4/2024-01-opus/tree/main/src/core/purger.cairo)**
- The Purger acts as the main tool for keeping the protocol financially healthy by eliminating risky "troves" (debt positions). Anyone can participate in this liquidation process, helping to maintain the protocol's stability.

**Trove liquidation and absorbtion** - Users can liquidate unhealthy troves' debts using their yin and gets the corresponding collateral value and a liquidation penalty as reward. This is done by calling the `liquidate` function. Absorbtion on the other hand is a way to help an unhealthy trove by using the Absorber's yin balance is used to pay down the unhealthy trove's debt. The Absorber receives the corresponding amount of collateral back, plus a liquidation penalty which is then given to the "absorb" function caller as reward. If the debt is more than the available yin, it is spread out among other users. If there is no yin, the entire debt is redistributed, but the caller still gets their reward.

 ```
                                              +---------------------+                                 
                                              |   Purger - CLoC361  |                                 
                                              |---------------------|                                 
                                              | - admin             |                                 
                                              | - shrine            |                                 
                                              | - sentinel          |                                 
                                              | - absorber          |                                 
                                              | - seer              |                                 
                                              | - penalty_scalar    |                                 
                                              +---------------------+                                 
                                              | + set_penalty_scalar|                                               
                                              | + liquidate         |                                               
                                              | + absorb            |                                               
                                              +---------------------+                                               
                                                        ^                                                               
                                                        |                                                        
                                                        v                                                               
                                              +--------------------+                              
                                              |     Shrine         |                              
                                              |--------------------|                              
                                              | - contract_address |                                
                                              |--------------------|                                
                                              | + get_trove_health |                                
                                              | + melt             |                                
                                              | + get_yin          |                                 
                                              +--------------------+                                

```
#### **[Absorber.cairo](https://github.com/code-423n4/2024-01-opus/tree/main/src/core/absorber.cairo)**
- The Absorber acts as a secondary layer for liquidations in Opus, allowing yin holders to participate in "absorptions" (liquidations) and earn rewards.

**Yin provision, request or removal** -  Users can provide yin for liquidations by calling the `provide` function. Doing this provides the user incentives in form of internal shares which can later be redeemed for yang. In cases where the provided yin proves to be more than enough, the user can make a yin removal request by calling the `request` function. The request is valid for a fixed period of time upon which it can be fulfilled, expired or be overwritten by making a new request. After, the request has met the required minimum timeframe, the `remove` function can be called to withdraw the yin.

**Collateral reaping and update** - The user who had provided yin for liquidation and had received shares in return can call the `reap` function to facilitate this withdrawal. After this is successful, the purger updates the accounting of the absorbed assets so as to prevent excessive reaping. It does this by calling the `update` function.

**Absorber shutdown** - The `kill` function is called by the admin to irreversibly pause the `provide` function and prevents users from depositing more yins into the absorber. At this point, no more asset rewards are processed for the users. 

```
                                       +-----------------------------------------+
                                       |           Absorber - CLoC617            |
                                       +-----------------------------------------+
                                       | - access_control                        |
                                       | - sentinel                              |
                                       | - shrine                                |
                                       | - is_live                               |
                                       | - current_epoch                         |
                                       | - absorptions_count                     |
                                       | - provider_last_absorption              |
                                       | - provisions                            |
                                       | - absorption_epoch                      |
                                       | - total_shares                          |
                                       | - asset_absorption                      |
                                       | - epoch_share_conversion_rate           |
                                       | - rewards_count                         |
                                       | - reward_id                             |
                                       | - rewards                               |
                                       | - cumulative_reward_amt_by_epoch        |
                                       | - provider_last_reward_cumulative       |
                                       | - provider_request                      |
                                       +-----------------------------------------+
                                       | + constructor()                         |
                                       | + provide()                             |
                                       | + request()                             |
                                       | + remove()                              |
                                       | + reap()                                |
                                       | + update()                              |
                                       | + kill()                                |
                                       | + get_rewards_count()                   |
                                       | + get_rewards()                         |
                                       | + get_current_epoch()                   |
                                       | + get_absorptions_count()               |
                                       | + get_absorption_epoch()                |
                                       | + get_total_shares_for_current_epoch()  |
                                       | + get_provision()                       |
                                       | + get_provider_last_absorption()        |
                                       | + get_provider_request()                |
                                       | + get_asset_absorption()                |
                                       | + get_cumulative_reward_amt_by_epoch()  |
                                       | + get_provider_last_reward_cumulative() |
                                       | + get_live()                            |
                                       | + is_operational()                      |
                                       | + preview_remove()                      |
                                       | + preview_reap()                        |
                                       | + set_reward()                          |
                                       +-----------------------------------------+

```
#### **[Flash_mint.cairo](https://github.com/code-423n4/2024-01-opus/blob/main/src/core/flash_mint.cairo)**	

- The Flash Mint module allows users to temporarily borrow and repay the yin within a single transaction, similar to a flash loan. This borrowing is free of fees. However, there's a limit on how much yin that can be borrowed, set as a fixed percentage of the total yin in circulation. This limit can be easily changed by updating the module itself if needed.
```
                          +--------------------------+                     +---------------------+
                          | Constructor              |                     | Shrine Dispatcher   |
                          |                          |                     |                     |
                          | + shrine                 |-------------------->| + get_total_yin     |
                          |                          |                     | + get_debt_ceiling  |
                          |                          |                     | + inject            |
                          |                          |                     | + eject             |
                          |                          |                     | + set_debt_ceiling  |
                          +--------------------------+                     +---------------------+
                                   |                                                 |
                                   |                                                 |
                                   v                                                 v
                          +--------------------------+                     +---------------------+
                          | IFlashMintImpl           |                     | IFlashBorrower      |
                          |                          |                     |                     |
                          | + max_flash_loan         |                     | + on_flash_loan     |
                          | + flash_fee              |                     |                     |
                          | + flash_loan             |                     |                     |
                          +--------------------------+                     +---------------------+
```


### **[3.4. Utility Modules](https://github.com/code-423n4/2024-01-opus/tree/main/src/core)**

These are basic contracts that store important protocol information, from which they can be accessed by any of the contracts.

#### **[Pragma.cairo](https://github.com/code-423n4/2024-01-opus/blob/main/src/core/pragma.cairo)**

- The pragma contract functions as the adapter for starknet's Pragma oracle. It's collects current collateral prices, checks them for staleness and for number of sources, all in all to keep the prices fresh and accurate. 

**Yang pair and price validity setup** - By calling the `set_yang_pair_id` and `set_price_validity_thresholds` functions, the contract admin sets the feed address attaching it to the yang and also sets the required number of sources and freshness interval required.

```
                                            +----------------------------------+
                                            |         Pragma - CLoC129         |
                                            +----------------------------------+
                                            | - access_control                 |
                                            | - oracle                         |
                                            | - price_validity_thresholds      |
                                            | - yang_pair_ids                  |
                                            +----------------------------------+
                                            | + constructor()                  |
                                            +----------------------------------+
                                                            ^
                                                            |
                                            +----------------------------------+
                                            |    IPragmaImpl                   |
                                            +----------------------------------+
                                            | + set_yang_pair_id()             |
                                            | + set_price_validity_thresholds()|
                                            +----------------------------------+
                                                             ^
                                                             |
                                            +----------------------------------+
                                            |    IOracleImpl                   |
                                            +----------------------------------+
                                            | + get_name()                     |
                                            | + get_oracle()                   |
                                            | + fetch_price()                  |
                                            +----------------------------------+
                                                             ^
                                                             |
                                            +----------------------------------+
                                            | PragmaInternalFunctions          |
                                            +----------------------------------+
                                            | + is_valid_price_update()        |
                                            +----------------------------------+

```
#### **[Types.cairo](https://github.com/code-423n4/2024-01-opus/blob/main/src/types.cairo)**

- This contract holds the various variable types used in the contract. Information here included the health of a trove/shrine, the suspension status of a yang, redistribution status, distribution info and so on. It also includes functions to pack and unpack the various data types in use in the protocol.

```
+---------------------+     +----------------------+      +----------------------------+       +---------------------+
|   Trove             |     |   YangBalance        |      |   AssetBalance             |       |  YangRedistribution |
+---------------------+     +----------------------+      +----------------------------+       +---------------------+ 
| - charge_from: u64  |     | - yang_id: u32       |      | - address: ContractAddress |       | - unit_debt: Wad    |
| - last_rate_era: u64|     | - amount: Wad        |      | - amount: u128             |       | - error: Wad        | 
| - debt: Wad         |     +----------------------+      +----------------------------+       | - exception: bool   |
+---------------------+                                                                        +---------------------+
                                                                    
+---------------------------------+           +-----------------------------+          +-------------------------------+ 
|   ExceptionalYangRedistribution |           |   DistributionInfo          |          |   Reward                      | 
+---------------------------------+           +-----------------------------+          +-------------------------------+
| - unit_debt: Wad                |           | - asset_amt_per_share: u128 |          | - asset: ContractAddress      |
| - unit_yang: Wad                |           | - error: u128               |          | - blesser: IBlesserDispatcher | 
+---------------------------------+           +-----------------------------+          | - is_active: bool             |
                                                                                       +-------------------------------+
                                                            
+---------------------+              +----------------------+              +-------------------------------------+ 
|   Provision         |              |   Request            |              |   PragmaPricesResponse              |
+---------------------+              +----------------------+              +-------------------------------------+ 
| - epoch: u32        |              | - timestamp: u64     |              | - price: u128                       |  
| - shares: Wad       |              | - timelock: u64      |              | - decimals: u32                     |
+---------------------+              | - has_removed: bool  |              | - last_updated_timestamp: u64       |
                                     +----------------------+              | - num_sources_aggregated: u32       | 
                                                                           | - expiration_timestamp: Option<u64> |
                                                                           +-------------------------------------+ 
                               
```
#### **[Roles.cairo](https://github.com/code-423n4/2024-01-opus/blob/main/src/core/roles.cairo)**

- Here, the access control of each contract and the admin is declared. The syntax involves declaring the contract and its roles, then declaring the contracts or admins who are allowed to fulfill the role. As an example, in the absorber contract, The roles are first declared, in this case `KILL`, `SET_REWARD`, `UPDATE`. Then the contracts/admins expected to fulfill the roles are then granted access. So the purger contract is granted the `UPDATE` role, while the admin gets the `KILL` and `SET_REWARD` functions. As a result, the admin cannot directly call the `update` function in the absorber contract.
```
mod absorber_roles {   //declares the roles available in the contract, 
    const KILL: u128 = 1;
    const SET_REWARD: u128 = 2;
    const UPDATE: u128 = 4;

    #[inline(always)] 
    fn purger() -> u128 { //declares the contract/callers who can call the role
        UPDATE
    } //so this means that the purger contract can call any function that requires the admin role

    #[inline(always)]
    fn default_admin_role() -> u128 {
        KILL + SET_REWARD
    }
}
```
```
         +---------------------+       +-----------------------+      +---------------------+       +----------------------+
         |     AbsorberRoles   |       |   AllocatorRoles      |      |     BlesserRoles    |       |   CaretakerRoles     |
         |---------------------|       |-----------------------|      |---------------------|       |----------------------|
         | - KILL: u128        |       | - SET_ALLOCATION: u128|      | - BLESS: u128       |       | - SHUT: u128         |
         | - UPDATE: u128      |       +-----------------------+      +---------------------+       +----------------------+
         | - SET_REWARD: u128  |     
         +---------------------+              

  +------------------------+    +----------------------+    +--------------------------------------+    +---------------------------+
  |     ControllerRoles    |    |   EqualizerRoles     |    |     PragmaRoles                      |    |   PurgerRoles             |
  |------------------------|    |----------------------|    |--------------------------------------|    |---------------------------|
  | - TUNE_CONTROLLER: u128|    | - SET_ALLOCATOR: u128|    | - ADD_YANG: u128                     |    | - SET_PENALTY_SCALAR: u128|
  +------------------------+    +----------------------+    | - SET_ORACLE_ADDRESS: u128           |    +---------------------------+
                                                            | - SET_PRICE_VALIDITY_THRESHOLDS: u128| 
                                                            +--------------------------------------+

  +-----------------------------+          +-------------------------------+          +--------------------------------+
  |     SeerRoles               |          |   SentinelRoles               |          |   ShineRoles                   |
  |-----------------------------|          |-------------------------------|          |--------------------------------|
  | - SET_ORACLES: u128         |          | - ADD_YANG: u128              |          | - ADD_YANG: u128               |
  | - SET_UPDATE_FREQUENCY: u128|          | - ENTER: u128                 |          | - ADJUST_BUDGET: u128          |         
  | - UPDATE_PRICES: u128       |          | - EXIT: u128                  |          | - ADVANCE: u128                |
  +-----------------------------+          | - KILL_GATE: u128             |          | - DEPOSIT: u128                |
                                           | - SET_YANG_ASSET_MAX: u128    |          | - EJECT: u128                  |
                                           | - UPDATE_YANG_SUSPENSION: u128|          | - FORGE: u128                  |
                                           +-------------------------------+          | - INJECT: u128                 |
                                                                                      | - KILL: u128                   |
                     +---------------------------------+                              | - MELT: u128                   |
                     |   TransmuterRoles               |                              | - REDISTRIBUTE: u128           |
                     |---------------------------------|                              | - SEIZE: u128                  |
                     | - ENABLE_RECLAIM: u128          |                              | - SET_DEBT_CEILING: u128       |
                     | - KILL: u128                    |                              | - SET_MINIMUM_TROVE_VALUE: u128|
                     | - SETTLE: u128                  |                              | - SET_MULTIPLIER: u128         |
                     | - SET_CEILING: u128             |                              | - SET_THRESHOLD: u128          |
                     | - SET_FEES: u128                |                              | - UPDATE_RATES: u128           |
                     | - SET_PERCENTAGE_CAP: u128      |                              | - UPDATE_YANG_SUSPENSION: u128 |
                     | - SET_RECEIVER: u128            |                              | - UPDATE_YIN_SPOT_PRICE: u128  |
                     | - SWEEP: u128                   |                              | - WITHDRAW: u128               |
                     | - TOGGLE_REVERSIBILITY: u128    |                              +--------------------------------+
                     +---------------------------------+

```

***
***
## **3. Codebase Overview**

- **Audit Information** - For the purpose of the security review, Opus comprises fifteen smart contracts totaling over 4110 CLoC. Its core design principle is composition, enabling efficient and flexible integration. Pragma oracle is used to generate token prices and a standard PI (Proportional-Integral) controller is used adjust protocol interest rates.

- **Documentation and CSS** - The codebase is divided into nine major sections - core, arbitrage, dao, launch, pool, price-feed, reward, stable and staking sections. There provided documentation provides a very good overview of each modules and its functions. The contracts are well commented (not necessarily to Cairo Comment Standard), and explanations where given as to expected functionalities of each function. Top tier.

- **Naming Conventions** - The protocol uses a non-standard naming convention which made the audit process a bit challenging. Yangs, yins, era, forge, eject and so on. A [glossary](https://demo-35.gitbook.io/untitled/introduction/glossary) was however provided to explain the meaning and their possible normal equivalents.

- **Protocol Ecosystem** - The protocol's ecosystem  relies on the starknet L2 chain, for deployments. The contracts are written in cairo 1, and are to be compiled with scarb and starknet foundry. Not a very common ecosystem like the solidity based contracts.
 
- **Token Support** - The protocol mainly works with ERC20 tokens. WBTC, WETH, wstETH, and yin are the main tokens interaction in the protocol. New tokens including non-standard erc20 tokens can be added by the protocol admin to function as collaterals. 

- **Testing** - The overall test coverage is about 90% and each section implements various test ideas. Consequently, this improved the modularity of the codebase and helped eliminate most of the basic bugs. There are no invariant or fuzz tests implemented.

- **Attack Vectors** - Various points of attack exists for a protocol of this size. Token pricing manipulations, Issues from non standard ERC20 tokens like WBTC, logical errors, unfair liquidations and so on.    

***

## **4. Centralization Risks** 
The protocol aims to be free of human intervention, and to a point follows through with this, with a number of autonomous modules. However, there's still a central admin role, which performs a number of important protocol functionalities. While this admin is essentially trusted, having such a centralized power puts the protocol at risks, some of which are: 
- Setting various protocol parameters extremely high or low to grief users.
- Malicious killing of troves, gates, shrines and so on.
- Malicious suspension of yangs, affecting liquidations, debt payments and yield generation. 

***
## **5. Systemic Risks** 

- Issues with protocol engagement as the ecosystem is not very mainstream.
- External dependencies from pragma oracle, imported contracts, starknet compilers and so on.
- Smart contract bugs which at best might be contained to erring contract and at worst could affect entire protocol.
- Non standard erc20 tokens which can be a source of attack. WBTC, one of the protocol's core token is pausable, which can cause issues with liquidation and collateral claiming.
- Lack of cap on the amount of yin tokens that can be minted, potentially leading to scalability issues. and so on.

***
## **6. Recommendations**

- Two step variable and address updates should be implemented including zero address checks. A timelock can also be considered for the setter functions to give users time to react to protocol changes. Adding these fixes can help protect from mistakes and unexepected behaviour.
- A proper pause function can be implemented to protect users from being unfairly liquidated upon yang unsuspension. It also helps for turbulent market situations and black swan events. This is also important as there's no fallback oracle on starknet yet, so if pragma goes down, protocol operations will be conducted at stale prices.
- User protections be put in place for troves without owners, as anyone can still deposit in them, but only trove owners are allowed to withdraw. This can to loss of funds for the users. 
- Issues from non-standard ERC20 tokens should also be noted, blocklisting properties, pausable properties, lack of return values and so on. For instance, a yang can be added multiple times, if it's a multiple address token like TUSD. wstETH has rebasing characteristics, and that also needs to be expected during protocol interactions.
- A more conventional naming convention can also be adopted to improve user experience and ease understanding protocol
- Testing can be improved to 100% and also fuzz and invariant tests incorporated.
***
## **7. Conclusions**

-  As an endnote, the codebase was pretty well designed, albeit hard to crack, due its fairly unconventional nautre. All the same, a number of risks were identified and they need to be fixed. Recommended measures should be implemented to protect the protocol from potential attacks. Timely audits and codebase cleanup should be conducted to keep the codebase fresh and up to date with evolving security times.
***
## **8. Resources**

- [C4 ReadMe](https://code4rena.com/audits/2024-01-opus#top)
- [Protocol overview documentation](https://demo-35.gitbook.io/untitled/)
- [Starknet docs](https://docs.starknet.io/documentation/)
- [Cairo Book](https://docs.cairo-lang.org/index.html)
- [Pragma Oracle Docs](https://docs.pragmaoracle.com/)

### Time spent:
50 hours