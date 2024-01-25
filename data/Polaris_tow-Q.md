# Lacks slippage protection
https://github.com/code-423n4/2024-01-opus/blob/4720e9481a4fb20f4ab4140f9cc391a23ede3817/src/core/gate.cairo#L137
https://github.com/code-423n4/2024-01-opus/blob/4720e9481a4fb20f4ab4140f9cc391a23ede3817/src/core/gate.cairo#L155
## Impact
The lack of slippage protection could expose users to sandwich attacks. 

## Proof of Concept
The Gate Module, which is similar to an ERC4626 implementation, lacks slippage protection in its contract. This makes users vulnerable to sandwich attacks. For example, a user might receive less yang_amt than expected when staking asset_amt through the `enter` function. The `exit` function has a similar issue. For specific implementation details, you can refer to the following:https://github.com/ERC4626-Alliance/ERC4626-Contracts/blob/main/src/ERC4626Router.sol 
https://ethereum.stackexchange.com/questions/125772/erc4626-and-slippage

```rust
        fn enter(ref self: ContractState, user: ContractAddress, trove_id: u64, asset_amt: u128) -> Wad {
            self.assert_sentinel();

            let yang_amt: Wad = self.convert_to_yang_helper(asset_amt);
            if yang_amt.is_zero() {
                return WadZeroable::zero();
            }

            let success: bool = self.asset.read().transfer_from(user, get_contract_address(), asset_amt.into());
            assert(success, 'GA: Asset transfer failed');
            self.emit(Enter { user, trove_id, asset_amt, yang_amt });

            yang_amt
        }
```
## Recommended Mitigation Steps
Implement slippage protection measures. For specific implementation details, you can refer to the ERC4626Router at ERC4626-Contracts on GitHub :https://github.com/ERC4626-Alliance/ERC4626-Contracts/blob/main/src/ERC4626Router.sol.

