1. The `flash_mint` module allows flashloan capabilities but only for the `Opus` ying (synthetic) as seen here https://github.com/code-423n4/2024-01-opus/blob/4720e9481a4fb20f4ab4140f9cc391a23ede3817/src/core/flash_mint.cairo#L79-L82. 
If the input is any other token, it returns `0`, the `flash_loan()` function provides for a token parameter which allows any user to input any token. Therefore, allowing for `0` flashloan amounts on any other token apart from the synthetic. If the intention of the project is to only allow the synthetic to be flashloaned, it would be advisable to remove the ability of the user to input any other `token` from the `flash_loan ()` function. 

2. The `pragma` module's freshness check bounds is set here https://github.com/code-423n4/2024-01-opus/blob/4720e9481a4fb20f4ab4140f9cc391a23ede3817/src/external/pragma.cairo#L37-L38 as:
```
    const LOWER_FRESHNESS_BOUND: u64 = 60; // 1 minute
    const UPPER_FRESHNESS_BOUND: u64 = consteval_int!(4 * 60 * 60); // 4 hours * 60 minutes * 60 seconds
```
Basically, the `price_validity_thresholds` is allowed to reach a maximum of 4 hrs. This is too dangerous because `Pragma` themselves have a backward compatibility of only `1hr` in their code. The widened range means there is possibility of allowing prices that pragma themselves consider stale. The project should consider adding an additional check when setting price validity thresholds here https://github.com/code-423n4/2024-01-opus/blob/4720e9481a4fb20f4ab4140f9cc391a23ede3817/src/external/pragma.cairo#L157 that ties the set price to what pragma determines as fresh data

