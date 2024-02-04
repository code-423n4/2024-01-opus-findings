# Summary
* L-01: Abbot allows to open empty troves inflating the `troves_count`
* L-02: Inability to remove suspended Yangs and zero prices can cause the protocol to freeze
* L-03: Inactive failing reward assets can cause DoS for main Absorber methods
* L-04: Users currently need to approve tokens to the Gates instead of the Abbot
* L-05: Unresolved ToDo about failing oracles
* L-06: Allowing collateral thresholds up to 100% is unhealthy for the protocol
* L-07: Flash loan interface deviates from ERC-3156 specification
* L-08: Absorber removal request timelock leads to unnecessary disadvantage for users
---
* NC-01: Take flash loan fee
* NC-02: Scarb version possibly incompatible with Starknet
* NC-03: Absorption threshold LTV should be set to 89%
* NC-04: Explicitly handle zero value token tranfers
* NC-05: Mistake in documentation - threshold vs. LTV
* NC-06: Mistake in inline comment - minimum vs. maximum

---

## L-01: Abbot allows to open empty troves inflating the `troves_count`

The [abbot::open_trove(...)](https://github.com/code-423n4/2024-01-opus/blob/4720e9481a4fb20f4ab4140f9cc391a23ede3817/src/core/abbot.cairo#L131-L161) method allows to open **empty** troves (with 0 collateral).  

**Impacts:**
* Unnecessary increase of [`troves_count: u64`](https://github.com/code-423n4/2024-01-opus/blob/4720e9481a4fb20f4ab4140f9cc391a23ede3817/src/core/abbot.cairo#L137) (very unlikely: DoS due to `u64_add Overflow`)
* Unnecessary network stress since no funds (except fees) are required

**PoC:**

The test case below demonstrates that an **empty** trove can be opened.  
Add the test case to `src/tests/abbot/test_abbot.cairo` and run it with `snforge test test_open_empty_trove_pass`.

```cairo
#[test]
fn test_open_empty_trove_pass() {
    let (shrine, _, abbot, yangs, gates) = abbot_utils::abbot_deploy(
        Option::None, Option::None, Option::None, Option::None, Option::None
    );

    let mut spy = spy_events(SpyOn::One(abbot.contract_address));

    // Deploying the first trove
    let trove_owner: ContractAddress = common::trove1_owner_addr();
    let forge_amt: Wad = 0_u128.into();
    //common::fund_user(trove_owner, yangs, abbot_utils::initial_asset_amts());
    let asset_amts: Array<u128> = array![0_u128, 0_u128];
    let deposited_amts: Span<u128> = asset_amts.span();
    let trove_id: u64 = common::open_trove_helper(abbot, trove_owner, yangs, deposited_amts, gates, forge_amt);

    // Check trove ID
    let expected_trove_id: u64 = 1;
    assert(trove_id == expected_trove_id, 'wrong trove ID');
    assert(
        abbot.get_trove_owner(expected_trove_id).expect('should not be zero') == trove_owner, 'wrong trove owner'
    );
    assert(abbot.get_troves_count() == expected_trove_id, 'wrong troves count');

    let mut expected_user_trove_ids: Array<u64> = array![expected_trove_id];
    assert(abbot.get_user_trove_ids(trove_owner) == expected_user_trove_ids.span(), 'wrong user trove ids');
}
```

**Recommendation:**  
Require a **minumum** deposit value when opening a trove.

## L-02: Inability to remove suspended Yangs and zero prices can cause the protocol to freeze

The protocol only provides functionality to [add yangs](https://github.com/code-423n4/2024-01-opus/blob/4720e9481a4fb20f4ab4140f9cc391a23ede3817/src/core/sentinel.cairo#L174-L214), but there is **no** corresponding method for removal.  
Furthermore, the protocol allows to [suspend yangs](https://github.com/code-423n4/2024-01-opus/blob/4720e9481a4fb20f4ab4140f9cc391a23ede3817/src/core/sentinel.cairo#L236-L239), but even if a yang is [permanently  suspened](https://github.com/code-423n4/2024-01-opus/blob/4720e9481a4fb20f4ab4140f9cc391a23ede3817/src/types.cairo#L18) it **cannot** be removed.

However, [all yangs ever added](https://github.com/code-423n4/2024-01-opus/blob/4720e9481a4fb20f4ab4140f9cc391a23ede3817/src/core/sentinel.cairo#L122-L133) (not matter if suspended) are [iterated by the `seer`](https://github.com/code-423n4/2024-01-opus/blob/4720e9481a4fb20f4ab4140f9cc391a23ede3817/src/core/seer.cairo#L202-L235) in order to query the orcale(s) for the underlying assets' prices.  

*Consider the following scenario:*  
A collateral asset is hacked and/or there is a scandal about the project team (for example: Terra Luna). Therefore, the asset crashes. It gets **delisted** from major exchanges and **suspended** from Opus.

**Impacts:**
* At some point, the orcale(s) might be unable to provide a price feed for the asset and therefore revert/panic.  
  This causes DoS for **all** price updates of the protocol, consequently freezing the whole protocol.
* There might still be a price feed, but the price is `0` (due to lack of data sources or the price is actually zero).   
  However, the [`shrine` reverts due to an assertion](https://github.com/code-423n4/2024-01-opus/blob/4720e9481a4fb20f4ab4140f9cc391a23ede3817/src/core/shrine.cairo#L726) in case of [`0` price provided by the `seer`](https://github.com/code-423n4/2024-01-opus/blob/4720e9481a4fb20f4ab4140f9cc391a23ede3817/src/core/seer.cairo#L224).  
  This causes DoS for **all** price updates of the protocol, consequently freezing the whole protocol.

**Recommendation:**   
Allow the protocol's admin to remove yangs or at least exclude them from price updates.  
Furthmore, the [`0` price assertion](https://github.com/code-423n4/2024-01-opus/blob/4720e9481a4fb20f4ab4140f9cc391a23ede3817/src/core/shrine.cairo#L726) is dangerous, even during normal operation. Zero prices should be handled in a graceful way, especially in case of suspended assets, in order to avoid freezing the whole protocol.


## L-03: Inactive failing reward assets can cause DoS for main Absorber methods

According to the [documentation](https://demo-35.gitbook.io/untitled/smart-contracts/absorber-module#rewards):
> The Absorber supports distribution of whitelisted rewards. The only requirement is that the vesting contracts adhere to the `Blesser` interface. **Caution should be exercised** (e.g. checking for non-standard behavior like blacklists) when whitelisting a reward token to ensure that a failure to vest reward tokens does not cause an absorption to revert. 

For this purpose, the [absorber::set_reward(asset, blesser, is_active)](https://github.com/code-423n4/2024-01-opus/blob/4720e9481a4fb20f4ab4140f9cc391a23ede3817/src/core/absorber.cairo#L365-L389) method has an `is_active` flag to deactivate a reward `asset`.  
This flag is also [correctly taken into account](https://github.com/code-423n4/2024-01-opus/blob/4720e9481a4fb20f4ab4140f9cc391a23ede3817/src/core/absorber.cairo#L939-L944) in [absorber::bestow()](https://github.com/code-423n4/2024-01-opus/blob/4720e9481a4fb20f4ab4140f9cc391a23ede3817/src/core/absorber.cairo#L914-L975) which is called by [absorber::update(...)](https://github.com/code-423n4/2024-01-opus/blob/4720e9481a4fb20f4ab4140f9cc391a23ede3817/src/core/absorber.cairo#L545-L625) on absorption in order to avoid permanent absorption DoS on failing reward tokens.  

However, the `is_active` flag is [**not** taken in account](https://github.com/code-423n4/2024-01-opus/blob/4720e9481a4fb20f4ab4140f9cc391a23ede3817/src/core/absorber.cairo#L999) in [absorber::get_provider_rewards(...)](https://github.com/code-423n4/2024-01-opus/blob/4720e9481a4fb20f4ab4140f9cc391a23ede3817/src/core/absorber.cairo#L980-L1046) called by [absorber::get_absorbed_and_rewarded_assets_for_provider()](https://github.com/code-423n4/2024-01-opus/blob/4720e9481a4fb20f4ab4140f9cc391a23ede3817/src/core/absorber.cairo#L768-L776) which is further called by [absorber::reap_helper(...)](https://github.com/code-423n4/2024-01-opus/blob/4720e9481a4fb20f4ab4140f9cc391a23ede3817/src/core/absorber.cairo#L780-L813) on [absorber::provide(...)](https://github.com/code-423n4/2024-01-opus/blob/4720e9481a4fb20f4ab4140f9cc391a23ede3817/src/core/absorber.cairo#L397-L429), [absorber::remove(...)](https://github.com/code-423n4/2024-01-opus/blob/4720e9481a4fb20f4ab4140f9cc391a23ede3817/src/core/absorber.cairo#L463-L525) and [absorber::reap(...)](https://github.com/code-423n4/2024-01-opus/blob/4720e9481a4fb20f4ab4140f9cc391a23ede3817/src/core/absorber.cairo#L529-L542).

**Impacts:**  
The methods [absorber::provide(...)](https://github.com/code-423n4/2024-01-opus/blob/4720e9481a4fb20f4ab4140f9cc391a23ede3817/src/core/absorber.cairo#L397-L429), [absorber::remove(...)](https://github.com/code-423n4/2024-01-opus/blob/4720e9481a4fb20f4ab4140f9cc391a23ede3817/src/core/absorber.cairo#L463-L525) and [absorber::reap(...)](https://github.com/code-423n4/2024-01-opus/blob/4720e9481a4fb20f4ab4140f9cc391a23ede3817/src/core/absorber.cairo#L529-L542) are subject to DoS in case a single reward asset becomes faulty (reverts on [transfer](https://github.com/code-423n4/2024-01-opus/blob/4720e9481a4fb20f4ab4140f9cc391a23ede3817/src/core/absorber.cairo#L794)). Deactivating the asset via `is_active = false` **cannot** avoid this issue in the current implementation.

**Recommendation:**  
Skip inactive reward assets within [absorber::get_provider_rewards(...)](https://github.com/code-423n4/2024-01-opus/blob/4720e9481a4fb20f4ab4140f9cc391a23ede3817/src/core/absorber.cairo#L980-L1046):

```diff
diff --git a/src/core/absorber.cairo b/src/core/absorber.cairo
index 473275f..710075f 100644
--- a/src/core/absorber.cairo
+++ b/src/core/absorber.cairo
@@ -997,6 +997,11 @@ mod absorber {
                 }
 
                 let reward: Reward = self.rewards.read(current_rewards_id);
+                if !reward.is_active {
+                    current_rewards_id += 1;
+                    continue;
+                }
+
                 let mut reward_amt: u128 = 0;
                 let mut epoch: u32 = provision.epoch;
                 let mut epoch_shares: Wad = provision.shares;

```

## L-04: Users currently need to approve tokens to the Gates instead of the Abbot

According to the [documentation](https://demo-35.gitbook.io/untitled/smart-contracts/gate-module):
> As the Gate is an internal-facing module, users will not be able to, and are **not expected to**, interact with the Gate directly.

Users are expected to handle their troves using the `abbot` module which indirectly interacts with the `gate` modules for each asset via the `sentinel` module:
![Image: Depositing collateral
](https://4180423341-files.gitbook.io/~/files/v0/b/gitbook-x-prod.appspot.com/o/spaces%2FBTWxg1bdHQ15qxTg6SOE%2Fuploads%2FXKfG3RpuZYyVh05LWVTB%2Fimage.png?alt=media&token=9fa74deb-031d-430a-930b-d622ed6a89c2)

However, a user still needs to know the address of a respective `gate` and give **token approval** to the `gate` instead of the `abbot` module, because the `gate` [pulls the assets from the user](https://github.com/code-423n4/2024-01-opus/blob/4720e9481a4fb20f4ab4140f9cc391a23ede3817/src/core/gate.cairo#L137-L150) on deposit:
```cairo
fn enter(ref self: ContractState, user: ContractAddress, trove_id: u64, asset_amt: u128) -> Wad {
    ...

    let success: bool = self.asset.read().transfer_from(user, get_contract_address(), asset_amt.into());
    assert(success, 'GA: Asset transfer failed');

    ...
}
```

**Impacts:**  
Complicated UX that partially counters the documentation.

**Recommendation**:  
The user should only have to give token approvals to the `abbot` module which pulls the assets from the user and subsequently forwards them to the respective `gate`.

## L-05: Unresolved ToDo about failing oracles
The following [TODO](https://github.com/code-423n4/2024-01-opus/blob/4720e9481a4fb20f4ab4140f9cc391a23ede3817/src/core/seer.cairo#L216-L218) is still unresolved:
```cairo
// TODO: when possible in Cairo, fetch_price should be wrapped
//       in a try-catch block so that an exception does not
//       prevent all other price updates
```

**Impacts**:  
It's entirely possible that an oracle pice data query fails/reverts/panics with an exception which causes the whole price update transaction to fail. This causes DoS for **all** price updates of the protocol, consequently freezing the whole protocol.

**Recommendation:**  
Make sure an oracle's mehods are declared with the [nopanic notation](https://docs.cairo-lang.org/language_constructs/panic.html#nopanic_notation), otherwise handle potential errors gracefully with the *match syntax* instead of [directly expecting return data](https://github.com/code-423n4/2024-01-opus/blob/4720e9481a4fb20f4ab4140f9cc391a23ede3817/src/external/pragma.cairo#L190).

## L-06: Allowing collateral thresholds up to 100% is unhealthy for the protocol

The protocol currently allows thresholds up to 100% (see [MAX_THRESHOLD](https://github.com/code-423n4/2024-01-opus/blob/4720e9481a4fb20f4ab4140f9cc391a23ede3817/src/core/shrine.cairo#L37)) for collateral assets. This is unhealthy because liquidations/absorptions can only happen if `ltv > threshold`. Furthermore, the liquidation penalty at `ltv >= 100%` is zero, i.e. there is no liquidation incentive and consequently troves with such high thresholds can effectively only be absorbed.  

**Recommendation:** Set the `MAX_THRESHOLD` to 90% in order to facilitate profitable searcher liquidations in any case.

## L-07: Flash loan interface deviates from ERC-3156 specification
The current implementation of the [flash_mint](https://github.com/code-423n4/2024-01-opus/blob/4720e9481a4fb20f4ab4140f9cc391a23ede3817/src/core/flash_mint.cairo) module only has methods in *snake_case*. However, the [ERC-3156](https://eips.ethereum.org/EIPS/eip-3156) specification requires the lenders's and receiver's methods in *camelCase* which could lead to severe interoperability problems.  

**Recommendation:** Additonally add the *camelCase* implementation which forwards to the original methods. For reference, see the [Shrine's Yin ERC-20 implementation](https://github.com/code-423n4/2024-01-opus/blob/4720e9481a4fb20f4ab4140f9cc391a23ede3817/src/core/shrine.cairo#L2281-L2296) where *camelCase* support was already implemented.

## L-08: Absorber removal request timelock leads to unnecessary disadvantage for users

If a user [requests](https://github.com/code-423n4/2024-01-opus/blob/4720e9481a4fb20f4ab4140f9cc391a23ede3817/src/core/absorber.cairo#L441-L465) to [remove](https://github.com/code-423n4/2024-01-opus/blob/4720e9481a4fb20f4ab4140f9cc391a23ede3817/src/core/absorber.cairo#L468-L530) yin from the Absorber **whithin** the `REQUEST_COOLDOWN` period, the `timelock` for removal multiplies.  
This makes sense in case the user immediately requests again **after** a yin removal. However, the `timelock` is also multiplied in case the user accidentally requests again **before** a removal, which is simply unfair.

**Recommendation:**  
Only increase the `timelock` on request within `REQUEST_COOLDOWN` **after** a removal:

```diff
diff --git a/src/core/absorber.cairo b/src/core/absorber.cairo
index 473275f..ab4ba07 100644
--- a/src/core/absorber.cairo
+++ b/src/core/absorber.cairo
@@ -447,7 +447,12 @@ mod absorber {
 
             let mut timelock: u64 = REQUEST_BASE_TIMELOCK;
             if request.timestamp + REQUEST_COOLDOWN > current_timestamp {
-                timelock = request.timelock * REQUEST_TIMELOCK_MULTIPLIER;
+                if request.has_removed {
+                    timelock = request.timelock * REQUEST_TIMELOCK_MULTIPLIER;
+                }
+                else {
+                    timelock = request.timelock;
+                }
             }
 
             let capped_timelock: u64 = min(timelock, REQUEST_MAX_TIMELOCK);

```

---

## NC-01: Take flash loan fee

Although the fee for yin flash loans is currently zero ([`FLASH_FEE = 0`](https://github.com/code-423n4/2024-01-opus/blob/4720e9481a4fb20f4ab4140f9cc391a23ede3817/src/core/flash_mint.cairo#L32)), it should still be accounted for when [withdrawing the loan `amount` from the borrower](https://github.com/code-423n4/2024-01-opus/blob/4720e9481a4fb20f4ab4140f9cc391a23ede3817/src/core/flash_mint.cairo#L141), just to make sure this step is not forgotten once the flash fee becomes non-zero in the future. 

```diff
diff --git a/src/core/flash_mint.cairo b/src/core/flash_mint.cairo
index d2114d8..97c21d9 100644
--- a/src/core/flash_mint.cairo
+++ b/src/core/flash_mint.cairo
@@ -138,7 +138,7 @@ mod flash_mint {
             assert(borrower_resp == ON_FLASH_MINT_SUCCESS, 'FM: on_flash_loan failed');
 
             // This function in Shrine takes care of balance validation
-            shrine.eject(receiver, amount_wad);
+            shrine.eject(receiver, amount_wad + FLASH_FEE.try_into().unwrap());
 
             if adjust_ceiling {
                 shrine.set_debt_ceiling(ceiling);

```

## NC-02: Scarb version possibly incompatible with Starknet

When compiling the protocol's modules via Scarb v2.4.0 as suggested by the `README`, the following warning arises:
```
This version is not yet supported on Starknet! If you want to develop contracts deployable to current Starknet, please stick with Scarb v2.3.1
```
This info might already be obsolete but definitely deserves a second look to be sure.

## NC-03: Absorption threshold LTV should be set to 89%

According to the protocol's [graph for liquidation penalty](https://demo-35.gitbook.io/untitled/smart-contracts/purger-module#resources), it becomes evident that the `MAX_PENALTY` can only be achieved for `ltv` up to `89%`. For higher `ltv` the penalty sinks again towards zero at `ltv = 100%`. Therefore, the searcher liquidation incentives are getting lower for `ltv >= 89%`.  
As a consequence, it would be more ideal to set the [ABSORPTION_THRESHOLD](https://github.com/code-423n4/2024-01-opus/blob/4720e9481a4fb20f4ab4140f9cc391a23ede3817/src/core/purger.cairo#L53) to `89%` instead of currently `90%`.

## NC-04: Explicitly handle zero value token tranfers

According to Starkscan, the desired collateral tokens [WBTC](https://starkscan.co/token/0x03fe2b97c1fd336e750087d68b9b867997fd64a2661ff3ca5a7c771641e8e7ac#read-write-contract-sub-write),  [ETH](https://starkscan.co/token/0x049d36570d4e46f48e99674bd3fcc84644ddd6b96f7c741b1562b82f9e004dc7#read-write-contract-sub-write) and [wstETH](https://starkscan.co/token/0x042b8f0484674ca266ac5d08e4ac6a3fe65bd3129795def2dca5c34ecc5f96d2#read-write-contract-sub-write) (mainnet addresses from [Starknet documentation](https://docs.starknet.io/documentation/tools/starkgate-bridge/#starkgate_supported_tokens)) do **not** panic on `0` value transfers which is a good thing.  
However, there might be other collateral tokens added in the future which show [such behaviour](https://github.com/d-xo/weird-erc20?tab=readme-ov-file#revert-on-zero-value-transfers). Therefore, explicitly check for and skip `0` value token transfers throughout the protocol.

## NC-05: Mistake in documentation - threshold vs. LTV

According to the [liquidation penalty documentation](https://demo-35.gitbook.io/untitled/smart-contracts/purger-module#liquidation-penalty):
> `s` is a scalar that is introduced to control how quickly the absorption penalty reaches the maximum possible penalty for ~thresholds~ **LTV** at or greater than 90%. This lets the protocol control how much to incentivize users to call `absorb` depending on how quick and/or desirable absorptions are for the protocol.

For reference, see [key functions documentation](https://demo-35.gitbook.io/untitled/smart-contracts/purger-module#description-of-key-functions):
> Absorption can happen only after an unhealthy trove's LTV has exceeded the LTV at which the maximum possible penalty is reached, **or if it has exceeded 90% LTV**. The liquidation penalty in this case will similarly be capped to the maximum of 12.5% or the maximum possible penalty.

## NC-06: Mistake in inline comment - minimum vs. maximum

The following [inline comment](https://github.com/code-423n4/2024-01-opus/blob/4720e9481a4fb20f4ab4140f9cc391a23ede3817/src/core/purger.cairo#L559-L561) is wrong because it contradicts the context
> // If the threshold is below the given minimum, we automatically  
> // return the ~minimum~ **maximum** penalty to avoid division by zero/overflow, or the largest possible penalty,  
> // whichever is smaller.  

and is not in line with the subsequent [code](https://github.com/code-423n4/2024-01-opus/blob/4720e9481a4fb20f4ab4140f9cc391a23ede3817/src/core/purger.cairo#L565-L567):
```cairo
if ltv >= MIN_THRESHOLD_FOR_PENALTY_CALCS.into() {
    return Option::Some(min(MAX_PENALTY.into(), (RAY_ONE.into() - ltv) / ltv));
}
```

