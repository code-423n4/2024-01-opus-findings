# Lows

### [1] Missing input validation checks on arguments `yang_asset_max` and `yang_threshold` on function `add_yang()`

- [fn add_yang()](https://github.com/code-423n4/2024-01-opus/blob/4720e9481a4fb20f4ab4140f9cc391a23ede3817/src/core/sentinel.cairo#L174-L181)

Missing zero checks on `yang_asset_max` and `yang_threshold` before they are written to state.

### [2] Race condition for Yin approve.

Though starknet currently has only one sequencer which makes frontrunning impossible since txs are included first come first serve basis. In the future when more sequencers are included this will be the same problem faced by EVM chains

POC:

https://www.adrianhetman.com/unboxing-erc20-approve-issues/

Recommendation:

Use increaseAllowance and decreaseAllowance functions as a wrapper to the approve function which either increments or decrements the allowance making it frontrun proof.

## QA

### [1] Call to `forge` in shrine not required when Yin is not to be created

https://github.com/code-423n4/2024-01-opus/blob/4720e9481a4fb20f4ab4140f9cc391a23ede3817/src/core/abbot.cairo#L156

When a user opens a trove by calling the `open_trove` function, they have the choice of not minting any debt by setting `forge_amount` to be zero. A call to shrine.forge can be skipped as no debt is to be minted to the trove.

### [2] Missing event emission

https://github.com/code-423n4/2024-01-opus/blob/4720e9481a4fb20f4ab4140f9cc391a23ede3817/src/core/shrine.cairo#L388

Missing `YinPriceUpdated` event emission in shrine constructor after yin spot price is set for the first time.

### [3] Misleading error message

https://github.com/code-423n4/2024-01-opus/blob/4720e9481a4fb20f4ab4140f9cc391a23ede3817/src/core/shrine.cairo#L658

Misleading error message in function update_rates when asserting input array yang length to be correct. Error message `“SH: Too few yangs”` falsely implies that the length of yangs in the input array is less than what is required when failing but it could be more as well.

Recommendation:
Change error message to "SH: Incorrect number of yangs"