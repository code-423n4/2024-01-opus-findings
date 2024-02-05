## More Flash loan than expected is released

Git:
https://github.com/code-423n4/2024-01-opus/blob/main/src/core/flash_mint.cairo#L81C17-L82C75

Issue:
max_flash_loan is not considering if any negative budget is in place. If negative budget is present (may be possible in future) then net yin supply will become lower. Thus flash loan amount become slightly higher

Recommendation:
It is better to give flash loan on net yin

## Unused role and event

Links:
https://github.com/code-423n4/2024-01-opus/blob/main/src/external/pragma.cairo#L89
https://github.com/code-423n4/2024-01-opus/blob/main/src/core/roles.cairo#L64

Issue:
role SET_ORACLE_ADDRESS and event OracleAddressUpdated are not used and can be removed

## Threshold change should be behind timelock

Link:
https://github.com/code-423n4/2024-01-opus/blob/main/src/core/shrine.cairo#L620

Issue:
Admin can change threshold if requirement arises but if threshold is lowered then multiple trove position may become unhealthy.
Changing threshold could be behind timelock which gives sufficient time to user for adding more collateral with respect to new threshold

## Oracle failure for one asset cause whole update fail

Link:
https://github.com/code-423n4/2024-01-opus/blob/main/src/core/seer.cairo#L224

Issue:
If update_prices_internal is called with force true and oracle returns 0 price for one of the asset(due to no source with price information on asset) then `shrine.advance` (0 price check) fails, which fails the entire loop, meaning no asset price gets updated

Recommendation:
If any of the asset price fetch cause exception then only that asset price update should be ignored by using exception handling


