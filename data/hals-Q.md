# Opus QA Report

| ID            | Title                                                                                                                                                   | Severity |
| ------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------- | -------- |
| [L-01](#l-01) | Trove's owner will be charge interest twice when closing the trove                                                                                      | Low      |
| [L-02](#l-02) | `abbot.deposit` function: troves can be charged without having actual deposited amount                                                                  | Low      |
| [L-03](#l-03) | `abbot.withdraw` function lacks slippage control                                                                                                        | Low      |
| [L-04](#l-04) | `sentinel.INITIAL_DEPOSIT_AMT` doesn't prevent inflation attack for all collaterals                                                                     | Low      |
| [L-05](#l-05) | `initial_total_yang` assigned in the shrine doesn't equal the amount deposited in the gate for some tokens which will result in wrong yang calculations | Low      |
| [L-06](#l-06) | If a gate is killed; its underlying yang can't be added with a new gate again                                                                           | Low      |
| [L-07](#l-07) | The protocol only supports tokens with decimals                                                                                                         | Low      |
| [L-08](#l-08) | Some collateral tokens might not return values on transfers so they can't be adopted by the protocol                                                    | Low      |
| [L-09](#l-09) | `seer` contract assumes that all oracles will return the price with same decimals                                                                       | Low      |
| [L-10](#l-10) | `seer.update_prices_internal` function updates `last_update_prices_call_timestamp` even if the prices are not all updated                               | Low      |
| [L-11](#l-11) | `seer` contract assumes that all pairs and oracles have the same heartbeat (`update_frequency`)                                                         | Low      |
| [L-12](#l-12) | `purger.liquidate` function lacks slippage mechanism                                                                                                    | Low      |
| [L-13](#l-13) | `purger` contract: wrong value assigned for the `COMPENSATION_CAP` constant                                                                             | Low      |
| [L-14](#l-14) | Approve race condition in `approve_helper` funtion                                                                                                      | Low      |

## [L-01] Trove's owner will be charge interest twice when closing the trove <a id="l-01" ></a>

## Details

- `abbot.close_trove` function allows troves owners from closing their troves after repaying its debt in full and withdrawing all their collaterals.

- `shrine.charge` function is called to charge the trove with intreset whenever an action is done on the trove, but it was noticed that when closing the trove, it will be charged twice:

  - first via `shrine.melt()`:

    ```javascript
    // Charge interest
    self.charge(trove_id);
    ```

    - second via `shrine.withdraw()`:

    ```javascript
    self.charge(trove_id);
    ```

- This will result in users paying more yin to close their troves as they will be charged more interest.

## Proof of Concept

[abbot.close_trove function](https://github.com/code-423n4/2024-01-opus/blob/4720e9481a4fb20f4ab4140f9cc391a23ede3817/src/core/abbot.cairo#L163C9-L189C1)

```javascript
// close a trove, repaying its debt in full and withdrawing all the Yangs
        fn close_trove(ref self: ContractState, trove_id: u64) {
            let user = get_caller_address();
            self.assert_trove_owner(user, trove_id);

            let shrine = self.shrine.read();
            // melting "max Wad" to instruct Shrine to melt *all* of trove's debt
            shrine.melt(user, trove_id, BoundedWad::max());

            let mut yangs: Span<ContractAddress> = self.sentinel.read().get_yang_addresses();
            // withdraw each and every Yang belonging to the trove from the system
            loop {
                match yangs.pop_front() {
                    Option::Some(yang) => {
                        let yang_amount: Wad = shrine.get_deposit(*yang, trove_id);
                        if yang_amount.is_zero() {
                            continue;
                        }
                        self.withdraw_helper(trove_id, user, *yang, yang_amount);
                    },
                    Option::None => { break; }
                };
            };

            self.emit(TroveClosed { trove_id });
        }

```

[abbot.withdraw_helper function](https://github.com/code-423n4/2024-01-opus/blob/4720e9481a4fb20f4ab4140f9cc391a23ede3817/src/core/abbot.cairo#L251C9-L261C10)

```javascript
fn withdraw_helper(
            ref self: ContractState, trove_id: u64, user: ContractAddress, yang: ContractAddress, yang_amt: Wad
        ) {
            // reentrancy guard is used as a precaution
            self.reentrancy_guard.start();

            self.sentinel.read().exit(yang, user, trove_id, yang_amt);
            self.shrine.read().withdraw(yang, trove_id, yang_amt);

            self.reentrancy_guard.end();
        }
```

[shrine.melt function/L887-L888](https://github.com/code-423n4/2024-01-opus/blob/4720e9481a4fb20f4ab4140f9cc391a23ede3817/src/core/shrine.cairo#L887-L888)

```javascript
// Charge interest
self.charge(trove_id);
```

[shrine.withdraw_helper function/L1389](https://github.com/code-423n4/2024-01-opus/blob/4720e9481a4fb20f4ab4140f9cc391a23ede3817/src/core/shrine.cairo#L1389)

```javascript
self.charge(trove_id);
```

## Recommendation

Update `abbot.close_trove` function (and the related `shrine` contract functions) to ensure that the user is only charged once for this operation.

## [L-02] `abbot.deposit` function: troves can be charged without having actual deposited amount <a id="l-02" ></a>

## Details

- `abbot.deposit_helper` function is meant to handle the deposit logic for a trove by interacting with the `gate` and `shrine` contracts, and this function is called in:
  - `abbot.open_trove` function: when the trove owner opens a trove by depositing a whitelisted collateral/collaterals and gain shares (yangs).
  - `abbot.deposit` function that allows anyone from depositing collaterals in any trove without being the trove owner.
- Knowing that with each trove operation the trove will be charged with interest; then anyone can deposit **zero** collateral in any trove and causing it to be charged with interest without having any actual amount deposited as there's no check in `abbot.deposit` function if the deposited collateral is > 0.

## Proof of Concept

[abbot.deposit function](https://github.com/code-423n4/2024-01-opus/blob/4720e9481a4fb20f4ab4140f9cc391a23ede3817/src/core/abbot.cairo#L190C8-L201C1)

```javascript
 // add Yang (an asset) to a trove
        fn deposit(ref self: ContractState, trove_id: u64, yang_asset: AssetBalance) {
            // There is no need to check the yang address is non-zero because the
            // Sentinel does not allow a zero address yang to be added.

            assert(trove_id != 0, 'ABB: Trove ID cannot be 0');
            assert(trove_id <= self.troves_count.read(), 'ABB: Non-existent trove');
            // note that caller does not need to be the trove's owner to deposit

            self.deposit_helper(trove_id, get_caller_address(), yang_asset);
        }
```

[abbot.deposit_helper function](https://github.com/code-423n4/2024-01-opus/blob/4720e9481a4fb20f4ab4140f9cc391a23ede3817/src/core/abbot.cairo#L240C4-L248C10)

```javascript
     fn deposit_helper(ref self: ContractState, trove_id: u64, user: ContractAddress, yang_asset: AssetBalance) {
            // reentrancy guard is used as a precaution
            self.reentrancy_guard.start();

            let yang_amt: Wad = self.sentinel.read().enter(yang_asset.address, user, trove_id, yang_asset.amount);
            self.shrine.read().deposit(yang_asset.address, trove_id, yang_amt);

            self.reentrancy_guard.end();
        }
```

[shrine.deposit function/L817](https://github.com/code-423n4/2024-01-opus/blob/4720e9481a4fb20f4ab4140f9cc391a23ede3817/src/core/shrine.cairo#L817)

```javascript
self.charge(trove_id);
```

## Recommendation

Update `abbot.deposit` function to check if the deposited amount greater than a minimum amount determined by the protocol for each collateral type.

## [L-03] `abbot.withdraw` function lacks slippage control <a id="l-03" ></a>

## Details

- `abbot.withdraw` function is meant to enable trove owners from withdrawing all/part of their collaterals as long as their trove remains healthy.
- It first calculate the amount of yang corresponding to the required withdrawn amount of assets via `sentinel.convert_to_yang` function, where this yang amount will be **burnt** from the user trove.
- But it was noticed that there's no check on the returned amount of the equivalent yang if it's acceptable by the user or not as the function doesn't implement any check for a maximum amount of yang to be burnt during withdrawal.

## Proof of Concept

[abbot.withdraw function](https://github.com/code-423n4/2024-01-opus/blob/4720e9481a4fb20f4ab4140f9cc391a23ede3817/src/core/abbot.cairo#L202C8-L212C10)

```javascript
 // remove Yang (an asset) from a trove
        fn withdraw(ref self: ContractState, trove_id: u64, yang_asset: AssetBalance) {
            // There is no need to check the yang address is non-zero because the
            // Sentinel does not allow a zero address yang to be added.

            let user = get_caller_address();
            self.assert_trove_owner(user, trove_id);

            let yang_amt: Wad = self.sentinel.read().convert_to_yang(yang_asset.address, yang_asset.amount);
            self.withdraw_helper(trove_id, user, yang_asset.address, yang_amt);
        }
```

## Recommendation

Update `abbot.withdraw` function to have a maximum yang to be burnt from the trove for each asset.

## [L-04] `sentinel.INITIAL_DEPOSIT_AMT` doesn't prevent inflation attack for all collaterals <a id="l-04" ></a>

## Details

- `sentinel.INITIAL_DEPOSIT_AMT` represents the amount that is going to be initially deposited in the `gate` contract to prevent first-depositor-attack (inflation attack) when calculating yang shares, and this value is set to:

  ```javascript
  const INITIAL_DEPOSIT_AMT: u128 = 1000;
  ```

- When the `sentinel.add_yang` function is invoked by the `ADD_YANG` role to add a new asset (collateral) to the system; an ininital asset amount is deposited from the caller to the new asset gate:

  ```javascript
  let initial_deposit_amt: u256 = INITIAL_DEPOSIT_AMT.into();

  let caller: ContractAddress = get_caller_address();
  let success: boolean = yang_erc20.transfer_from(
    caller,
    gate.contract_address,
    initial_deposit_amt
  );
  ```

- Knowing that the protocol will initially adopt `WBTC`, `ETH` and `wstETH` tokens as collaterals, where:

  - `BTC` (`WBTC` as well) has [**8 decimals**](https://starkscan.co/token/0x03fe2b97c1fd336e750087d68b9b867997fd64a2661ff3ca5a7c771641e8e7ac)

  - `ETH` has [**18 decimals**](https://starkscan.co/token/0x049d36570d4e46f48e99674bd3fcc84644ddd6b96f7c741b1562b82f9e004dc7)

- So the `INITIAL_DEPOSIT_AMT` of `WBTC` is going to be 1000/1e8 which is `0.00001`, while the `INITIAL_DEPOSIT_AMT` of `ETH` is going to be 1000/1e18 which is `0.000000000000001` which is much smaller and has lesser value than `WBTC`; hence it **will not provide sufficient protection against inflation/first depositor attack**.

## Proof of Concept

[sentinel.INITIAL_DEPOSIT_AMT constant](https://github.com/code-423n4/2024-01-opus/blob/4720e9481a4fb20f4ab4140f9cc391a23ede3817/src/core/sentinel.cairo#L33)

```javascript
const INITIAL_DEPOSIT_AMT: u128 = 1000;
```

[sentinel.add_yang function/L202-L205](https://github.com/code-423n4/2024-01-opus/blob/4720e9481a4fb20f4ab4140f9cc391a23ede3817/src/core/sentinel.cairo#L202-L205)

```javascript
let initial_deposit_amt: u256 = INITIAL_DEPOSIT_AMT.into();

let caller: ContractAddress = get_caller_address();
let success: boolean = yang_erc20.transfer_from(
  caller,
  gate.contract_address,
  initial_deposit_amt
);
```

## Recommendation

Update `sentinel.add_yang` function to have another parameter for `initial_deposit_amount` to allow assigning it dynamically for each added collateral instead of adopting a one-constant-fits-all `INITIAL_DEPOSIT_AMT`.

## [L-05] `initial_total_yang` assigned in the shrine doesn't equal the amount deposited in the gate for some tokens which will result in wrong yang calculations <a id="l-05" ></a>

## Details

- `sentinel.INITIAL_DEPOSIT_AMT` represents the amount that is going to be initially deposited in the `gate` contract to prevent first-depositor-attack (inflation attack) when calculating yang shares, and this value is set to:

  ```javascript
  const INITIAL_DEPOSIT_AMT: u128 = 1000;
  ```

- When the `sentinel.add_yang` function is invoked by the `ADD_YANG` role to add a new asset (collateral) to the system; an ininital asset amount is deposited from the caller to the new asset gate:

  ```javascript
  let initial_deposit_amt: u256 = INITIAL_DEPOSIT_AMT.into();

  let caller: ContractAddress = get_caller_address();
  let success: boolean = yang_erc20.transfer_from(
    caller,
    gate.contract_address,
    initial_deposit_amt
  );
  ```

  and the new asset is added to the shrine with an `initial_yang_amt`:

  ```javascript
  let initial_yang_amt: Wad = fixed_point_to_wad(
    INITIAL_DEPOSIT_AMT,
    yang_erc20.decimals()
  );
  //some code...
  shrine.add_yang(
    yang,
    yang_threshold,
    yang_price,
    yang_rate,
    initial_yang_amt
  );
  ```

- As can be noticed; the `initial_yang_amt` is calculated via `fixed_point_to_wad` based on the token decimals (`1000*1e(18-assetDecimal)`):

  ```javascript
  fn fixed_point_to_wad(n: u128, decimals: u8) -> Wad {
      assert(decimals <= WAD_DECIMALS, 'More than 18 decimals');
      let scale: u128 = pow(10_u128, WAD_DECIMALS - decimals);
      (n * scale).into()
  }
  ```

- Knowing that the protocol will initially adopt `WBTC`, `ETH` and `wstETH` tokens as collaterals, where:

  - `BTC` (`WBTC` as well) has [**8 decimals**](https://starkscan.co/token/0x03fe2b97c1fd336e750087d68b9b867997fd64a2661ff3ca5a7c771641e8e7ac)

  - `ETH` has [**18 decimals**](https://starkscan.co/token/0x049d36570d4e46f48e99674bd3fcc84644ddd6b96f7c741b1562b82f9e004dc7)

- So the `initial_yang_amt` will be:

  - for `WBTC`: `1000*1e10`
  - for `ETH` : `1000`

- And as can be noticed; the `initial_yang_amt` for `WBTC` is much greater than that for `ETH`, where this value is going to be assigned to the `shrine.yang_total` to prevent first depositor front running.

- But this will result in wrong calculation of shares (yang) for `WBTC` asset: as the amount deposited in the gate via direct transfer (`WBTC` gate balance) is `1000`, which is way less than the amount of shares assigned in the shrine (`1000*1e10`), so this will result in depositors for this asset getting way more yang amount for their deposits than intended:

```javascript
// Helper function to calculate the amount of yang corresponding to the given
        // amount of assets.
        // `asset_amt` is denominated in the decimals of the asset.
        fn convert_to_yang_helper(self: @ContractState, asset_amt: u128) -> Wad {
            let asset: IERC20Dispatcher = self.asset.read();

            // @audit-issue amount of total_yang_helper in the shrine, which is initially set to 1000*1e10 for WBTC
            let total_yang: Wad = self.get_total_yang_helper(asset.contract_address);


            if total_yang.is_zero() {
                let decimals: u8 = asset.decimals();
                // Otherwise, scale `asset_amt` up by the difference to match `Wad`
                // precision of yang. If asset is of `Wad` precision, then the same
                // value is returned
                fixed_point_to_wad(asset_amt, decimals)
            } else {

                // @audit-issue total_yang for WBTC will be 1000*1e10, and get_total_assets_helper(asset) will be 1000
                //so this will be for the first depositor: asset_amount * 1000*1e10/1000
                // which is asset_amount*1e10
                (asset_amt.into() * total_yang) / get_total_assets_helper(asset).into()
            }
        }
```

- While for `ETH`: the amount deposited in the gate is equal to the `initial_yang_amt` added in the shrine , and it's `1000`.

- So as can be noticed; this issue is present for any collateral with decimals != 18, as the amount of collateral desposited in the gate will be less than the amount of `initial_yang_amt` assigned for that yang in the `shrine` contract.

## Proof of Concept

[sentinel.INITIAL_DEPOSIT_AMT constant](https://github.com/code-423n4/2024-01-opus/blob/4720e9481a4fb20f4ab4140f9cc391a23ede3817/src/core/sentinel.cairo#L33)

```javascript
const INITIAL_DEPOSIT_AMT: u128 = 1000;
```

[sentinel.add_yang function/L198-L210](https://github.com/code-423n4/2024-01-opus/blob/4720e9481a4fb20f4ab4140f9cc391a23ede3817/src/core/sentinel.cairo#L198-L210)

```javascript
// Require an initial deposit when adding a yang to prevent first depositor from front-running
            let yang_erc20 = IERC20Dispatcher { contract_address: yang };
            // scale `asset_amt` up by the difference to match `Wad` precision of yang
            let initial_yang_amt: Wad = fixed_point_to_wad(INITIAL_DEPOSIT_AMT, yang_erc20.decimals());
            let initial_deposit_amt: u256 = INITIAL_DEPOSIT_AMT.into();

            let caller: ContractAddress = get_caller_address();
            let success: bool = yang_erc20.transfer_from(caller, gate.contract_address, initial_deposit_amt);
            assert(success, 'SE: Yang transfer failed');

            let shrine: IShrineDispatcher = self.shrine.read();
            shrine.add_yang(yang, yang_threshold, yang_price, yang_rate, initial_yang_amt);
```

## Recommendation

Update `sentinel.add_yang` function to have another parameter for `initial_deposit_amount` to allow assigning it dynamically for each added collateral instead of adopting a one-constant-fits-all `INITIAL_DEPOSIT_AMT`.

## [L-06] If a gate is killed; its underlying yang can't be added with a new gate again <a id="l-06" ></a>

## Details

- The `KILL_GATE` role of the `sentinel` contract can kill a gate and its relevant collateral (yang) via `sentinel.kill_gate` function, where it will permanently pauses `enter` for a gate. This prevents users from depositing further amounts of that yang's collateral token:

  ```javascript
          fn kill_gate(ref self: ContractState, yang: ContractAddress) {
              self.access_control.assert_has_role(sentinel_roles::KILL_GATE);

              self.yang_is_live.write(yang, false);

              self.emit(GateKilled { yang, gate: self.yang_to_gate.read(yang).contract_address });
          }
  ```

- But killing a gate will prevent adding it's underlying collateral token (yang) again with a new gate, since `sentinel.add_yang` function will check if the `yang_to_gate` exists or not before adding a new yang while `sentinel.kill_gate` doesn't deattach the yang from the killed gate:

  ```javascript
  assert(
    self.yang_to_gate.read(yang).contract_address.is_zero(),
    "SE: Yang already added"
  );
  ```

## Proof of Concept

[sentinel.kill_gate function](https://github.com/code-423n4/2024-01-opus/blob/4720e9481a4fb20f4ab4140f9cc391a23ede3817/src/core/sentinel.cairo#L228-L234)

```javascript
        fn kill_gate(ref self: ContractState, yang: ContractAddress) {
            self.access_control.assert_has_role(sentinel_roles::KILL_GATE);

            self.yang_is_live.write(yang, false);

            self.emit(GateKilled { yang, gate: self.yang_to_gate.read(yang).contract_address });
        }
```

[sentinel.add_yang function/L186](https://github.com/code-423n4/2024-01-opus/blob/4720e9481a4fb20f4ab4140f9cc391a23ede3817/src/core/sentinel.cairo#L186)

```javascript
assert(
  self.yang_to_gate.read(yang).contract_address.is_zero(),
  "SE: Yang already added"
);
```

## Recommendation

Add a mechanism that enables adding a yang again to the protocol when its gate is killed.

## [L-07] The protocol only supports tokens with decimals <= 18 <a id="l-07" ></a>

## Details

- When the `sentinel.add_yang` function is invoked by the `ADD_YANG` role to add a new asset (collateral) to the system; an ininital asset amount is deposited from the caller to the new asset gate, and an initial_yang_amount is added to the shrine to prevent first depositor/inflation attack:

  ```javascript
  let initial_yang_amt: Wad = fixed_point_to_wad(
    INITIAL_DEPOSIT_AMT,
    yang_erc20.decimals()
  );
  //some code...
  shrine.add_yang(
    yang,
    yang_threshold,
    yang_price,
    yang_rate,
    initial_yang_amt
  );
  ```

- As can be noticed; the `initial_yang_amt` is calculated via `fixed_point_to_wad` based on the token decimals (`1000*1e(18-assetDecimal)`):

  ```javascript
  fn fixed_point_to_wad(n: u128, decimals: u8) -> Wad {
      assert(decimals <= WAD_DECIMALS, 'More than 18 decimals');
      let scale: u128 = pow(10_u128, WAD_DECIMALS - decimals);
      (n * scale).into()
  }
  ```

- So if the token decimals is greater than 18; the yang can't be added to the protocol, which will limit the number of supported collaterals in the future.

## Proof of Concept

[math.fixed_point_to_wad function](https://github.com/code-423n4/2024-01-opus/blob/4720e9481a4fb20f4ab4140f9cc391a23ede3817/src/utils/math.cairo#L25C1-L29C2)

```javascript
fn fixed_point_to_wad(n: u128, decimals: u8) -> Wad {
    assert(decimals <= WAD_DECIMALS, 'More than 18 decimals');
    let scale: u128 = pow(10_u128, WAD_DECIMALS - decimals);
    (n * scale).into()
}
```

## Recommendation

Add another mechanism to support tokens with decimals > 18.

## [L-08] Some collateral tokens might not return values on transfers so they can't be adopted by the protocol<a id="l-08" ></a>

## Details

- The protocol assumes that all collateral tokens will return a value of: `1` if the transfer is successful and `0` if not, where the returned value is checked to ensure the transfer is successful (`1` is returned) before proceeding.

- This might not be the case for other collateral tokens that might be added/expected to be added in the future, as they will not return a value when the asset is transferred, which will make them not being adopted by the protocol.

## Proof of Concept

[sentinel.add_yang function/L205-L206](https://github.com/code-423n4/2024-01-opus/blob/4720e9481a4fb20f4ab4140f9cc391a23ede3817/src/core/sentinel.cairo#L205-L206)

```javascript
let success: boolean = yang_erc20.transfer_from(
  caller,
  gate.contract_address,
  initial_deposit_amt
);
assert(success, "SE: Yang transfer failed");
```

## Recommendation

Add another mechanism to handle tokens that doesn't return a value upon successful/unsuccessful transfers.

## [L-09] `seer` contract assumes that all oracles will return the price with same decimals <a id="l-09" ></a>

## Details

- `seer` contract acts as a coordinator of individual oracle modules, reading the price of the underlying collateral tokens of yangs from the adapter modules of oracles and submitting them to the `shrine` contract.

- Currently the protocol will use only one oracle for each pair (collateral/USD), which is `pragma` oracle, where it will return the pair price of the currently adopted assets in 8 decimals: `BTC/USD` , `WSTETH/USD` and `ETH/USD` [pairs](https://docs.pragmaoracle.com/Resources/Cairo%201/data-feeds/supported-assets#spot).

- But if fall-back oracles are added in the future where the returned pair price decimals is different; this will result in inconsistency of the assigned prices when the two oracles used to extract the price of the same asset return different decimals as the `seer.update_prices_internal` function is not capable of handelling this case as it assumes that all oracles will return the same pair price decimals.

- To explain this inconsistency, let's see the following scenario :
  1.  asset A has assigned oracle#1 that returns A/USD price in 8 decimals, and it is assigned another fall-back oracle#2 that returns A/USD price in 10 decimals.
  2.  In the first price update run, oracle#1 returned the price with 8 decimals.
  3.  In the second price update run, oracle#1 failed to fetch the price, so the fall-back oracle#2 is called to fetch the price with 10 decimals.

## Proof of Concept

[seer.update_prices_internal function/L220-L231](https://github.com/code-423n4/2024-01-opus/blob/4720e9481a4fb20f4ab4140f9cc391a23ede3817/src/core/seer.cairo#L220-L231)

```javascript
 match oracle.fetch_price(*yang, force_update) {
                                Result::Ok(oracle_price) => {
                                    let asset_amt_per_yang: Wad = sentinel.get_asset_amt_per_yang(*yang);
                                    let price: Wad = oracle_price * asset_amt_per_yang;
                                    shrine.advance(*yang, price);
                                    self.emit(PriceUpdate { oracle: oracle.contract_address, yang: *yang, price });
                                    break;
                                },
                                // try next oracle for this yang
                                Result::Err(_) => { oracle_index += 1; }
                            }
```

## Recommendation

Add a mechanism to ensure that all adopted oracles that fetch the price of a one asset return the same decimals (consistent).

## [L-10] `seer.update_prices_internal` function updates `last_update_prices_call_timestamp` even if the prices are not all updated <a id="l-10" ></a>

## Details

- `seer.update_prices_internal` function is called whenever the assets prices are going to be updated either automatically (enforced) when absorbtion occures, or manually called if the time to update the prices comes, and this is done via `seer.execute_task` function, where the prices of all assets are updated in one call.

- `seer.execute_task` function checks first if the time passed from the last prices update time is greater than `update_frequency` (which represents the minimal time difference in seconds of how often the price is to be fetched from the oracle):

  ```javascript
          fn probe_task(self: @ContractState) -> bool {
              let seconds_since_last_update: u64 = get_block_timestamp() - self.last_update_prices_call_timestamp.read();
              self.update_frequency.read() <= seconds_since_last_update
          }
  ```

- And if it has been too long from the last prices update (when `self.update_frequency.read() <= seconds_since_last_update`); the prices will be fetched from the oracles and updated in the shrine:

  ```javascript
          fn execute_task(ref self: ContractState) {
              assert(self.probe_task(), "SEER: Too soon to update prices");
              self.update_prices_internal(false);
          }
  ```

- The `last_update_prices_call_timestamp` will be updated whenever `seer.update_prices_internal` function is called:

  ```javascript
  self.last_update_prices_call_timestamp.write(get_block_timestamp());
  ```

-`last_update_prices_call_timestamp` is updated regardless of all the assets prices being updated or not:

```javascript
fn update_prices_internal(ref self: ContractState, force_update: bool) {
            let shrine: IShrineDispatcher = self.shrine.read();
            let sentinel: ISentinelDispatcher = self.sentinel.read();

            // loop through all yangs
            // for each yang, loop through all oracles until a
            // valid price update is fetched, in which case, call shrine.advance()
            // the expectation is that the primary oracle will provide a
            // valid price in most cases, but if not, we can fallback to other oracles
            let mut yangs: Span<ContractAddress> = sentinel.get_yang_addresses();
            loop {
                match yangs.pop_front() {
                    Option::Some(yang) => {
                        let mut oracle_index: u32 = LOOP_START;
                        loop {
                            let oracle: IOracleDispatcher = self.oracles.read(oracle_index);
                            if oracle.contract_address.is_zero() {
                                // if branch happens, it means no oracle was able to
                                // fetch a price for yang, i.e. we're missing a price update
                                self.emit(PriceUpdateMissed { yang: *yang });
                                break;
                            }

                            // TODO: when possible in Cairo, fetch_price should be wrapped
                            //       in a try-catch block so that an exception does not
                            //       prevent all other price updates

                            match oracle.fetch_price(*yang, force_update) {
                                Result::Ok(oracle_price) => {
                                    let asset_amt_per_yang: Wad = sentinel.get_asset_amt_per_yang(*yang);
                                    let price: Wad = oracle_price * asset_amt_per_yang;
                                    shrine.advance(*yang, price);
                                    self.emit(PriceUpdate { oracle: oracle.contract_address, yang: *yang, price });
                                    break;
                                },
                                // try next oracle for this yang
                                Result::Err(_) => { oracle_index += 1; }
                            }
                        };
                    },
                    Option::None => { break; }
                };
            };

            self.last_update_prices_call_timestamp.write(get_block_timestamp());
            self.emit(UpdatePricesDone { forced: force_update });
        }
```

As can be seen, the function is going to loop over the oracles of each asset trying to fetch the price from one of them, and if the asset price is updated or if all asset oracles failed to fetch the price; the loop will break to move to the next asset to update its price, and once all the assets are looped over; the `last_update_prices_call_timestamp` will be set to the current timestamp.

- So if one asset price is failed to be updated, it will continue to use its old price since `last_update_prices_call_timestamp` will be set to the current timestamp regardless of all prices being updated or some (or none) of the prices being updated.

- This will result in preventing updating assets prices that failed to be updated (fetched) from the previous run for a long time, and this will lead to using stale prices before update is allowed.

## Proof of Concept

[seer.update_prices_internal function](https://github.com/code-423n4/2024-01-opus/blob/4720e9481a4fb20f4ab4140f9cc391a23ede3817/src/core/seer.cairo#L193-L239)

```javascript
fn update_prices_internal(ref self: ContractState, force_update: bool) {
            let shrine: IShrineDispatcher = self.shrine.read();
            let sentinel: ISentinelDispatcher = self.sentinel.read();

            // loop through all yangs
            // for each yang, loop through all oracles until a
            // valid price update is fetched, in which case, call shrine.advance()
            // the expectation is that the primary oracle will provide a
            // valid price in most cases, but if not, we can fallback to other oracles
            let mut yangs: Span<ContractAddress> = sentinel.get_yang_addresses();
            loop {
                match yangs.pop_front() {
                    Option::Some(yang) => {
                        let mut oracle_index: u32 = LOOP_START;
                        loop {
                            let oracle: IOracleDispatcher = self.oracles.read(oracle_index);
                            if oracle.contract_address.is_zero() {
                                // if branch happens, it means no oracle was able to
                                // fetch a price for yang, i.e. we're missing a price update
                                self.emit(PriceUpdateMissed { yang: *yang });
                                break;
                            }

                            // TODO: when possible in Cairo, fetch_price should be wrapped
                            //       in a try-catch block so that an exception does not
                            //       prevent all other price updates

                            match oracle.fetch_price(*yang, force_update) {
                                Result::Ok(oracle_price) => {
                                    let asset_amt_per_yang: Wad = sentinel.get_asset_amt_per_yang(*yang);
                                    let price: Wad = oracle_price * asset_amt_per_yang;
                                    shrine.advance(*yang, price);
                                    self.emit(PriceUpdate { oracle: oracle.contract_address, yang: *yang, price });
                                    break;
                                },
                                // try next oracle for this yang
                                Result::Err(_) => { oracle_index += 1; }
                            }
                        };
                    },
                    Option::None => { break; }
                };
            };

            self.last_update_prices_call_timestamp.write(get_block_timestamp());
            self.emit(UpdatePricesDone { forced: force_update });
        }
```

## Recommendation

Add a `last_update_prices_call_timestamp` variable for each asset, and update this value only if the price is updated, and allow updating the price of each asset individually.

## [L-11] `seer` contract assumes that all pairs and oracles have the same heartbeat (`update_frequency`) <a id="l-11" ></a>

## Details

- `seer` contract has a mechanism to update all assets prices in one call, and there's no function to update an asset price individually.

- The assets prices are updated in **one call** when the time passed from the last update is greater than `update_frequency` (which represents the minimal time difference in seconds of how often the price is to be fetched from the oracle).

- But this is not going to be the case for all assets; as some asset pairs might have a heartbeat longer/or shorter than the others, and unifying the update to be done at the same time for all assets **wrongly** assumes that all asset pairs of the used oracle have the same heartbeat.

- Knowing that the protocol currently uses pragma oracle to fetch pairs prices; so assuming that all pairs of this oracle will have the same heartbeat will not be necessarily the case when other oracles are used by the protocol.

## Proof of Concept

[seer.update_frequency functivariable](https://github.com/code-423n4/2024-01-opus/blob/4720e9481a4fb20f4ab4140f9cc391a23ede3817/src/core/seer.cairo#L51C9-L53C31)

```javascript
// The minimal time difference in seconds of how often we
// want to fetch from the oracle.
 update_frequency: u64,
```

[seer.probe_task & seer.execute_task functions](https://github.com/code-423n4/2024-01-opus/blob/4720e9481a4fb20f4ab4140f9cc391a23ede3817/src/core/seer.cairo#L176C1-L184C10)

```javascript
        fn probe_task(self: @ContractState) -> bool {
            let seconds_since_last_update: u64 = get_block_timestamp() - self.last_update_prices_call_timestamp.read();
            self.update_frequency.read() <= seconds_since_last_update
        }

        fn execute_task(ref self: ContractState) {
            assert(self.probe_task(), 'SEER: Too soon to update prices');
            self.update_prices_internal(false);
        }
```

## Recommendation

Add a separate `update_frequency` for each asset pairs, and check it in `update_prices_internal` function before updating the asset price.

## [L-12] `purger.liquidate` function lacks slippage mechanism<a id="l-12" ></a>

## Details

- `purger.liquidate` function is called by a liquidator to liquidate unhealthy trove by partially/fully paying its `yin` debt in order to get the freed trove collaterals on a discount.

- But it was noticed that this function doesn't implement a slippage mechanism, where the user is going to get a collateral amount of at least a minimum value determined by him as the collateral price might increase (while the trove is still unhealthy) resulting in the liquidator getting less collateral than he wished for.

## Proof of Concept

[purger.liquidate function](https://github.com/code-423n4/2024-01-opus/blob/4720e9481a4fb20f4ab4140f9cc391a23ede3817/src/core/purger.cairo#L222C9-L224C34)

```javascript
fn liquidate(
            ref self: ContractState, trove_id: u64, amt: Wad, recipient: ContractAddress
        ) -> Span<AssetBalance> {
```

## Recommendation

Update `purger.liquidate` function to have a maximum yang to be burnt from the trove for each liquidated asset.

## [L-13] `purger` contract: wrong value assigned for the `COMPENSATION_CAP` constant <a id="l-13" ></a>

## Details

- As per the protocol [documentation](https://demo-35.gitbook.io/untitled/smart-contracts/purger-module#description-of-key-functions:~:text=The%20compensation%20to%20incentivize%20users%20to%20call%20absorb%20is%20currently%20set%20at%20the%20minimum%20of%203%25%20of%20a%20trove%27s%20collateral%20value%2C%20or%2050%20USD%2C%20whichever%20is%20lower):

  > The compensation to incentivize users to call absorb is currently set at the minimum of 3% of a trove's collateral value, or 50 USD, whichever is lower.

  Where the `COMPENSATION_CAP` is set as a constant with the following value, where it assumes that USD has 18 decimals (Wad):

  ```javascript
  // Cap on compensation value: 50 (Wad)
  const COMPENSATION_CAP: u128 = 50000000000000000000;
  ```

- By knowing that `USD` has [**8 decimals**](https://docs.pragmaoracle.com/Resources/Cairo%201/data-feeds/supported-assets#abstract-currencies), where 1 USD is represented as 1\*1e8, then the assigned `COMPENSATION_CAP` is set to be `500000000000*1e8`, which is **10** times larger than intended.

- This will result in compensating the caller of the `absorb` with 3% of the trove's value in almost all cases (as long as the 3% of trove's value less than `500000000000` USD).

## Proof of Concept

[purger.COMPENSATION_CAP constant](https://github.com/code-423n4/2024-01-opus/blob/4720e9481a4fb20f4ab4140f9cc391a23ede3817/src/core/purger.cairo#L59C5-L60C57)

```javascript
// Cap on compensation value: 50 (Wad)
const COMPENSATION_CAP: u128 = 50000000000000000000;
```

[Check USD decimals](https://docs.pragmaoracle.com/Resources/Cairo%201/data-feeds/supported-assets#abstract-currencies)

## Recommendation

Modify `COMPENSATION_CAP` constant to reflect the actual USD decimals.

## [L-14] Approve race condition in `approve_helper` funtion <a id="l-14" ></a>

## Details

- The `ERC20Helpers.approve_helper` doesn't have any protection against the multiple withdrawal attack on the `approve` and `transferFrom` functions of the ERC20 implementation.

- This race condition can occur when a user calls `approve` function, then calls `transferFrom` on the same token in two separate transactions, and if there's another `transferFrom` transaction between the approve and `transferFrom` calls made by the same spender that its allowance is going to be changed, this will lead to the user's tokens being spent twice by the spender.

## Proof of Concept

[shrine.approve_helper function](https://github.com/code-423n4/2024-01-opus/blob/4720e9481a4fb20f4ab4140f9cc391a23ede3817/src/core/shrine.cairo#L2318C9-L2326C1)

```javascript
fn approve_helper(ref self: ContractState, owner: ContractAddress, spender: ContractAddress, amount: u256) {
            assert(spender.is_non_zero(), 'SH: No approval of 0 address');
            assert(owner.is_non_zero(), 'SH: No approval for 0 address');

            self.yin_allowances.write((owner, spender), amount);

            self.emit(Approval { owner, spender, value: amount });
        }

```

## Recommendation

Add functions to increase/decrease allowance.
