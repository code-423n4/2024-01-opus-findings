# QA REPORT
| ID    |   Title    |
|:---:  |:---         |
| 1.  | The Protocol Cannot use Tokens with More than 18 Decimals.   |
| 2   | The Protocol Doesn’t implement Setters for Contract Addresses, this Could Cause that a Minor Change in a Contract would Require a new Deployment for Almost the Whole Protocol Contracts.  |
| 3.  | Some Minor Checks Could be Optimized to Avoid Duplicating Code.   |
| 4.  | Check The TODO’s in the Code.   |
| 5.  | Validate Correctness in the `update_rates` function Require a Full Array Comparition.    |
| 6.  |  Inline Directive Duplicate Code which Could Have an Undesired Effect.  |
| 7.  |  The OracleAddressUpdated Event is not Added to the Enum Event in Pragma Contract.  |
| 8.  |  Improving Contracts Documentation.  |
| 9.  |  Consider Adding a Pausing Mechanism to the contracts.  |

## 1.- The Protocol Cannot use Tokens with More than 18 Decimals.

The gates of the system currently can only work with tokens that have at most 18 decimals because the gate contract is using a function `fixed_point_to_wad` to convert the collateral asset deposit to the equivalent value of the yang, this function reverts if the token that is being added to the system has more that 18 decimal.

```cairo
fn fixed_point_to_wad(n: u128, decimals: u8) -> Wad {
    assert(decimals <= WAD_DECIMALS, 'More than 18 decimals');
    let scale: u128 = pow(10_u128, WAD_DECIMALS - decimals);
    (n * scale).into()
}
```

Also, the sentinel contract uses this function when a yang is added to the system, so the current implementation of these contracts doesn’t allow tokens that have more than 18 decimals, this is something to consider if in the future the protocol needs to use tokens with more than 18 decimal, the protocol would need to change the `fixed_point_to_wad` function to not revert with tokens that have more than 18 decimals. 


## 2.- The Protocol Doesn’t implement Setters for Contract Addresses, this Could Cause that a Minor Change in a Contract would Require a new Deployment for Almost the Whole Protocol Contracts.

Currently, the contracts in the protocol that have a connection to other contracts in the system are linked during the deployment of the contracts, but the contracts don’t implement a way to update the address that references another contract, so in case one of these contract addresses change, the protocol would have to redeploy and migrate the data (balances) and the Assets (value) that other contracts have to update the protocol contract to the new address, this migration process is error-prone and can cause mayor problems for the protocol.

It may be a good idea to implement setters in the contracts to change the addresses of other contracts if needed and avoid a redeploy and migration process when a minor change or a bug is discovered in the protocol.

To talk about a specific contract, let’s see the next example with the **Absorber** contract, this contract need to know the implementation address of the Sentinel and the Shrine contracts, as you can see in the storage of the Aboserber contract.

```cairo
#[storage]
    struct Storage {
	***
        // Sentinel associated with the Shrine for this Absorber
        sentinel: ISentinelDispatcher,
        // Shrine associated with this Absorber
        shrine: IShrineDispatcher,
	***
}
```
The addresses of the sentinel and the shrine are added during the deployment of the absorber contract, so the absorber can communicate to these contracts.

```cairo
fn constructor(
        ref self: ContractState,
        admin: ContractAddress,
        shrine: ContractAddress,
        sentinel: ContractAddress,
    ) {
        self.access_control.initializer(admin, Option::Some(absorber_roles::default_admin_role()));
        self.shrine.write(IShrineDispatcher { contract_address: shrine });
        self.sentinel.write(ISentinelDispatcher { contract_address: sentinel });
        self.is_live.write(true);
        self.current_epoch.write(FIRST_EPOCH);
    }

```

But the absorber doesn’t have any setter to update these variables whenever needed, if a bug or a minor/major change is made to the sentinel for example, the protocol would need to redeploy all the contracts that have a reference to the sentinel contract, this can require a major task cause the data in all the contracts that connect to the sentinel/shrine and de assets on all these contracts would need to be migrated which is prone to errors.

If the contracts have setters to update these variables in every contract, the admin/governance would be able to call these functions in the contracts needed to update the address of the new sentinel/shrine contracts, so a major migration wouldn’t be necessary for minor changes or bugs.


## 3.- Some Minor Checks Could be Optimized to Avoid Duplicating Code.

The contracts are pretty good designed and optimized, they extract away some chunks of logic (if checks) to avoid duplicating code, following this good practice some checks in the contracts can be extracted into a function and just call this function when they need to use these checks.

 For example in the Purger contract, the “If” that checks the LTV <= threshold can be separated into another function and just call this function when needed.

