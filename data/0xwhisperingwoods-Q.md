### [qa-1] The `liquidate` in `mod purger::liquidate` function does not prioritize a caller fee

**Description:** The `mod purger::liquidate` function fails to prioritize the caller fee. In some cases, the fee could end up being zero, the consequence of which there would be minimal incentive to liquidate. 

The `mod purger::liquidate` function does not explicitly define how the caller receives their fee. Therefore the liquidations could fail to occur on time, leading to bad debts. 

**Impact:** The callers could lack incentive to the point that they are unwilling to participate in the liquidation of bad debts. As a consequence, the protocol could end up with too much bad debt as the threshold could be surpused on a downward spiral.  

**Proof of Concept:** 
1. The user tries to `liquidate` a falling position by calling the function. 
2. Seeing that there is no sufficient incentive, gets off
3. Bad debt could accrue in the protocol as a result

**Recommended Mitigation:** The `mod purger::liquidate` function should independently calculate the caller fee. This would prevent a scenario where they arrive at `0` and thus are no longer interested in liquidation. 


### [qa-2] minimal problem in `mod gate::enter` and `mod gate::exit` that could allow reentrancy

**Description:** Although the `self.assert_sentinal()` would have checks for potential reentrancy, the fact that the functions are open in this manner could still pose some minimal risks. 

`enter`

```javascript
     let success: bool = self
                .asset
                .read()
                .transfer_from(user, get_contract_address(), asset_amt.into());
            assert(success, 'GA: Asset transfer failed');
```
`exit`

```javascript
    
        @>  let success: bool = self.asset.read().transfer(user, asset_amt.into());
            assert(success, 'GA: Asset transfer failed');

            
```


**Impact:** In case the `self.assert_sentinel` is bypassed, the user could tamper with the functionality of the functions. 

**Recommended Mitigation:** The `mod gate::enter` and `mod gate::exit` should avoid assumption of safety and put an extra measure by using the CEI standard. By doing so, ensure that external calls comes after all state changes and computations have been done. 