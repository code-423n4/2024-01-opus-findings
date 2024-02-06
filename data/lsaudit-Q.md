
# [1] `TroveClosed` should emit how many Yangs have been withdrawn

**File:** `abbot.cairo`

[File: core/abbot.cairo](https://github.com/code-423n4/2024-01-opus/blob/main/src/core/abbot.cairo#L173)
```cairo
173:             // withdraw each and every Yang belonging to the trove from the system
[...]
187:             self.emit(TroveClosed { trove_id }); 
```
Function `close_trove` closes a trove, repaying its debt in full and withdraws all the Yangs. At line 187, it emits a `TraveClosed` event. This event contains only the id of the closed trove.
Since `close_trove` withdraws all the Yangs, it's a good practice to emit the value of withdrawn Yangs in the event.

# [2] Use constants instead of hard-coding numbers into the contract

**File:** `shrine.cairo`

Using constants improves the code readability - especially when the name of the constant is self-explanatory.

[File: core/shrine.cairo](https://github.com/code-423n4/2024-01-opus/blob/main/src/core/shrine.cairo#L400)
```cairo
400:         // Setting initial rate era to 1
401:         self.rates_latest_era.write(1); 
```

Declare constant, e.g., `INITIAL_ERA_RATE`, instead of hard-coding `1`.


# [3] Remove test-related code from the code-base

**File:** `roles.cairo`

[File: core/roles.cairo](https://github.com/code-423n4/2024-01-opus/blob/main/src/core/roles.cairo#L204)
```cairo
204:     #[cfg(test)] 
205:     #[inline(always)]
206:     fn all_roles() -> u128 {
207:         ADD_YANG
208:             + ADJUST_BUDGET
```

Even though `#[cfg(test)]` annotation tells Cairo to compile and run the test code only when `scarb cairo-test` is run - it's still a good practice to not include it in the production code.
Function `all_roles()` should be removed.

# [4] Use `>=`/`<=` operators in `bound_multiplier()`

**File:** `controller.cairo`

[File: core/controller.cairo](https://github.com/code-423n4/2024-01-opus/blob/main/src/core/controller.cairo#L283)
```cairo
283:     fn bound_multiplier(multiplier: SignedRay) -> SignedRay {
284:         if multiplier > MAX_MULTIPLIER.into() { 
285:             MAX_MULTIPLIER.into()
286:         } else if multiplier < MIN_MULTIPLIER.into() {
287:             MIN_MULTIPLIER.into()
288:         } else {
289:             multiplier
290:         }
291:     }
```

When we will use `>=`/`<=` operators instead of `>`/`<`, we won't be needed to enter last `else` branch when `multiplier == MAX_MULTIPLIER.into()` or `multiplier == MIN_MULTIPLIER.into()`.
Please notice, that in the current code-base, when `multiplier == MAX_MULTIPLIER.into()`, then the first `if` condition won't be fulfilled. The 2nd `else if` won't be fulfilled either - and then - the last `else` will be finally executed.
However, since `multiplier == MAX_MULTIPLIER.into()` we can immediately return `MAX_MULTIPLIER.into()`. The same issue occurs for `MIN_MULTIPLIER.into()`:

```
283:     fn bound_multiplier(multiplier: SignedRay) -> SignedRay {
284:         if multiplier >= MAX_MULTIPLIER.into() { 
285:             MAX_MULTIPLIER.into()
286:         } else if multiplier <= MIN_MULTIPLIER.into() {
287:             MIN_MULTIPLIER.into()
288:         } else {
289:             multiplier
290:         }
291:     }
```

Using `>=`/`<=` operators instead of `>`/`<` will save us from two additional comparisons (entering `else if` and `else` branch). This will increase the code readability and saves execution's costs (gas usage) for the end-user.

# [5] Function `update_rate()` does not emit an event

**File:** `shrine.cairo`

Function `update_rate()` updates the base rates of all yangs, thus it changes the state of the contract. However, it does not emit any event.
It's a good practice to emit events whenever some important settings of the contract are being changed. Our recommendation is to emit event in `update_rate()` function.

[File: core/shrine.cairo](https://github.com/code-423n4/2024-01-opus/blob/main/src/core/shrine.cairo#L652)
```cairo
652:         fn update_rates(ref self: ContractState, yangs: Span<ContractAddress>, new_rates: Span<Ray>) { 
```

# [6] Emit events at the end of the function

**File:** `shrine.cairo`

While most of the functions emit events whenever some state had been changed - the exception of this rule was noticed in `update_yin_spot_price()`. 
Above function emits an event before updating the spot price. While this does not lead to any security implication - it's recommended to stick to one way of emitting events. In other functions, the events are being emitted after updating the contract's values:


[File: core/shrine.cairo](https://github.com/code-423n4/2024-01-opus/blob/main/src/core/shrine.cairo#L793)
```cairo
793:         fn update_yin_spot_price(ref self: ContractState, new_price: Wad) {
794:             self.access_control.assert_has_role(shrine_roles::UPDATE_YIN_SPOT_PRICE);
795:             self.emit(YinPriceUpdated { old_price: self.yin_spot_price.read(), new_price }); 
796:             self.yin_spot_price.write(new_price);
```

Follow the Checks - Effects - Interactions pattern by emitting the events when the contract's state variables have already been changed. Our recommendation is to call `self.yin_spot_price.write(new_price);` before `self.emit(YinPriceUpdated { old_price: self.yin_spot_price.read(), new_price });`.

Please notice, that sometimes (to save gas) - it's better to emit an event before the operation which changes the state. However, this scenario does not occur in this example. Moreover, every other function in the protocol follows CEI pattern for emitting events (function executes some action and then emits an event). For the code readability, our recommendation is to stick to this pattern also in `update_yin_spot_price()`.

# [7] Lack of `amount` value check in `withdraw()` and `deposit()` functions in `shrine.cairo`

**File:**  `shrine.cairo`

[File: core/shrine.cairo](https://github.com/code-423n4/2024-01-opus/blob/main/src/core/shrine.cairo#L812)
```cairo
812:         fn deposit(ref self: ContractState, yang: ContractAddress, trove_id: u64, amount: Wad) {
[...]
821:             let new_total: Wad = self.yang_total.read(yang_id) + amount;  
822:             self.yang_total.write(yang_id, new_total);
823: 
824:             // Update trove balance
825:             let new_trove_balance: Wad = self.deposits.read((yang_id, trove_id)) + amount;
826:             self.deposits.write((yang_id, trove_id), new_trove_balance);
[...]
834:         fn withdraw(ref self: ContractState, yang: ContractAddress, trove_id: u64, amount: Wad) {
[...]
839:             self.withdraw_helper(yang, trove_id, amount);
840:             self.assert_valid_trove_action(trove_id);  
841:         }
```

Whenever `amount` is set to `0`, the state of the contract won't be changed. Depositing or withdrawing `0` does not change the state of the contract. Although unlikely this will introduce problems, it is more consistent to check for 0. Checking non-zero operations (such as withdrawing or depositing) allows to avoid expensive operations. 
Verify if the `amount > 0` whenever calling either `deposit` or `withdraw`.


# [8] Check `forge_amount` value in `open_trove()` in `abbot.cairo`

**Files:** `abbot.cairo`, `shrine.cairo`

According to the source-comment in `abbot.cairo`:

[File: core/abbot.cairo](https://github.com/code-423n4/2024-01-opus/blob/main/src/core/abbot.cairo#L130)
```cairo
130:         // optionally forging Yin in the same operation (if `forge_amount` is 0, no Yin is created)
131:         fn open_trove(
```
`open_trove` can forge Yin whenever `forge_amount > 0`. However, this check is nowhere performed.

[File: core/abbot.cairo](https://github.com/code-423n4/2024-01-opus/blob/main/src/core/abbot.cairo#L156)
```cairo
156:             self.shrine.read().forge(user, new_trove_id, forge_amount, max_forge_fee_pct);
```

Function `open_trove` calls `forge()` from `shrine.cairo`:

[File: core/shrine.cairo](https://github.com/code-423n4/2024-01-opus/blob/main/src/core/shrine.cairo#L853)
```cairo
853:             let forge_fee = amount * forge_fee_pct;
854:             let debt_amount = amount + forge_fee;
[...]
862:             let mut trove: Trove = self.troves.read(trove_id);
863:             trove.debt += debt_amount;
864:             self.troves.write(trove_id, trove);
865: 
866:             self.assert_valid_trove_action(trove_id);
867: 
[...]
1360:         fn forge_helper(ref self: ContractState, user: ContractAddress, amount: Wad) {
1361:             self.yin.write(user, self.yin.read(user) + amount);
1362:             self.total_yin.write(self.total_yin.read() + amount);
```

As demonstrated above, neither `open_trove()` nor `forge()` verifies if the forging amount is non-zero. While forging `0` amount does not change the state of the contract, it's a good idea to perform additional check which will verify that `forge_amount` is indeed greater than `0`. 

Our recommendation is to add this check in both `open_trove` and `forge`.
In function `open_trove` (`abbot.cairo`), call `self.shrine.read().forge(user, new_trove_id, forge_amount, max_forge_fee_pct); ` only when `forge_amount > 0`.
In function `forge` (`shrine.cairo`), do not perform any action when `amount` is not greater than `0`.


# [9] `new_trove_id` in `abbot.cairo` can be declared earlier

**File:** `abbot.cairo`

[File: core/abbot.cairo](https://github.com/code-423n4/2024-01-opus/blob/main/src/core/abbot.cairo#L136)
```cairo
136:             let troves_count: u64 = self.troves_count.read();
137:             self.troves_count.write(troves_count + 1);
138: 
139:             let user = get_caller_address();
140:             let user_troves_count: u64 = self.user_troves_count.read(user);
141:             self.user_troves_count.write(user, user_troves_count + 1);
142: 
143:             let new_trove_id: u64 = troves_count + 1;
144:             self.user_troves.write((user, user_troves_count), new_trove_id);
145:             self.trove_owner.write(new_trove_id, user);
```

Since line `137` sets `troves_count` as `troves_count + 1`, which is basically the same as `new_trove_id`, we can declare this variable earlier in the code-base, to avoid redundant addition.

```cairo
            let troves_count: u64 = self.troves_count.read();
            let new_trove_id: u64 = troves_count + 1;
            self.troves_count.write(new_trove_id);

            let user = get_caller_address();
            let user_troves_count: u64 = self.user_troves_count.read(user);
            self.user_troves_count.write(user, user_troves_count + 1);

            self.user_troves.write((user, user_troves_count), new_trove_id);
            self.trove_owner.write(new_trove_id, user);
```


# [10] Function `unsuspend_yang` can be called even when Yang is not suspended

**File:** `shrine.cairo`

[File: core/shrine.cairo](https://github.com/code-423n4/2024-01-opus/blob/main/src/core/shrine.cairo#L636)
```cairo
636:         fn unsuspend_yang(ref self: ContractState, yang: ContractAddress) {
637:             self.access_control.assert_has_role(shrine_roles::UPDATE_YANG_SUSPENSION);
638: 
639:             assert(
640:                 self.get_yang_suspension_status(yang) != YangSuspensionStatus::Permanent, 'SH: Suspension is permanent' 
641:             );
642: 
643:             self.yang_suspension.write(self.get_valid_yang_id(yang), 0);
644:             self.emit(YangUnsuspended { yang, timestamp: get_block_timestamp() });
645:         }
```
Function `unsuspend_yang()` checks only if status is not `YangSuspensionStatus::Permanent`. However, it misses a check if status is not `YangSuspensionStatus::None`. When the status is `YangSuspensionStatus::None`, we shouldn't be able to call `unsuspend_yang()`, because the Yang is not yet suspended.

It's possible to call `unsuspend_yang()` even when it's not suspended (its status is `YangSuspensionStatus::None`). Calling `unsuspend_yang()` when Yang is not suspended still emits an `YangUnsuspended` event. This might be very misleading to the end-user. Make sure to fully verify the `self.get_yang_suspension_status(yang)` and do not allow to call `unsuspend_yang` when it is not suspended.


# [11] Emit both old and new values

**Files:** `shrine.cairo`, `controller.cairo`

[File: core/shrine.cairo](https://github.com/code-423n4/2024-01-opus/blob/main/src/core/shrine.cairo#L765)
```cairo
765:         fn set_minimum_trove_value(ref self: ContractState, value: Wad) {
766:             self.access_control.assert_has_role(shrine_roles::SET_MINIMUM_TROVE_VALUE);
767: 
768:             self.minimum_trove_value.write(value);
769: 
770:             // Event emission
771:             self.emit(MinimumTroveValueUpdated { value });  
772:         }
773: 
774:         fn set_debt_ceiling(ref self: ContractState, ceiling: Wad) {
775:             self.access_control.assert_has_role(shrine_roles::SET_DEBT_CEILING);
776:             self.debt_ceiling.write(ceiling);
777: 
778:             //Event emission
779:             self.emit(DebtCeilingUpdated { ceiling });
780:         }
```

Above functions update the state of the contract (they set minimum trove value and debt ceiling), however, they emit only new (updated) value. It's a good practice to emit both old (previous) value and the new (updated) one - whenever setting/updating important protocol's parameters.

Similar issues exist in the below instances. Functions emits only the new (updated) values, instead of both old and new ones.

[File: core/controller.cairo](https://github.com/code-423n4/2024-01-opus/blob/main/src/core/controller.cairo#L187)
```cairo
187:         fn set_i_gain(ref self: ContractState, i_gain: Ray) {
188:             self.access_control.assert_has_role(controller_roles::TUNE_CONTROLLER);
[...]
198:             // Reset the integral term if the i_gain is set to zero
199:             if i_gain.is_zero() {
200:                 self.i_term.write(SignedRayZeroable::zero());
201:             }
202: 
203:             self.i_gain.write(i_gain.into());
204:             self.emit(GainUpdated { name: 'i_gain', value: i_gain });  
205:         }
206: 
207: 
208:         fn set_alpha_p(ref self: ContractState, alpha_p: u8) {
209:             self.access_control.assert_has_role(controller_roles::TUNE_CONTROLLER);
210:             assert(alpha_p % 2 == 1, 'CTR: alpha_p must be odd');
211:             self.alpha_p.write(alpha_p);
212:             self.emit(ParameterUpdated { name: 'alpha_p', value: alpha_p });
213:         }
214: 
215: 
216:         fn set_beta_p(ref self: ContractState, beta_p: u8) {
217:             self.access_control.assert_has_role(controller_roles::TUNE_CONTROLLER);
218:             assert(beta_p % 2 == 0, 'CTR: beta_p must be even');
219:             self.beta_p.write(beta_p);
220:             self.emit(ParameterUpdated { name: 'beta_p', value: beta_p });
221:         }
222: 
223: 
224:         fn set_alpha_i(ref self: ContractState, alpha_i: u8) {
225:             self.access_control.assert_has_role(controller_roles::TUNE_CONTROLLER);
226:             assert(alpha_i % 2 == 1, 'CTR: alpha_i must be odd');
227:             self.alpha_i.write(alpha_i);
228:             self.emit(ParameterUpdated { name: 'alpha_i', value: alpha_i });
229:         }
230: 
231: 
232:         fn set_beta_i(ref self: ContractState, beta_i: u8) {
233:             self.access_control.assert_has_role(controller_roles::TUNE_CONTROLLER);
234:             assert(beta_i % 2 == 0, 'CTR: beta_i must be even');
235:             self.beta_i.write(beta_i);
236:             self.emit(ParameterUpdated { name: 'beta_i', value: beta_i });
237:         }
```



# [12] Functions which update the value do not verify if the value was really changed

**Files:** `shrine.cairo`, `controller.cairo`

[File: core/shrine.cairo](https://github.com/code-423n4/2024-01-opus/blob/main/src/core/shrine.cairo#L793)
```cairo
793:         fn update_yin_spot_price(ref self: ContractState, new_price: Wad) {
794:             self.access_control.assert_has_role(shrine_roles::UPDATE_YIN_SPOT_PRICE);
795:             self.emit(YinPriceUpdated { old_price: self.yin_spot_price.read(), new_price }); 
796:             self.yin_spot_price.write(new_price);
797:         }
```

Whenever we update/set a new value of some state variable, it's a good practice to make sure that, that value is being indeed updated. In the current implementation of `update_yin_spot_price` in `shrine.cairo`, even when the `new_price` won't be changed, function will still emit an `YinPriceUpdated` event - which might be misleading to the end user. 
E.g., let's assume that `yin_spot_price =  123`. Calling `update_yin_spot_price(123)` won't update/set any new spot price, but `YinPriceUpdated` will still be emitted.

Our recommendation is to add additional check which will verify that provided values are indeed different that the current ones.

The same issue occurs in multiple of instances:

[File: core/shrine.cairo](https://github.com/code-423n4/2024-01-opus/blob/main/src/core/shrine.cairo#L765)
```cairo
765:         fn set_minimum_trove_value(ref self: ContractState, value: Wad) {
766:             self.access_control.assert_has_role(shrine_roles::SET_MINIMUM_TROVE_VALUE);
767: 
768:             self.minimum_trove_value.write(value);
769: 
770:             // Event emission
771:             self.emit(MinimumTroveValueUpdated { value });  
772:         }
773: 
774:         fn set_debt_ceiling(ref self: ContractState, ceiling: Wad) {
775:             self.access_control.assert_has_role(shrine_roles::SET_DEBT_CEILING);
776:             self.debt_ceiling.write(ceiling);
777: 
778:             //Event emission
779:             self.emit(DebtCeilingUpdated { ceiling });
780:         }
```

[File: core/controller.cairo](https://github.com/code-423n4/2024-01-opus/blob/main/src/core/controller.cairo#L187)
```cairo
187:         fn set_i_gain(ref self: ContractState, i_gain: Ray) {
188:             self.access_control.assert_has_role(controller_roles::TUNE_CONTROLLER);
[...]
198:             // Reset the integral term if the i_gain is set to zero
199:             if i_gain.is_zero() {
200:                 self.i_term.write(SignedRayZeroable::zero());
201:             }
202: 
203:             self.i_gain.write(i_gain.into());
204:             self.emit(GainUpdated { name: 'i_gain', value: i_gain });  
205:         }
206: 
207: 
208:         fn set_alpha_p(ref self: ContractState, alpha_p: u8) {
209:             self.access_control.assert_has_role(controller_roles::TUNE_CONTROLLER);
210:             assert(alpha_p % 2 == 1, 'CTR: alpha_p must be odd');
211:             self.alpha_p.write(alpha_p);
212:             self.emit(ParameterUpdated { name: 'alpha_p', value: alpha_p });
213:         }
214: 
215: 
216:         fn set_beta_p(ref self: ContractState, beta_p: u8) {
217:             self.access_control.assert_has_role(controller_roles::TUNE_CONTROLLER);
218:             assert(beta_p % 2 == 0, 'CTR: beta_p must be even');
219:             self.beta_p.write(beta_p);
220:             self.emit(ParameterUpdated { name: 'beta_p', value: beta_p });
221:         }
222: 
223: 
224:         fn set_alpha_i(ref self: ContractState, alpha_i: u8) {
225:             self.access_control.assert_has_role(controller_roles::TUNE_CONTROLLER);
226:             assert(alpha_i % 2 == 1, 'CTR: alpha_i must be odd');
227:             self.alpha_i.write(alpha_i);
228:             self.emit(ParameterUpdated { name: 'alpha_i', value: alpha_i });
229:         }
230: 
231: 
232:         fn set_beta_i(ref self: ContractState, beta_i: u8) {
233:             self.access_control.assert_has_role(controller_roles::TUNE_CONTROLLER);
234:             assert(beta_i % 2 == 0, 'CTR: beta_i must be even');
235:             self.beta_i.write(beta_i);
236:             self.emit(ParameterUpdated { name: 'beta_i', value: beta_i });
237:         }
```


# [13] Lack of `address(0)` check in constructor

**Files:** `absorber.cairo`, `equalizer.cairo`, `caretaker.cairo`, `gate.cairo`, `purger.cairo`, `controller.cairo`, `abbot.cairo`, `flash_mint.cairo`

`constructor` does not verify if provided address is not `address(0)`. This issue was identified in multiple of instances. Our recommendation is to implement additional check which won't allow to assign `address(0)` as any `ContractAddress`.

[File: core/abbot.cairo](https://github.com/code-423n4/2024-01-opus/blob/main/src/core/abbot.cairo#L79)
```cairo
79:     fn constructor(ref self: ContractState, shrine: ContractAddress, sentinel: ContractAddress) {
80:         self.shrine.write(IShrineDispatcher { contract_address: shrine });
81:         self.sentinel.write(ISentinelDispatcher { contract_address: sentinel });
```

[File: core/absorber.cairo](https://github.com/code-423n4/2024-01-opus/blob/main/src/core/absorber.cairo#L227)
```cairo
227:     fn constructor(
228:         ref self: ContractState, admin: ContractAddress, shrine: ContractAddress, sentinel: ContractAddress,
[...]
232:         self.shrine.write(IShrineDispatcher { contract_address: shrine });  
233:         self.sentinel.write(ISentinelDispatcher { contract_address: sentinel });
```


[File: core/caretaker.cairo](https://github.com/code-423n4/2024-01-opus/blob/main/src/core/caretaker.cairo#L100)
```cairo
100:     fn constructor(
[...]
111:         self.shrine.write(IShrineDispatcher { contract_address: shrine });
112:         self.sentinel.write(ISentinelDispatcher { contract_address: sentinel }); 
113:         self.equalizer.write(IEqualizerDispatcher { contract_address: equalizer });
114:     }
```

[File: core/controller.cairo](https://github.com/code-423n4/2024-01-opus/blob/main/src/core/controller.cairo#L081)
```cairo
081:     fn constructor(
[...]
102:         self.shrine.write(shrine);
[...]
```

[File: core/equalizer.cairo](https://github.com/code-423n4/2024-01-opus/blob/main/src/core/equalizer.cairo#L84)
```cairo
84:     fn constructor(
[...]
89:         self.shrine.write(IShrineDispatcher { contract_address: shrine });
```

[File: core/flash_mint.cairo](https://github.com/code-423n4/2024-01-opus/blob/main/src/core/flash_mint.cairo#L66)
```cairo
66:     fn constructor(ref self: ContractState, shrine: ContractAddress) {
67:         self.shrine.write(IShrineDispatcher { contract_address: shrine });
```

[File: core/gate.cairo](https://github.com/code-423n4/2024-01-opus/blob/main/src/core/gate.cairo#L63)
```cairo
63:     fn constructor(
64:         ref self: ContractState, shrine: ContractAddress, asset: ContractAddress, sentinel: ContractAddress
65:     ) {
66:         self.shrine.write(IShrineDispatcher { contract_address: shrine });
67:         self.asset.write(IERC20Dispatcher { contract_address: asset });  
68:         self.sentinel.write(sentinel);
```

[File: core/purger.cairo](https://github.com/code-423n4/2024-01-opus/blob/main/src/core/purger.cairo#L134)
```cairo
134:     fn constructor(
[...]
144:         self.shrine.write(IShrineDispatcher { contract_address: shrine });
145:         self.sentinel.write(ISentinelDispatcher { contract_address: sentinel });  
146:         self.absorber.write(IAbsorberDispatcher { contract_address: absorber });
147:         self.seer.write(ISeerDispatcher { contract_address: seer });
```

# [14] Incorrect documentation

***File:*** `shrine.cairo`

According to provided [documentation](https://demo-35.gitbook.io/untitled/smart-contracts/shrine-module#emergency-mechanism) - when the Shrine is killed, all user-facing actions are disabled. The documentation lists: `deposit`, `withdraw`, `forge` and `melt`. However, it misses an `inject()` function. 

[File: core/shrine.cairo](https://github.com/code-423n4/2024-01-opus/blob/main/src/core/shrine.cairo#L765)
```cairo
958:        fn inject(ref self: ContractState, receiver: ContractAddress, amount: Wad) {
959:            self.access_control.assert_has_role(shrine_roles::INJECT);
960:            // Prevent any debt creation, including via flash mints, once the Shrine is killed
961:            self.assert_live();
```

As demonstrated above, function `inject()` cannot be called when Shrine is killed (because of the `self.assert_live()` line).

Our recommendation is to update the documentation - please add `inject()` function to the list of functions which cannot be called when the Shrine is killed.

# [15] Incorrect punctuation

According to American style guides, `i.e.` should be followed by commna: `i.e.,`. The comma is missed in the below instances:

```
./core/purger.cairo:218:        // - the trove is not liquidatable (i.e. LTV > threshold).
./core/purger.cairo:431:        //       LTV is not worse off (i.e. penalty == (1 - usable_ltv)/usable_ltv).
./core/caretaker.cairo:247:            // drained (i.e. no shares in current epoch), receives a portion of the minted
./core/seer.cairo:211:                                // fetch a price for yang, i.e. we're missing a price update
./core/shrine.cairo:42:    // suspended permanently, i.e. can't be used in the system ever again.
./core/shrine.cairo:125:        // - If amount is negative, then there is a deficit i.e. `total_yin` > total debt
./core/shrine.cairo:127:        // - If amount is positive, then there is a surplus i.e. total debt > `total_yin`
./core/shrine.cairo:1051:                // This `if` branch handles a corner case where a trove without any yangs deposited (i.e. zero value)
./core/shrine.cairo:1137:        // 1. the trove is healthy i.e. its LTV is equal to or lower than its threshold
./core/shrine.cairo:1148:        // is less than the debt ceiling. Otherwise, if the budget is negative (i.e. there is a deficit),
./core/shrine.cairo:1170:        // yang address has not been added (i.e. yang ID = 0)
./core/shrine.cairo:1669:            // For exceptional redistribution of yangs (i.e. not deposited by any other troves, and
./core/shrine.cairo:1796:                            // `debt_to_distribute_for_yang != raw_debt_to_distribute_for_yang` (i.e. `1 != 0`).
./core/shrine.cairo:1829:                            // balance by the amount deposited in the trove has the effect of rebasing (i.e. appreciating)

```

However, according to British style guids, `i.e.` should not be followed by comma: `i.e.`. The comma was noticed in the below instances:

```
./core/absorber.cairo:3:// Non-Wad/Ray fixed-point values (i.e., values whose number of decimals is something other than 18 or 27)
``` 

Our recommendation is to stick to one style of punctuation in abbreviations.

# [16] Typos

[File: absorber.cairo](https://github.com/code-423n4/2024-01-opus/blob/4720e9481a4fb20f4ab4140f9cc391a23ede3817/src/core/absorber.cairo#L434)
```
 //   frontrunning tactics.
```

should be `front-running`

[File: caretaker.cairo](https://github.com/code-423n4/2024-01-opus/blob/4720e9481a4fb20f4ab4140f9cc391a23ede3817/src/core/caretaker.cairo#L203)
```
            // Note that the total system debt may stil be higher than total yin after this
```

should be `still be`

[File: caretaker.cairo](https://github.com/code-423n4/2024-01-opus/blob/4720e9481a4fb20f4ab4140f9cc391a23ede3817/src/core/caretaker.cairo#L319)
```
        //             After User A reclaims, total system yin decreaes to 900, and the Caretaker's balance of
```

should be `decreases`.