```cairo
Fn check_ltv_to_threshold(threshold: Ray, ltv: Ray) -> Option<Ray> {
if ltv <= threshold {
                return Option::None;
            }
}
```
And then just call this function in the `get_liquidation_penalty_internal` and in the `get_absorption_penalty_internal`.


## 4.- Check The TODO’s in the Code.

The Code has some TODO’s to be resolved. 
the Seer and the Shrine contracts have some ToDo’s in the code that would require attention when the time comes.

```
// TODO: when possible in Cairo, fetch_price should be wrapped
//       in a try-catch block so that an exception does not
//       prevent all other price updates
```

Just adding this here so the devs can remember this when needed.


## 5.- Validate Correctness in the `update_rates` function Require a Full Array Comparition. 

In the `update_rates` function from the Shrine, the comments at the final loop say that even when the function is an admin function is necessary to validate that the rates were updated correctly to avoid a major problem in the protocol, but this final loop only checks that the `yang_rates` variable is non zero.

 In order to validate the correctness, the `yang_rates` values and the `new_rates`variable should be compared in the loop, so you can be sure that every rate was updated correctly.

```
// Verify that all rates were updated correctly
            // This is necessary because we don't enforce that the `yangs` array really contains
            // every single yang, only that its length is the same as the number of yangs.
            // For all we know, `yangs` could contain one yang address 10 times.
            // Even though this is an admin/governance function, such a mistake could break
            // interest rate calculations, which is why it's important that we verify that all yangs'
            // rates were correctly updated.
            let mut idx: u32 = num_yangs;
            loop {
                if idx == 0 {
                    break ();
                }
                //@audit-info si buscan que todo sea correcto deberian checar que el valor sea igual al enviado en el array de rates
                // and the addresses of yangs no tengan yangs repetidos.
                assert(
                    self.yang_rates.read((idx, rate_era)).is_non_zero(), 'SH: Incorrect rate update'
                );
                idx -= 1;
            };
```



## 6.- Inline Directive Duplicate Code which Could Have an Undesired Effect.

Keep in mind that the inline(always) directive is a compiler optimization that eliminates the overhead of a function call by adding the function’s code directly into the calling function, this
 duplicates code in the Sierra contract.

This optimization is especially useful for frequently called small functions. Inlining can reduce the overhead of function calls and lead to faster and more optimized executions, as values don't need to be pushed to memory.

It’s recommended that the function that uses this directive are small functions that are called frequently because this directive can increase the overall program size by duplicating the code of a big function on every function calling these inline functions.


## 7.- The OracleAddressUpdated Event is not Added to the Enum Event in Pragma Contract.

In the Pragma contract the event `OracleAddressUpdated`is declared but is not added to the enum event and is not used in the contract, so if this event is going to be used, then it should be added to the enum so the compiler can create the event trait, if not, delete the event from the contract to avoid unused code.

```cairo
#[event]
    #[derive(Copy, Drop, starknet::Event, PartialEq)]
    enum Event {
        AccessControlEvent: access_control_component::Event,
        InvalidPriceUpdate: InvalidPriceUpdate,
        PriceValidityThresholdsUpdated: PriceValidityThresholdsUpdated,
        YangPairIdSet: YangPairIdSet,
    }
 *** // Code omitted
#[derive(Copy, Drop, starknet::Event, PartialEq)]
    struct OracleAddressUpdated {
        old_address: ContractAddress,
        new_address: ContractAddress
    }
```
## 8.- Improving Contracts Documentation.

The contracts are pretty well documented in general, but in order to improve the documentation the NatSpect syntaxis can be added to the contract.

For example, the function can document the goal of the function, what are the parameters that are received, and what are the return variables.

## 9.- Consider Adding a Pausing Mechanism to the contracts.

Currently, the system implements a kill mechanism to stop the contracts forever if this is necessary, this is a good stop mechanism, but there may be situations where a full kill of the contract is not needed, and only a pause for a moment is necessary for this reason is recommended to add a pausing mechanism, so the governance or the admin can pause the protocol contracts when is necessary.

Imagine that a minor/major bug is discovered in one contract, or the oracle stops for any reason or starts reporting bad prices, this can be a situation where a full kill of the contracts is not necessary, but only pausing of the system for a determined time, currently, this is not possible for the protocol cause only a kill mechanism is implemented, so a minor problem can lead to a total kill of the protocol, when a simple pause could be used to solve the problem and protect the protocol user’s money.