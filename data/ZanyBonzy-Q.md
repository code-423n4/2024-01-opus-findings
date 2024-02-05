
# 1. `flash_mint.cairo` isn't fully ERC-3165 compliant 
Links to affected code *
https://github.com/code-423n4/2024-01-opus/blob/4720e9481a4fb20f4ab4140f9cc391a23ede3817/src/core/flash_mint.cairo#L100-L154
## Impact
The protocol advertises the flash_mint contract as ERC-3165 compliant and follows the standard for the most part. 

In the flashFee and flashLoan function from the spec, a check for the loan currencies are required to be conducted.

```
    /**
     * @dev The fee to be charged for a given loan.
     * @param token The loan currency. Must match the address of this contract. //@note
     * @param amount The amount of tokens lent.
     * @return The amount of `token` to be charged for the loan, on top of the returned principal.
     */
    function flashFee( 

        /**
     * @dev Loan `amount` tokens to `receiver`, and takes it back plus a `flashFee` after the ERC3156 callback.
     * @param receiver The contract receiving the tokens, needs to implement the `onFlashLoan(address user, uint256 amount, uint256 fee, bytes calldata)` interface.
     * @param token The loan currency. Must match the address of this contract. //@note
     * @param amount The amount of tokens lent.
     * @param data A data parameter to be passed on to the `receiver` for any custom use.
     */
    function flashLoan(
```

