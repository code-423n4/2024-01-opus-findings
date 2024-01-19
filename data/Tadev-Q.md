## [L‑01] `deposit`, `withdraw`, `forge` and `melt` functions in Abbot contract don't check for 0 amount, leading to potential event emissions spamming.


## [NC‑01] `ref self: ContractState` should be modified to `self: @ContractState` in multiple functions which do not write into contract storage in Sentinel contract.

A snapshot is enough and will save gas.
Are concerned `suspend_yang`, `unsuspend_yang`, `enter` and `exit` functions.

Also `enter` and `exit` in Gate contract.


## [NC‑02] Wrong natspec about `yang_addresses` variable in Sentinel module, as it is a `1-based array` instead of a `0-based array`.

Indeed, first index to be used is 1, with a LOOP_START variable starting from 1 to retrieve yang addresses in `get_yang_addresses` function.
