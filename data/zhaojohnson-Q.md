- In seer.update_frequency, system has one boundary for frequency, from LOWER_UPDATE_FREQUENCY_BOUND to UPPER_UPDATE_FREQUENCY_BOUND. In constructor, we do not add some check to make sure input parameter is valid.
https://github.com/code-423n4/2024-01-opus/blob/4720e9481a4fb20f4ab4140f9cc391a23ede3817/src/core/seer.cairo#L98-L110

- In flash_mint::flash_loan(), after borrower's callback, we will eject amount_wad + FLASH_FEE. Although currently FLASH_FEE is zero, we'd better to eject(amount_wad + FLAHS_FEE) considering possible FLASH_FEE's change in future.
https://github.com/code-423n4/2024-01-opus/blob/4720e9481a4fb20f4ab4140f9cc391a23ede3817/src/core/flash_mint.cairo#L100-L132
