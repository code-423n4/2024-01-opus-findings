## Findings Summary

| Label | Description |
| - | - |
| [L-01](#) |`absorber.provide()` needs to clear `provider_request` to avoid flash loan attacks|
| [L-02](#) |`absorber.convert_to_yin()` may lose precision|
| [L-03](#) |`absorber.transfer_assets()` lacks a check for whether the return value is true|
| [L-04](#) |`flash_loan().eject` does not include `FLASH_FEE`|
| [L-05](#) |`update_prices_internal()` If `force_update == true`, `PriceUpdateMissed` should not occur|
| [L-06](#) |`set_reward()` should restrict the asset from being `ying`|

## [L-01] `absorber.provide()` needs to clear `provider_request` to avoid flash loan attacks
The protocol adds a `request->remove` mechanism to prevent attacks. However, `absorber.provide()` does not clear `request`. This allows malicious users to execute a `flash loan attack` within a single transaction.
1.request()
2. when remove() can execute
   - flash loan
   - execute absorber.provide() Amplify shares
   - trigger equalizer.allocate() to distribute blesser
   - execute absorber.remove() get rewards
   - repay flash loan

suggest:
```diff
        fn provide(ref self: ContractState, amount: Wad) {
            self.assert_live();
+           //force remove request
+           let mut request: Request = self.provider_request.read(provider);
+           request.has_removed = true;
+           self.provider_request.write(provider, request);

```

## [L-02] `absorber.convert_to_yin()` may lose precision
in ` absorber.convert_to_yin() `
```rust
        fn convert_to_yin(self: @ContractState, shares_amt: Wad) -> Wad {
            let total_shares: Wad = self.total_shares.read();

            // If no shares are issued yet, then it is a new epoch and absorber is emptied.
            if total_shares.is_zero() {
                return WadZeroable::zero();
            }

            let absorber: ContractAddress = get_contract_address();
            let yin_balance: Wad = self.yin_erc20().balance_of(absorber).try_into().unwrap();
@>          (shares_amt * yin_balance) / total_shares
        }
```
Using the formula: `(shares_amt * yin_balance) / total_shares = shares_amt.val * yin_balance.val / 1e18 * 1e18 / total_shares.val`. Dividing by `1e18` first can lead to precision loss. 
It is recommended to modify it to not perform `/ 1e18 * 1e18`.


```diff
        fn convert_to_yin(self: @ContractState, shares_amt: Wad) -> Wad {
            let total_shares: Wad = self.total_shares.read();

            // If no shares are issued yet, then it is a new epoch and absorber is emptied.
            if total_shares.is_zero() {
                return WadZeroable::zero();
            }

            let absorber: ContractAddress = get_contract_address();
            let yin_balance: Wad = self.yin_erc20().balance_of(absorber).try_into().unwrap();
-           (shares_amt * yin_balance) / total_shares
+           let   result:u256 = (shares_amt.val * yin_balance.val) / total_shares.val
+           Wad { val:result.try_into().expect('u128')}
        }
```

## [L-03] `absorber.transfer_assets()` lacks a check for whether the return value is true
in `absorber.transfer_assets()`
```rust
        fn transfer_assets(ref self: ContractState, to: ContractAddress, mut asset_balances: Span<AssetBalance>) {
            loop {
                match asset_balances.pop_front() {
                    Option::Some(asset_balance) => {
                        if (*asset_balance.amount).is_non_zero() {
@>                          IERC20Dispatcher { contract_address: *asset_balance.address }
                                .transfer(to, (*asset_balance.amount).into());
                        }
                    },
                    Option::None => { break; },
                };
            };
        }
```

The above method does not check if the return value is `== true`. It is recommended to add this check.
```diff
        fn transfer_assets(ref self: ContractState, to: ContractAddress, mut asset_balances: Span<AssetBalance>) {
            loop {
                match asset_balances.pop_front() {
                    Option::Some(asset_balance) => {
                        if (*asset_balance.amount).is_non_zero() {
-                            IERC20Dispatcher { contract_address: *asset_balance.address }
-                               .transfer(to, (*asset_balance.amount).into());
+                            let success:bool = IERC20Dispatcher { contract_address: *asset_balance.address }
+                              .transfer(to, (*asset_balance.amount).into());
+                           asset(success,"not success");
                        }
                    },
                    Option::None => { break; },
                };
            };
        }
```
## [L-04] `flash_loan().eject` does not include `FLASH_FEE`
Currently, since `FLASH_FEE` defaults to 0, `shrine.eject()` does not include it. It is recommended to add it to avoid errors in the future.

```diff
        fn flash_loan(
            ref self: ContractState,
            receiver: ContractAddress,
            token: ContractAddress,
            amount: u256,
            call_data: Span<felt252>
        ) -> bool {
..
-           shrine.eject(receiver, amount_wad);
+          shrine.eject(receiver, amount_wad + FLASH_FEE.into());

            if adjust_ceiling {
                shrine.set_debt_ceiling(ceiling);
            }

            self.emit(FlashMint { initiator, receiver, token, amount });

            self.reentrancy_guard.end();

            true
        }
```


## [L-05] `update_prices_internal()` If `force_update == true`, `PriceUpdateMissed` should not occur
If the oracle is cancelled, it may cause `force_update == true` to not actually update, `purger.absorb()`->`seer.update_prices()`. Not forcibly updating prices can cause major problems.
```diff
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
+                              assert(force_update == false, 'force_update is true ,can't miss');
                                self.emit(PriceUpdateMissed { yang: *yang });
                                break;
                            }
```

## [L-06] `set_reward()` should restrict the asset from being `ying`
In `absorber.cairo`, `ying` is used as an asset. If the reward is also this, it may be mistakenly taken away as an asset. It is recommended to restrict the reward from being `ying`.
```diff
        fn set_reward(ref self: ContractState, asset: ContractAddress, blesser: ContractAddress, is_active: bool) {
            self.access_control.assert_has_role(absorber_roles::SET_REWARD);

            assert(asset.is_non_zero() && blesser.is_non_zero(), 'ABS: Address cannot be 0');
+           assert(asset != self.shrine.read().contract_address,'not ying'); 

```
