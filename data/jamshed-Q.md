## \[L-01\] All inputs are not validated

While some basic validations are present, such as checking that a trove ID is not zero and ensuring that certain arrays are non-empty, there are areas where additional validation could enhance security. Here are some specific points to consider:

1.  **Function Parameters**:
    
    - In functions like `open_trove`, `deposit`, `withdraw`, `forge`, and `melt`, there is minimal explicit validation of the parameters beyond basic assertions. For example, while it checks that `trove_id` is not zero, it does not validate against potential overflow or ensure that `yang_assets` is valid input. Similarly, in `withdraw`, `forge`, and `melt`, it's assumed that the caller is the trove owner without explicitly verifying it. Depending on the contract's requirements, additional checks such as verifying user permissions or ensuring that input values are within acceptable ranges may be necessary.
2.  **Array Access**:
    
    - In the `get_user_trove_ids` function, while iterating over `user_troves`, it's assumed that the provided `user` exists in `user_troves_count`. If `user` does not exist in `user_troves_count`, the function may not behave as expected. Adding a check to ensure that `user` exists in `user_troves_count` before attempting to access `user_troves` would enhance robustness.

### Inputs to functions should be validated to prevent unexpected behavior.

https://github.com/code-423n4/2024-01-opus/blob/main/src/core/abbot.cairo

## \[L-02\] forced type casting

### Description:Explicit type casting does not revert on overflow/underflow.

### Remediation:Avoid a forced type casting as much as possible and ensure values are in the range of type limit.

there are forced type castings in the [src/types.cairo](https://github.com/code-423n4/2024-01-opus/blob/main/src/types.cairo). Forced type casting is when a value of one data type is converted into another data type explicitly by the programmer, even if there may be potential loss of data or precision. Here are some instances of forced type casting in the code:

1.  In the `TroveStorePacking` implementation:
    
    - `pack` function:
        - `value.charge_from.into()`, `value.last_rate_era.into()`, and `value.debt.into()` are forced type castings from `u64` to `u256`.
    - `unpack` function:
        - `charge_from.try_into().unwrap()`, `last_rate_era.try_into().unwrap()`, and `debt.try_into().unwrap()` are forced type castings from `u256` to `u64`.
2.  In the `YangRedistributionStorePacking` implementation:
    
    - `pack` function:
        - `value.unit_debt.into()` and `capped_error.into()` are forced type castings from `Wad` to `u256`.
    - `unpack` function:
        - `unit_debt.try_into().unwrap()` and `error.try_into().unwrap()` are forced type castings from `u256` to `Wad`.
3.  In the `DistributionInfoStorePacking` implementation:
    
    - `pack` function:
        - `value.asset_amt_per_share.into()` and `capped_error.into()` are forced type castings from `u128` to `u256`.
    - `unpack` function:
        - `asset_amt_per_share.try_into().unwrap()` and `error.try_into().unwrap()` are forced type castings from `u256` to `u128`.
4.  In the `ProvisionStorePacking` implementation:
    
    - `pack` function:
        - `value.epoch.into()` and `value.shares.into()` are forced type castings from `u32` and `Wad` to `u256`.
    - `unpack` function:
        - `epoch.try_into().unwrap()` and `shares.try_into().unwrap()` are forced type castings from `u256` to `u32` and `Wad`.
5.  In the `RequestStorePacking` implementation:
    
    - `pack` function:
        - `value.timestamp.into()`, `value.timelock.into()`, and `value.has_removed.into()` are forced type castings from `u64` and `bool` to `u256`.
    - `unpack` function:
        - `timestamp.try_into().unwrap()` and `timelock.try_into().unwrap()` are forced type castings from `u256` to `u64`.

In each of these cases, the programmer explicitly converts values from one data type to another using the `.into()` and `.try_into().unwrap()` methods, which indicates forced type casting.

## \[L-3\] functions get an index of an array as an argument

### Description: If an array is supposed to be updated (removal in the middle), the indexes will change.

### Remediation: Do not use an index of an array that is supposed to be updated as a parameter of a function.

&nbsp;

there is a function in `shrine.cairo` that takes an index of an array as an argument. The function `update_rates` in the `IShrineImpl` implementation of the `IShrine` contract takes an array of yangs and their updated rates, where the index of each yang in the array corresponds to the index of the new rate in the n`ew_rates` array. This function updates the base rates of all yangs based on the provided indices and new rates.

https://github.com/code-423n4/2024-01-opus/blob/main/src/core/shrine.cairo

&nbsp;

# \[L-4\]  Arithmetic Overflow

The default primitive type, the field element (felt), behaves much like an integer in other languages, but there are a few important differences to keep in mind. A felt can be interpreted as a signed integer in the range (-P/2, P/2) or as an unsigned integer in the range (0, P\]. P represents the prime used by Cairo, which is currently a 252-bit number. Arithmetic operations using felts are unchecked for overflow, which can lead to unexpected results if not properly accounted for. Since the range of values includes both negative and positive values, multiplying two positive numbers can result in a negative value and vice versa—multiplying two negative numbers does not always produce a positive result.

For more robust integer support, consider using SafeUint256 from OpenZeppelin's Contracts for Cairo.

## Mitigations

- Always add checks for overflow when working with felts or Uint256s directly.
- Consider using the [<ins>OpenZeppelin Contracts for Cairo's SafeUint256 functions</ins>](https://github.com/OpenZeppelin/cairo-contracts/blob/main/src/openzeppelin/security/safemath/library.cairo) instead of performing arithmetic directly.

```
const TWO_POW_32: felt252 = 0x100000000;
const TWO_POW_64: felt252 = 0x10000000000000000;
const TWO_POW_122: felt252 = 0x4000000000000000000000000000000;
const TWO_POW_128: felt252 = 0x100000000000000000000000000000000;
const TWO_POW_250: felt252 = 0x400000000000000000000000000000000000000000000000000000000000000;
```

https://github.com/code-423n4/2024-01-opus/blob/4720e9481a4fb20f4ab4140f9cc391a23ede3817/src/types.cairo#L8C1-L13C1

## \[L-03\] Insufficient coverage

Description  
The test coverage rate of the project is ~90%. Testing all functions is best practice in terms of security criteria.

Due to its capacity, test coverage is expected to be 100%.

```
What is the overall line coverage percentage provided by your tests?: 90%
```

&nbsp;

## \[N-01\] Use a single file for all system-wide constants

There are many addresses and constants used in the system. It is recommended to put the most used ones in one file (for example constants.cairo, use inheritance to access these values).

This will help with readability and easier maintenance for future changes. This also helps with any issues, as some of these hard-coded values are admin addresses.

## \[N-02\] Generate perfect code headers every time

Description:  
I recommend using header for Solidity code layout and readability

```
/*//////////////////////////////////////////////////////////////
                           TESTING 123
//////////////////////////////////////////////////////////////*/
```

## \[N-03\] Split Large Functions:

Break down large functions into smaller, more manageable functions for easier understanding.

&nbsp;