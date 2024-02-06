# Low

## [L-01] Loss of precission in `absorber::update_absorber_asset`

Most programming languages like Cairo rounds down when doing integer divisions, so it is recommended to do any operation before the division, in our situation, a multiplication. For example, the possible impacts are from losing a few wei that would not be distributed due to rounding down, to making `asset_amt_per_share` be `0` if `asset_amt_per_share * WAD_ONE < total_recipient_shares.val` and pumping `error` to be equal to `total_amount_to_distribute`, which is a not desired state of the contract:

[**absorber, function update_absorber_asset**](https://github.com/code-423n4/2024-01-opus/blob/4720e9481a4fb20f4ab4140f9cc391a23ede3817/src/core/absorber.cairo#L734C1-L737C1)

```rs
            ...

            let asset_amt_per_share: u128 = u128_wdiv(total_amount_to_distribute, total_recipient_shares.val); // @audit division before multiplication
            let actual_amount_distributed: u128 = u128_wmul(asset_amt_per_share, total_recipient_shares.val);
            let error: u128 = total_amount_to_distribute - actual_amount_distributed;

            ...
```

Reorder it to make the `u128_wmul` before the `u128_wdiv`.

## [L-02] Loss of precission in `absorber::bestow`

The same as above, not gonna repeat myself.

[**absorber, function bestow**](https://github.com/code-423n4/2024-01-opus/blob/4720e9481a4fb20f4ab4140f9cc391a23ede3817/src/core/absorber.cairo#L955C1-L958C1)

```rs
                    ...

                    let total_amount_to_distribute: u128 = blessed_amt + epoch_reward_info.error;

                    let asset_amt_per_share: u128 = u128_wdiv(total_amount_to_distribute, total_recipient_shares.val); // @audit division before multiplication
                    let actual_amount_distributed: u128 = u128_wmul(asset_amt_per_share, total_recipient_shares.val);
                    let error: u128 = total_amount_to_distribute - actual_amount_distributed;

                    let updated_asset_amt_per_share: u128 = epoch_reward_info.asset_amt_per_share + asset_amt_per_share;

                    ...
```

Reorder it to make the `u128_wmul` before the `u128_wdiv`.

## [L-03] Loss of precission in `shrine::scale_threshold_for_recovery_mode`

The same as above, not gonna repeat myself.

[**shrine, function scale_threshold_for_recovery_mode**](https://github.com/code-423n4/2024-01-opus/blob/4720e9481a4fb20f4ab4140f9cc391a23ede3817/src/core/shrine.cairo#L1208)

```rs
        fn scale_threshold_for_recovery_mode(self: @ContractState, mut threshold: Ray) -> Ray {
            let shrine_health: Health = self.get_shrine_health();

            if self.is_recovery_mode_helper(shrine_health) {
                let recovery_mode_threshold: Ray = shrine_health.threshold * RECOVERY_MODE_THRESHOLD_MULTIPLIER.into();
                return max(
                    threshold * THRESHOLD_DECREASE_FACTOR.into() * (recovery_mode_threshold / shrine_health.ltv), // @audit rounding / before *
                    (threshold.val / 2_u128).into()
                );
            }

            threshold
        }
```

Change it to:

```diff
        fn scale_threshold_for_recovery_mode(self: @ContractState, mut threshold: Ray) -> Ray {
            let shrine_health: Health = self.get_shrine_health();

            if self.is_recovery_mode_helper(shrine_health) {
                let recovery_mode_threshold: Ray = shrine_health.threshold * RECOVERY_MODE_THRESHOLD_MULTIPLIER.into();
                return max(
-                   threshold * THRESHOLD_DECREASE_FACTOR.into() * (recovery_mode_threshold / shrine_health.ltv),
+                   (threshold * THRESHOLD_DECREASE_FACTOR.into() * recovery_mode_threshold) / shrine_health.ltv,
                    (threshold.val / 2_u128).into()
                );
            }

            threshold
        }
```

## [L-04] Loss of precision in `shrine::get_yang_threshold_helper`

The same as above, not gonna repeat myself.

[**shrine, function get_yang_threshold_helper**](https://github.com/code-423n4/2024-01-opus/blob/4720e9481a4fb20f4ab4140f9cc391a23ede3817/src/core/shrine.cairo#L1256)

```rs
        fn get_yang_threshold_helper(self: @ContractState, yang_id: u32) -> Ray {
            let base_threshold: Ray = self.thresholds.read(yang_id);
            match self.get_yang_suspension_status_helper(yang_id) {
                YangSuspensionStatus::None => { base_threshold },
                YangSuspensionStatus::Temporary => {
                    // linearly decrease the threshold from base_threshold to 0
                    // based on the time passed since suspension started
                    let ts_diff: u64 = get_block_timestamp() - self.yang_suspension.read(yang_id);
                    base_threshold * ((SUSPENSION_GRACE_PERIOD - ts_diff).into() / SUSPENSION_GRACE_PERIOD.into()) // @audit / before *
                },
                YangSuspensionStatus::Permanent => { RayZeroable::zero() },
            }
```

Change it to:

```diff
        fn get_yang_threshold_helper(self: @ContractState, yang_id: u32) -> Ray {
            let base_threshold: Ray = self.thresholds.read(yang_id);
            match self.get_yang_suspension_status_helper(yang_id) {
                YangSuspensionStatus::None => { base_threshold },
                YangSuspensionStatus::Temporary => {
                    // linearly decrease the threshold from base_threshold to 0
                    // based on the time passed since suspension started
                    let ts_diff: u64 = get_block_timestamp() - self.yang_suspension.read(yang_id);
-                   base_threshold * ((SUSPENSION_GRACE_PERIOD - ts_diff).into() / SUSPENSION_GRACE_PERIOD.into())
+                   (base_threshold * (SUSPENSION_GRACE_PERIOD - ts_diff).into()) / SUSPENSION_GRACE_PERIOD.into()
                },
                YangSuspensionStatus::Permanent => { RayZeroable::zero() },
            }
```

## [L-05] Failing tests

The next tests inside `test_purger.cairo` are failing, which cover a critical part of the project:

1. Remove all the `#[ignore]` inside the repo and run `scarb test`
   1. [FAIL] opus::tests::purger::test_purger::test_purger::test_liquidate_suspended_yang_threshold_near_zero
   2. [FAIL] opus::tests::purger::test_purger::test_purger::test_liquidate_suspended_yang
   3. [FAIL] opus::tests::purger::test_purger::test_purger::test_absorb_low_thresholds

# Non-critical

## [NC-01] Threshold not bounded

In

[**pragma, line 123**](https://github.com/code-423n4/2024-01-opus/blob/4720e9481a4fb20f4ab4140f9cc391a23ede3817/src/external/pragma.cairo#L123)

```rs
    #[constructor]
    fn constructor(
        ref self: ContractState,
        admin: ContractAddress,
        oracle: ContractAddress,
        freshness_threshold: u64,
        sources_threshold: u32
    ) {
        self.access_control.initializer(admin, Option::Some(pragma_roles::default_admin_role()));

        // init storage
        self.oracle.write(IPragmaOracleDispatcher { contract_address: oracle });
        let new_thresholds = PriceValidityThresholds { freshness: freshness_threshold, sources: sources_threshold }; // @audit not bounded
        self.price_validity_thresholds.write(new_thresholds);

        self
            .emit(
                PriceValidityThresholdsUpdated {
                    old_thresholds: PriceValidityThresholds { freshness: 0, sources: 0 }, new_thresholds
                }
            );
    }
```

the newly setted thresholds are not bounded like in

```rs
        fn set_price_validity_thresholds(ref self: ContractState, freshness: u64, sources: u32) {
            self.access_control.assert_has_role(pragma_roles::SET_PRICE_VALIDITY_THRESHOLDS);
            assert(
                LOWER_FRESHNESS_BOUND <= freshness && freshness <= UPPER_FRESHNESS_BOUND, 'PGM: Freshness out of bounds'
            );
            assert(LOWER_SOURCES_BOUND <= sources && sources <= UPPER_SOURCES_BOUND, 'PGM: Sources out of bounds');

            let old_thresholds: PriceValidityThresholds = self.price_validity_thresholds.read();
            let new_thresholds = PriceValidityThresholds { freshness, sources };
            self.price_validity_thresholds.write(new_thresholds);

            self.emit(PriceValidityThresholdsUpdated { old_thresholds, new_thresholds });
        }
```

consider bounding them.

## [NC-02] `FLASH_FEE` should be a variable instead of a constant hard-coded to `0`

[**flash_mint, line 34**](https://github.com/code-423n4/2024-01-opus/blob/4720e9481a4fb20f4ab4140f9cc391a23ede3817/src/core/flash_mint.cairo#L32)

```rs
    const FLASH_FEE: u256 = 0; //  should be a variable instead
```

## [NC-03] Unreachable code in `gate::convert_to_yang_helper`

[**gate, function convert_to_yang_helper**](https://github.com/code-423n4/2024-01-opus/blob/4720e9481a4fb20f4ab4140f9cc391a23ede3817/src/core/gate.cairo#L213C1-L219C21)

```rs
        fn convert_to_yang_helper(self: @ContractState, asset_amt: u128) -> Wad {
            let asset: IERC20Dispatcher = self.asset.read();
            let total_yang: Wad = self.get_total_yang_helper(asset.contract_address);

            if total_yang.is_zero() {
                let decimals: u8 = asset.decimals();
                // Otherwise, scale `asset_amt` up by the difference to match `Wad`
                // precision of yang. If asset is of `Wad` precision, then the same
                // value is returned
                fixed_point_to_wad(asset_amt, decimals)
            } else {
                (asset_amt.into() * total_yang) / get_total_assets_helper(asset).into()
            }
        }
```

As the first branch of the conditional checks for the balance of the gate to be zero, and we have that `sentinel` does make a direct transfer to the `gate` of `INITIAL_DEPOSIT_AMT` in:

```rs
        fn add_yang(
            ref self: ContractState,
            yang: ContractAddress,
            yang_asset_max: u128,
            yang_threshold: Ray,
            yang_price: Wad,
            yang_rate: Ray,
            gate: ContractAddress
        ) {
            
            ...

            // Require an initial deposit when adding a yang to prevent first depositor from front-running
            let yang_erc20 = IERC20Dispatcher { contract_address: yang };
            // scale `asset_amt` up by the difference to match `Wad` precision of yang
            let initial_yang_amt: Wad = fixed_point_to_wad(INITIAL_DEPOSIT_AMT, yang_erc20.decimals());
            let initial_deposit_amt: u256 = INITIAL_DEPOSIT_AMT.into();

            let caller: ContractAddress = get_caller_address();
            let success: bool = yang_erc20.transfer_from(caller, gate.contract_address, initial_deposit_amt);

            ...

        }
```

we have a non-removable amount of yang in the gate, which makes it impossible to trigger:

```rs
            let total_yang: Wad = self.get_total_yang_helper(asset.contract_address);

            if total_yang.is_zero() {
                let decimals: u8 = asset.decimals();
                // Otherwise, scale `asset_amt` up by the difference to match `Wad`
                // precision of yang. If asset is of `Wad` precision, then the same
                // value is returned
                fixed_point_to_wad(asset_amt, decimals)
            } 
```
