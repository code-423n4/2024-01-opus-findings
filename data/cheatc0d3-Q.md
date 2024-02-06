# QA REPORT

## Insufficient Input Validation in deposit and withdraw function

https://github.com/code-423n4/2024-01-opus/blob/4720e9481a4fb20f4ab4140f9cc391a23ede3817/src/core/abbot.cairo#L191

https://github.com/code-423n4/2024-01-opus/blob/4720e9481a4fb20f4ab4140f9cc391a23ede3817/src/core/abbot.cairo#L203

The contract lacks explicit checks for zero addresses in operations involving asset transfers or interactions. This omission can result in assets being irretrievably sent to a zero address, leading to a permanent loss of those assets. Additionally, the contract does not ensure that the `trove_id` parameter falls within an expected range before usage. Utilizing an out-of-range `trove_id` could trigger unexpected behavior or vulnerabilities, potentially compromising contract integrity and user assets.

### Code Snippet

The following code snippets highlight areas where insufficient input validation may pose risks:

```
fn deposit(ref self: ContractState, trove_id: u64, yang_asset: AssetBalance) {
    // Absence of zero address check and `trove_id` range validation
    ...
}

fn withdraw(ref self: ContractState, trove_id: u64, yang_asset: AssetBalance) {
    // Absence of zero address check and `trove_id` range validation
    ...
}
```

### Impact

The absence of input validation for zero addresses and parameter ranges can lead to the irreversible loss of assets and unauthorized manipulation of contract states. These vulnerabilities expose users to financial risks and undermine the contract's security and reliability.

### Mitigation

- Implement explicit checks to ensure that addresses involved in asset transfers and interactions are non-zero.
- Introduce validation logic to confirm that parameters like `trove_id` are within valid ranges before they are used in contract operations.

## Insufficient Input Validation in provide function

The code does not show explicit validation for the `amount` parameter in the `provide` function to ensure that it is a positive number and not excessively large, which could potentially lead to integer overflow issues. 

```
fn provide(ref self: ContractState, amount: Wad) {
    ...
    let success: bool = self.yin_erc20().transfer_from(provider, absorber, amount.into());
    assert(success, 'ABS: Transfer failed');
    ...
}
```

An attacker could exploit the lack of validation to disrupt the contract's economic mechanisms, for example, by providing an extremely large amount of `yin` that does not correspond to a real economic value.

The contract should include checks to validate the `amount` before proceeding with the `provide` operation. Here is an example of how the code could be modified to include such checks:

```
fn provide(ref self: ContractState, amount: Wad) {
    // Ensure that the amount is greater than zero and less than a maximum safe value.
    assert(amount > WadZeroable::zero(), 'ABS: Amount must be positive');
    assert(amount < MAX_SAFE_AMOUNT, 'ABS: Amount is too large');

    ...
    let success: bool = self.yin_erc20().transfer_from(provider, absorber, amount.into());
    assert(success, 'ABS: Transfer failed');
    ...
}
```

In this mitigation code, `WadZeroable::zero()` represents the zero value for the `Wad` type, and `MAX_SAFE_AMOUNT` would be a constant defined elsewhere in the contract representing the maximum safe amount that can be provided without risking overflow. The `assert` statements ensure that the `amount` is within the expected range before proceeding with the transfer.

## equalize Function can be more optimized

https://github.com/code-423n4/2024-01-opus/blob/main/src/core/equalizer.cairo#L127

The `equalize` function is designed to mint surplus debt for the contract by consulting the budget available from the `shrine` contract. It begins by verifying that the budget is positive, which is a basic form of input validation. However, the function then proceeds to convert the budget from a `SignedWad` to a `Wad` and performs several operations based on this data without implementing robust error handling or validation of the data received from the `shrine` contract. This lack of thorough validation and error handling could result in the function executing with incorrect or unexpected data, potentially leading to issues such as incorrect minting of surplus debt or interactions with the `shrine` contract that do not behave as intended.

## Code Snippet

```
fn equalize(ref self: ContractState) -> Wad {
    let shrine: IShrineDispatcher = self.shrine.read();

    let budget: SignedWad = shrine.get_budget();
    if !budget.is_positive() {
        return WadZeroable::zero();
    }

    let minted_surplus: Wad = budget.try_into().unwrap();

    // ... (subsequent operations involving shrine contract) ...
}
```

## Impact

Incorrect minting of surplus debt due to improper handling of the budget data.

## Recommendations

- Enhance the `equalize` function with comprehensive error handling mechanisms to gracefully manage unexpected or erroneous responses from the `shrine` contract.
- Introduce robust validation for the conversion process from `SignedWad` to `Wad` to ensure that only valid data is processed for minting surplus debt.

## Absence of Explicit Input Validation for trove_id 

https://github.com/code-423n4/2024-01-opus/blob/main/src/core/purger.cairo#L266

The function begins by fetching the trove's health using `trove_id` but does not explicitly validate the existence or state of the trove associated with this ID before proceeding with absorption logic.

Before proceeding with any operations, explicitly validate that the `trove_id` corresponds to an existing and valid trove. This could involve checking that the trove is in a state that allows for absorption (e.g., not already absorbed or in a state that would make absorption invalid).

```
let trove_exists = shrine.trove_exists(trove_id);
assert(trove_exists, "PU: Trove does not exist");

let trove_state = shrine.get_trove_state(trove_id);
assert(trove_state.allows_absorption(), "PU: Trove state does not allow absorption");
```

## Lack of Input Validation in set_oracles Function

https://github.com/code-423n4/2024-01-opus/blob/main/src/core/seer.cairo#L135

The set_oracles function updates the list of oracles without validating the uniqueness or validity of the oracle addresses provided. This could potentially lead to duplicate or invalid oracle addresses being stored.

## Recommendation
For set_oracles, ensure that each oracle address is unique and valid (not zero) before updating the list of oracles.

## Error Handling in update_prices_internal Function

https://github.com/code-423n4/2024-01-opus/blob/main/src/core/seer.cairo#L193

## Summary
The update_prices_internal function attempts to fetch prices from oracles and update them. However, there's a comment indicating a future improvement to wrap fetch_price calls in a try-catch block. Currently, if an oracle call fails, it could halt the entire price update process, affecting the contract's reliability.

## Impact
The lack of robust error handling in the update_prices_internal function means that a single failing oracle can disrupt the price update process. This not only affects the contract's ability to provide timely and accurate data but also its overall reliability and trustworthiness.

## Recommendation
Implement the suggested try-catch mechanism around fetch_price calls in the update_prices_internal function. This would allow the contract to skip over failing oracle calls without halting the update process, improving the contract's resilience.
