# Summary

| Id | Title |
| --- | --- |
| L-1 | Return values of token transfer are unchecked | 
| L-2 | `MAX_NATURAL_EXPONENT ~ 42.67`, hence `x0 = 2^7` and `x1 = 2^6`, `a0`, `a1` will never be utilized |
| L-3 | Changing the threshold could lead to trove liquidation |
| L-4 | Interest is not accrued when Shrine is shutdown |
| L-5 | Rebase tokens and fee-on-transfer tokens cannot be used as underlying assets for yang |
| NC-1 | Missing NatSpec for most functions |


# L-1. Return values of token transfer are unchecked

https://github.com/code-423n4/2024-01-opus/blob/04583e0411dbf8027952d668a8678fda0cb5b160/src/core/absorber.cairo#L882

## Detail
In the Absorber, the return values of token transfer are not checked to be `true`, which could be false if the transferred tokens are not ERC20-compliant. In that case, the transfer fails without being noticed by the calling contract.

## Recommendation
Check the return value of `transfer()`.

---

# L-2. `MAX_NATURAL_EXPONENT ~ 42.67`, hence `x0 = 2^7` and `x1 = 2^6`, `a0`, `a1` will never be utilized

https://github.com/code-423n4/2024-01-opus/blob/04583e0411dbf8027952d668a8678fda0cb5b160/src/utils/exp.cairo#L21-L24

## Detail
The library `exp.cairo` is a port from `LogExpMath.sol`. While `LogExpMath.sol` supports `MAX_NATURAL_EXPONENT ~ 130.70`, `exp.cairo` only supports `MAX_NATURAL_EXPONENT ~ 42.67`. Consequently, the constants `x0, a0, x1, a1` will never be used since `2^6 = 64`, which is already larger than `MAX_NATURAL_EXPONENT`.

## Recommendation
Remove the uses of x0, a0, x1, and a1.

---

# L-3. Changing the threshold could lead to trove liquidation

https://github.com/code-423n4/2024-01-opus/blob/04583e0411dbf8027952d668a8678fda0cb5b160/src/core/shrine.cairo#L623

## Details
In the Shrine, the `set_threshold()` function allows an authorized caller to set a new threshold for a yang, which takes effect immediately. This abrupt change could cause the `ltv` of the troves to be lower than the new threshold, leading to liquidation.

## Recommendation
Gradually apply the new threshold to prevent user liquidation.

---

# L-4. Interest does not accrue when Shrine is shutdown

https://github.com/code-423n4/2024-01-opus/blob/04583e0411dbf8027952d668a8678fda0cb5b160/src/core/shrine.cairo#L799

## Details
When the Caretaker shuts down the Shrine, it only sets `is_live = false`. However, the interest on the trove is not accrued. This means users no longer need to pay it because the Shrine is now inactive.

---

# L-5. Rebase tokens and fee-on-transfer tokens cannot be used as underlying assets for yang

## Details

In Opus, each yang must maintain a price, which is calculated by multiplying the underlying asset price with the yang-to-asset exchange rate.

However, if a rebase token is used as the underlying asset, its balance changes continuously without corresponding updates to the yang price. This could result in an incorrect yang price stored in Shrine, leading to inaccurate valuation of troves.

Similarly, with fee-on-transfer tokens, the actual amount of the deposited asset decreases when entering a pool, reducing the price of yang whenever a user makes a deposit.

## Recommendation
Document and avoid using rebase and fee-on-transfer tokens as underlying assets in Opus.

---

# NC-1. Missing NatSpec for most functions

## Details
Ensure that the code is well commented on both with NatSpec and inline comments for better readability and maintainability. The comments should accurately reflect what the corresponding code does. 

Currently, most functions are missing NatSpec comments and just uses inline comments.

## Recommendation
Following SNIP-4 here
https://github.com/starknet-io/SNIPs/blob/main/SNIPS/snip-4.md