However, in the `flash_loan` function, there is no check that the loan currency is the address of the shrine, causing that loans for unsupported tokens can be taken although zero amounts. This breaks the [flashloan standard](https://eips.ethereum.org/EIPS/eip-3156#flash-mint-reference-implementation).

```
        fn flash_loan(
            ref self: ContractState,
            receiver: ContractAddress,
            token: ContractAddress,
            amount: u256,
            call_data: Span<felt252>
        ) -> bool {
            // prevents looping which would lead to excessive minting
            // we only allow a FLASH_MINT_AMOUNT_PCT percentage of total
            // yin to be minted, as per spec
            self.reentrancy_guard.start();

            assert(amount <= self.max_flash_loan(token), 'FM: amount exceeds maximum');

            let shrine = self.shrine.read();

            let amount_wad: Wad = amount.try_into().unwrap();

            // temporarily increase the debt ceiling by the loan amount so that
            // flash loans still work when total yin is at or exceeds the debt ceiling
            let ceiling: Wad = shrine.get_debt_ceiling();
            let total_yin: Wad = shrine.get_total_yin();
            let budget_adjustment: Wad = match shrine.get_budget().try_into() {
                Option::Some(surplus) => { surplus },
                Option::None => { WadZeroable::zero() }
            };
            let adjust_ceiling: bool = total_yin + amount_wad + budget_adjustment > ceiling;
            if adjust_ceiling {
                shrine.set_debt_ceiling(total_yin + amount_wad + budget_adjustment);
            }

            shrine.inject(receiver, amount_wad);

            let initiator: ContractAddress = get_caller_address();

            let borrower_resp: u256 = IFlashBorrowerDispatcher { contract_address: receiver }
                .on_flash_loan(initiator, token, amount, FLASH_FEE, call_data);

            assert(borrower_resp == ON_FLASH_MINT_SUCCESS, 'FM: on_flash_loan failed');

            // This function in Shrine takes care of balance validation
            shrine.eject(receiver, amount_wad);

            if adjust_ceiling {
                shrine.set_debt_ceiling(ceiling);
            }

            self.emit(FlashMint { initiator, receiver, token, amount });

            self.reentrancy_guard.end();

            true
        }
```
Not sure if QA or medium as EIP compliance is broken, albeit no funds are lost.
## Recommended Mitigation Steps

Include a check like is done the `flash_fee` function
```
            assert(self.shrine.read().contract_address == token, "FM: Unsupported token");
```

***
***

# 2. Allowing flash_loan initiators to pass in receiver addresses is not advisable
Links to affected code *

https://github.com/code-423n4/2024-01-opus/blob/4720e9481a4fb20f4ab4140f9cc391a23ede3817/src/core/flash_mint.cairo#L102
## Impact
The `flash_loan` function allows the caller to specify the receiver address. Yin is then minted to the address, and yin + fee is burnt from after the callback. Currently, its not an issue as fees are currently not charged. Doing this leaves the receiver vulnerable to malicious draining of account through fees by flashloaning on behalf of the receiver without their consent. 
## Recommended Mitigation Steps
Consider checking that `initiator` == `receiver` 

***
***

# 3. Roles can be renounced
Links to affected code *

https://github.com/lindy-labs/access_control/blob/5365102134c5d96d5cf67d5f3f4cefd11cbf26f2/src/access_control.cairo#L11
https://github.com/code-423n4/2024-01-opus/blob/4720e9481a4fb20f4ab4140f9cc391a23ede3817/src/core/absorber.cairo#L9
https://github.com/code-423n4/2024-01-opus/blob/4720e9481a4fb20f4ab4140f9cc391a23ede3817/src/core/allocator.cairo#L3
https://github.com/code-423n4/2024-01-opus/blob/4720e9481a4fb20f4ab4140f9cc391a23ede3817/src/core/caretaker.cairo#L3
https://github.com/code-423n4/2024-01-opus/blob/4720e9481a4fb20f4ab4140f9cc391a23ede3817/src/core/controller.cairo#L3
https://github.com/code-423n4/2024-01-opus/blob/4720e9481a4fb20f4ab4140f9cc391a23ede3817/src/core/equalizer.cairo#L3
https://github.com/code-423n4/2024-01-opus/blob/4720e9481a4fb20f4ab4140f9cc391a23ede3817/src/core/purger.cairo#L3
https://github.com/code-423n4/2024-01-opus/blob/4720e9481a4fb20f4ab4140f9cc391a23ede3817/src/core/seer.cairo#L3
https://github.com/code-423n4/2024-01-opus/blob/4720e9481a4fb20f4ab4140f9cc391a23ede3817/src/core/sentinel.cairo#L3
https://github.com/code-423n4/2024-01-opus/blob/4720e9481a4fb20f4ab4140f9cc391a23ede3817/src/core/shrine.cairo#L3
https://github.com/code-423n4/2024-01-opus/blob/4720e9481a4fb20f4ab4140f9cc391a23ede3817/src/external/pragma.cairo#L10

## Impact
The contract implemnets lindy-labs access_control contract which implements the `renounce_role` function. This can represent a certain risk if the role is renounced for any other reason than by design. This will leave the contract without an admin or important roles, thereby removing any functionality that is only available to the these roles. 

## Recommended Mitigation Steps
Consider implementing a function to override this function.

***
***

# 4. Protocol is highly vulnerable to wbtc pausable property
Links to affected code *

https://github.com/code-423n4/2024-01-opus/blob/4720e9481a4fb20f4ab4140f9cc391a23ede3817/src/core/shrine.cairo#L565
## Impact

The protocol relies on WBTC as one of the main collateral tokens for the protocol. Thus, any processes involving the transfer or transferFrom of collateral tokens will be affected. For instance, users looking to improve trove health by depositing more wbtc collateral will not be able to. This can cause accrual of bad debt and lead to unfair liquidations in certain cases. Collateral withrawal and reclaim will also be affected.
## Recommended Mitigation Steps
To prevent unfair liquidations after WBTC unpause, a short downtime period can be put in place on the liquidate function, to allow borrowers to execute their repayments first. 

***
***

# 5. Use of unsafe ERC20 methods
Links to affected code *

https://github.com/code-423n4/2024-01-opus/blob/4720e9481a4fb20f4ab4140f9cc391a23ede3817/src/core/gate.cairo#L145
https://github.com/code-423n4/2024-01-opus/blob/4720e9481a4fb20f4ab4140f9cc391a23ede3817/src/core/gate.cairo#L163
https://github.com/code-423n4/2024-01-opus/blob/4720e9481a4fb20f4ab4140f9cc391a23ede3817/src/core/absorber.cairo#L793
https://github.com/code-423n4/2024-01-opus/blob/4720e9481a4fb20f4ab4140f9cc391a23ede3817/src/core/absorber.cairo#L794
https://github.com/code-423n4/2024-01-opus/blob/4720e9481a4fb20f4ab4140f9cc391a23ede3817/src/core/absorber.cairo#L882
https://github.com/code-423n4/2024-01-opus/blob/4720e9481a4fb20f4ab4140f9cc391a23ede3817/src/core/caretaker.cairo#L358
https://github.com/code-423n4/2024-01-opus/blob/4720e9481a4fb20f4ab4140f9cc391a23ede3817/src/core/sentinel.cairo#L205

## Impact
There are many non-standard ERC20 Tokens that won't work correctly using the standard IERC20 interface.

As an example
```
        fn transfer_assets(ref self: ContractState, to: ContractAddress, mut asset_balances: Span<AssetBalance>) {
            loop {
                match asset_balances.pop_front() {
                    Option::Some(asset_balance) => {
                        if (*asset_balance.amount).is_non_zero() {
                            IERC20Dispatcher { contract_address: *asset_balance.address }
                                .transfer(to, (*asset_balance.amount).into()); 
                        }
                    },
                    Option::None => { break; },
                };
            };
        }
```
The `transfer_assets` functions which is used extensively in the `reap_helper` of the absorber contract upon calling the `provide`, `remove`, `reap` functions will malfunction when certain token types are used as assets. Using a a token that returns bool on failure for instance, will cause that the `transfer_assets` to silently fail, causing loss of funds to the user.

## Recommended Mitigation Steps
Consider implementing safer methods. 

***
***

# 6. Important address updates should have address(0) checks 
Links to affected code *

https://github.com/code-423n4/2024-01-opus/blob/4720e9481a4fb20f4ab4140f9cc391a23ede3817/src/core/abbot.cairo#L79C1-L82C6
https://github.com/code-423n4/2024-01-opus/blob/4720e9481a4fb20f4ab4140f9cc391a23ede3817/src/core/absorber.cairo#L227-L236
https://github.com/code-423n4/2024-01-opus/blob/4720e9481a4fb20f4ab4140f9cc391a23ede3817/src/core/seer.cairo#L135
https://github.com/code-423n4/2024-01-opus/blob/4720e9481a4fb20f4ab4140f9cc391a23ede3817/src/core/equalizer.cairo#L116
## Impact

Parameters of type address in your functions should be checked to ensure that they are not assigned the null address (address(0x0)). Failure to validate these parameters can lead to transaction reverts, wasted gas, the need for transaction resubmission, and may even require redeployment of contracts within the protocol in certain situations. Implement checks for address(0x0) to avoid these potential issues.

***
***

# 7. Centralization risks from single points of failure
Links to affected code *

https://github.com/code-423n4/2024-01-opus/blob/4720e9481a4fb20f4ab4140f9cc391a23ede3817/src/core/roles.cairo#L30
## Impact
Utilizing an externally owned account (EOA) as the owner of contracts poses significant dangers of centralization and represents a vulnerable single point of failure. A single private key is susceptible to theft during a hacking incident, or the sole possessor of the key may encounter difficulties in retrieving it when required. It is advisable to contemplate transitioning to a multi-signature arrangement.

***
***

# 8. Proxy tokens can be added multiple times by passing the 1 yang and 1 reward asset rule.
Links to affected code *

https://github.com/code-423n4/2024-01-opus/blob/4720e9481a4fb20f4ab4140f9cc391a23ede3817/src/core/shrine.cairo#L565
https://github.com/code-423n4/2024-01-opus/blob/4720e9481a4fb20f4ab4140f9cc391a23ede3817/src/core/absorber.cairo#L373
## Impact
Some tokens implement proxies, as such have multiple addresses. These tokens can be added as yangs or as reward assets multiple times potentially causing issues with accounting and breaking the 1 yang and 1 reward asset rule. Such tokens can also temporarily escape the yang suspensions.

## Recommended Mitigation Steps
Consider introducing checks for these tokens

***
***

# 9. Introduce timelocks and sanity checks
Links to affected code *
https://github.com/code-423n4/2024-01-opus/blob/4720e9481a4fb20f4ab4140f9cc391a23ede3817/src/core/sentinel.cairo#L216
https://github.com/code-423n4/2024-01-opus/blob/4720e9481a4fb20f4ab4140f9cc391a23ede3817/src/core/shrine.cairo#L620
https://github.com/code-423n4/2024-01-opus/blob/4720e9481a4fb20f4ab4140f9cc391a23ede3817/src/core/shrine.cairo#L652
https://github.com/code-423n4/2024-01-opus/blob/4720e9481a4fb20f4ab4140f9cc391a23ede3817/src/core/shrine.cairo#L754
https://github.com/code-423n4/2024-01-opus/blob/4720e9481a4fb20f4ab4140f9cc391a23ede3817/src/core/shrine.cairo#L772
https://github.com/code-423n4/2024-01-opus/blob/4720e9481a4fb20f4ab4140f9cc391a23ede3817/src/core/shrine.cairo#L781
https://github.com/code-423n4/2024-01-opus/blob/4720e9481a4fb20f4ab4140f9cc391a23ede3817/src/core/shrine.cairo#L789
https://github.com/code-423n4/2024-01-opus/blob/4720e9481a4fb20f4ab4140f9cc391a23ede3817/src/core/shrine.cairo#L800
## Recommended Mitigation Steps

Consider introducing sanity checks to prevent user griefing in cases of unfavourable settings. 
A timelock can also be introduced to give users enough time to react to these changes.

***
***

# 11. Should include a check that yang belongs to the pair, to avoid pairing up yangs with wrong oracle addresse
Links to affected code *

https://github.com/code-423n4/2024-01-opus/blob/4720e9481a4fb20f4ab4140f9cc391a23ede3817/src/external/pragma.cairo#L152
## Impact
The `set_yang_pair_id` is used to pair an oracle address to the yang address. An issue can occur if the wrong address is paired to a yang, e.g pairing wstETH/USD address with WBTC, which can lead to wrong pricing and protocol destablization. 
```
            self.yang_pair_ids.write(yang, pair_id);
```
## Recommended Mitigation Steps
Consider implementing a check that the yang belongs to the oracle pair id.
***
***

# 12. Introduce a check in `add_yang` function to prevent yin from being used as yang 
Links to affected code *

https://github.com/code-423n4/2024-01-opus/blob/4720e9481a4fb20f4ab4140f9cc391a23ede3817/src/core/shrine.cairo#L565
## Impact
The `add_yang` function allows for adding a new token to function as collateral. Currently, there's no check preventing that the yin is added as a yang, this introduces the possibility of minting yins backed by yins. This can inflate the total supply of yins without any asset actual backing. Either by intent or error, adding the yin token as a yang will seriously destabilize the protocol.


## Recommended Mitigation Steps
Add a check in the `add_yang` token checking that the yang token != yin.

***
***

# 13. Should also check if suspension status is none to prevent unsuspending "unsuspended" yangs
Links to affected code *
https://github.com/code-423n4/2024-01-opus/blob/4720e9481a4fb20f4ab4140f9cc391a23ede3817/src/core/shrine.cairo#L639-L640
## Impact

The `unsuspend_yang` function checks for yangsuspensionstatus before unsuspending. It however checks if its not permanently suspended. Doing this causes that unsuspended yangs i.e yangs with yangsuspensionstatus None can be unsuspended. Consider checking instead for temporary status.
```
        fn unsuspend_yang(ref self: ContractState, yang: ContractAddress) { 
            self.access_control.assert_has_role(shrine_roles*9::UPDATE_YANG_SUSPENSION);
            assert(
                self.get_yang_suspension_status(yang) != YangSuspensionStatus::Permanent, "SH: Suspension is permanent"
            );
            self.yang_suspension.write(self.get_valid_yang_id(yang), 0);
            self.emit(YangUnsuspended { yang, timestamp: get_block_timestamp() });
        }
```
## Recommended Mitigation Steps
```

            assert(
                self.get_yang_suspension_status(yang) = YangSuspensionStatus::Temporary, "Unsuspension impossible"
            );
```
