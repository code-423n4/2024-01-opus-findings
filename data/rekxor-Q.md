# QA Report : Opus Audit Contest | 9th Jan 2024 - 6th Feb 2024 by @rekxor

## Executive summary

## Overview

|              |                                                      |
| :----------- | ---------------------------------------------------- |
| Project Name | Opus                                                 |
| Repository   | [Github](https://github.com/code-423n4/2024-01-opus) |
| Website      | [Opus](https://www.opus.money)                       |
| X(Twitter)   | [Twitter](https://twitter.com/OpusMoney)             |
| Methods      | Manual Review                                        |
| Total nSLOC  | 4119 over 13 contracts                               |

---

## Findings Summary

| ID              | Title                                                                                                | Severity       |
| --------------- | ---------------------------------------------------------------------------------------------------- | -------------- |
| [L - 01](#L-01) | Reentrancy Guard missing for `allocate()` - yin transfer to recipients.                              | _Low_          |
| [L - 02](#L-02) | Console Account Creation Without Policies Leads to Enforcement Challenges                            | _Low_          |
| [L - 03](#L-03) | transmuter.cairo :: `settle()` function misses reentrancy guards on the calls made                   | _Low_          |
| [NC-01](#NC-01) | Refactor `get_trove_asset_balance` for Improved Code Readability and Functionality                   | _Non Critical_ |
| [NC-02](#NC-02) | Important operations are missing event emission.                                                     | _Non Critical_ |
| [NC-03](#NC-03) | `Transfer_asstets()` performs important operation, better to have Event Emission.                    | _Non Critical_ |
| [NC-04](#NC-04) | abbot.cairo :: event Please define the `struct` before defining the `event` for enhanced readability | _Non Critical_ |

---

---

## [L-01] Reentrancy Guard missing for equalizer.cairo:: `allocate()` - yin transfer to recipients

[Code Snippet](https://github.com/code-423n4/2024-01-opus/blob/main/src/core/equalizer.cairo#L170-#L200)

```rust
loop {
    match recipients_copy.pop_front() {
    Option::Some(recipient) => {
        let amount: Wad = wadray::rmul_wr(balance, *(percentages_copy.pop_front().unwrap()));
        //below is the transfer function made to IERC20-yin
        yin.transfer(*recipient, amount.into());
        amount_allocated += amount;
    },
    Option::None => { break; }
    };
};
```

#### Recommendation:

Add the `self.reentrancy_guard.start();` before the logic code and end the function with `self.reentrancy_guard.end();`, also following the CEI pattern mitigates all potential reentrancy(s). Updating `amount_allocated` before making interaction will be better.

##

## [L-02] flash_mint.cairo :: `flash_loan()` doesn't perform any access control checks

[Code location](https://github.com/code-423n4/2024-01-opus/blob/main/src/core/flash_mint.cairo#L100C9-L152C10):

```rust
fn flash_loan(
    ref self: ContractState,
    receiver: ContractAddress,
    token: ContractAddress,
    amount: u256,
    call_data: Span<felt252>
        ) -> bool {
    self.reentrancy_guard.start();
    assert(amount <= self.max_flash_loan(token), 'FM: amount exceeds maximum');

    let shrine = self.shrine.read();
    let amount_wad: Wad = amount.try_into().unwrap();
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

#### Recommendation:

Since anyone can call flashloan() with any random borrower as target with random data, we must ensure data passed is not malicious. Perform check on flash loan receiver contract, it should only allow restricted set of initiators.

##

## [L-03] transmuter.cairo :: `settle()` function misses reentrancy guards on the calls made

[Code Location](https://github.com/code-423n4/2024-01-opus/blob/main/src/core/transmuter.cairo#L417C9-L453C1):

```rust
// code
let receiver: ContractAddress = self.receiver.read();
yin.transfer(receiver, (yin_amt - settle_amt).into());

let asset: IERC20Dispatcher = self.asset.read();
asset.transfer(receiver, asset.balance_of(transmuter));
```

#### Recommendation:

Ensure `Check Effects Interaction(CEI)` pattern is followed and add reentrancy guard to the located function for best practices against mitigating potential risks in our code base.

##

## [NC-01] Refactor `get_trove_asset_balance` for Improved Code Readability and Functionality

 [Code Snippet](https://github.com/code-423n4/2024-01-opus/blob/4720e9481a4fb20f4ab4140f9cc391a23ede3817/src/core/abbot.cairo#L121-#L122):

```rust
fn get_trove_asset_balance(self: @ContractState, trove_id: u64, yang: ContractAddress) -> u128 {
	 self.sentinel.read().convert_to_assets(yang, self.shrine.read().get_deposit(yang, trove_id))
}
```

#### Recommendation:

```rust
fn get_trove_asset_balance(
    self: @ContractState,
    trove_id: u64,
    yang: ContractAddress) -> u128
    {
    let shrine_read = self.shrine.read();
    let sentinel_read = self.sentinel.read();
    let deposit = shrine_read.get_deposit(yang, trove_id);
    sentinel_read.convert_to_assets(yang, deposit)
}
```

## [NC-02] Important operations are missing event emission.

[Code Location](https://github.com/code-423n4/2024-01-opus/blob/4720e9481a4fb20f4ab4140f9cc391a23ede3817/src/core/abbot.cairo#L215-#L225):

The above linked code defines `forge()` and `melt()` that **creates** and **destroy** Yin.

#### Recommendation:

Adding events in the `enum` and emitting them in our desired functions.

```diff
#[event]
enum Event {
        TroveOpened: TroveOpened,
        TroveClosed: TroveClosed,
+       Melted: Melted
+       Forged: Forged
        ReentrancyGuardEvent: reentrancy_guard_component::Event
}
+ struct Melted {
+	#[key]
+	trove_id: u64
+ }
//can include any other variable like amount: Wad
+}
+ struct Forged {
+	#[key]
+	trove_id: u64
+ }
```

```diff
fn forge(ref self: ContractState, trove_id: u64, amount: Wad, max_forge_fee_pct: Wad) {
     let user = get_caller_address();
            self.assert_trove_owner(user, trove_id);
     self.shrine.read().forge(user, trove_id, amount, max_forge_fee_pct);
+    self.emit(Forged { trove_id });
 }

 // destroy Yin from a trove
fn melt(ref self: ContractState, trove_id: u64, amount: Wad) {
   // note that caller does not need to be the trove's owner to melt
      self.shrine.read().melt(get_caller_address(), trove_id, amount);
+     self.emit(Melted { trove_id });
}
```

## [NC-03] absorber.cairo:: `Transfer_asstets()` performs important operation, better to have an Event Emission.

[Code location](https://github.com/code-423n4/2024-01-opus/blob/4720e9481a4fb20f4ab4140f9cc391a23ede3817/src/core/absorber.cairo#L876):

```rust
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

##### Issue: 
As mentioned in the above [NC-02]

#### Recommendation:

Adding events in the `enum` and emitting them in our respective function.

## [NC-04] abbot.cairo:: `event` Please define the struct before defining the event for enhanced readability

[Code location](https://github.com/code-423n4/2024-01-opus/blob/main/src/core/abbot.cairo#L53-#L72):

```rust
	#[event]
    #[derive(Copy, Drop, starknet::Event, PartialEq)]
    enum Event {
        TroveOpened: TroveOpened,
        TroveClosed: TroveClosed,
        // Component events
        ReentrancyGuardEvent: reentrancy_guard_component::Event
    }

    #[derive(Copy, Drop, starknet::Event, PartialEq)]
    struct TroveOpened {
        #[key]
        user: ContractAddress,
        #[key]
        trove_id: u64
    }

    #[derive(Copy, Drop, starknet::Event, PartialEq)]
    struct TroveClosed {
        #[key]
        trove_id: u64
    }
```

#### _Issue:_

The code snippet defines the Event well before then defining the structs used in the events. Though `cairo` programming language won't raise any alert/warning but for understanding the codebase it will be helpful for fellow researchers.

#### Recommendation:

Just by placing the structs that we use in our event above the `enum event`, it does the job effortlessly.

```rust
    #[derive(Copy, Drop, starknet::Event, PartialEq)]
    struct TroveOpened {
    //#[key]:it plays a significant role in indexing or uniquely identifying instances of this structure.
        #[key]
        user: ContractAddress,
        #[key]
        trove_id: u64
    }

    #[derive(Copy, Drop, starknet::Event, PartialEq)]
    struct TroveClosed {
        #[key]
        trove_id: u64
    }
     enum Event {
        TroveOpened: TroveOpened,
        TroveClosed: TroveClosed,
        // Component events
        ReentrancyGuardEvent: reentrancy_guard_component::Event
     }


```
