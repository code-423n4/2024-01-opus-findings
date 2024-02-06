## [L-1] `TODO`s are still left in the codebase
There are 3 instances of these
- https://github.com/code-423n4/2024-01-opus/blob/4720e9481a4fb20f4ab4140f9cc391a23ede3817/src/core/seer.cairo#L216
- https://github.com/code-423n4/2024-01-opus/blob/4720e9481a4fb20f4ab4140f9cc391a23ede3817/src/core/shrine.cairo#L680
- https://github.com/code-423n4/2024-01-opus/blob/4720e9481a4fb20f4ab4140f9cc391a23ede3817/src/core/shrine.cairo#L1667

The presence of `TODO`s comments in a codebase indicates unfinished work on the codebase.
```
File: src/core/seer.cairo
216: // TODO: when possible in Cairo, fetch_price should be wrapped
217: //       in a try-catch block so that an exception does not
218: //       prevent all other price updates
```
```
File: src/core/shrine.cairo
680:  // TODO: temporary workaround for issue with borrowing snapshots in loops
1667: // TODO: temporary workaround for issue with borrowing snapshots in loops
```

Recommendation:
Consider implementing all necessary TODOs and remove the `TODO` comments in the codebase.

## [L-2] Use the latest stable version of the `starknet` package to update the Cairo compiler.
There is 1 instance of this
- https://github.com/code-423n4/2024-01-opus/blob/4720e9481a4fb20f4ab4140f9cc391a23ede3817/Scarb.toml#L17

It is important to use latest version of softwares as they would have latest security fixes and updates. 

The `starknet` package of `scarb` includes a Cairo `compiler` plugin and the version of the `starknet` package is coupled to Cairo version included in `Scarb`.


As of the time of writting this report the `2.5.3` is the latest version of `scarb` which has the latest version of Cairo compiler in the `starknet` package. 

Each `scarb` update version comes with the same version of Cairo compiler so `scarb` version 2.5.3 comes with `Cairo` compiler version `2.5.3`.

See [scarb releases](https://github.com/software-mansion/scarb/releases)
> This version of Scarb comes with Cairo v2.5.3

However this project used version `2.4.0` of the `starknet` package when there are 9 more updated version `releases` after the version 2.4.0.

In Addition, the version `2.4.0` of `scarb` that is used in this project has a warning on it. You can find the link [here]()


>Warning
>This version is not yet supported on Starknet! If you want to develop contracts >deployable to current Starknet, please stick with Scarb v2.3.1
```
[dependencies]
starknet = "2.4.0"
```
Recommendation:
Consider using the latest version `2.5.3` of the `starknet` package to update the Cairo compiler version.

## [L-3] decimals is OPTIONAL for ERC20 tokens.
There are 4 instances of this
- https://github.com/code-423n4/2024-01-opus/blob/4720e9481a4fb20f4ab4140f9cc391a23ede3817/src/core/transmuter.cairo#L564
- https://github.com/code-423n4/2024-01-opus/blob/4720e9481a4fb20f4ab4140f9cc391a23ede3817/src/core/transmuter.cairo#L544
- https://github.com/code-423n4/2024-01-opus/blob/4720e9481a4fb20f4ab4140f9cc391a23ede3817/src/core/gate.cairo#L196
- https://github.com/code-423n4/2024-01-opus/blob/4720e9481a4fb20f4ab4140f9cc391a23ede3817/src/core/gate.cairo#L214


decimals is not part of ERC20 specifications. However this codebase assumes every ERC20 implements the `decimals` state variable and function
- [EIP20 spec](https://eips.ethereum.org/EIPS/eip-20) 

Impact:
Transactions will revert for ERC20 tokens that do not implement `decimals` property and function because it OPTIONAL according to the specification.


Recommendation:
Consider getting `decimals` in a try catch block or manually ensure every token that will be used implements the function.

## [L-4] Return value of ERC20 `transfer(...)` not checked.

There are 4 instances of these
- https://github.com/code-423n4/2024-01-opus/blob/4720e9481a4fb20f4ab4140f9cc391a23ede3817/src/core/transmuter.cairo#L407
- https://github.com/code-423n4/2024-01-opus/blob/4720e9481a4fb20f4ab4140f9cc391a23ede3817/src/core/transmuter.cairo#L448
- https://github.com/code-423n4/2024-01-opus/blob/4720e9481a4fb20f4ab4140f9cc391a23ede3817/src/core/transmuter.cairo#L482
- https://github.com/code-423n4/2024-01-opus/blob/4720e9481a4fb20f4ab4140f9cc391a23ede3817/src/core/absorber.cairo#L882


From the ERC20 interface, the `transfer(...)` returns a boolean that indicates the success of the token transfer. 

However the project does not check the return value of transfer of asset to ensure it returns true.

An example is in the sweep function of transmuter.cairo below.
```
File: transmuter.cairo
417: fn settle(ref self: ContractState) {
     ...
        //@audit return value not checked
448: @> asset.transfer(receiver, asset.balance_of(transmuter));
449:
450:            // Emit event
451:            self.emit(Settle { deficit: total_transmuted })
452:        }
```

Impact:
Loss of asset when the transfer of an asset returns false.


Recommendation:
Consider checking the return values of the token `transfer(...)` to ensure assets are successfully transferred.


## [L-5] Using only snake-case ERC20 functions would render the contracts unusable on starknet mainnet because top ERC20 still use Camelcase naming.

There are 5 instances of this.
- https://github.com/code-423n4/2024-01-opus/blob/4720e9481a4fb20f4ab4140f9cc391a23ede3817/src/core/transmuter.cairo#L581
- https://github.com/code-423n4/2024-01-opus/blob/4720e9481a4fb20f4ab4140f9cc391a23ede3817/src/core/gate.cairo#L231
- https://github.com/code-423n4/2024-01-opus/blob/4720e9481a4fb20f4ab4140f9cc391a23ede3817/src/core/caretaker.cairo#L180
- https://github.com/code-423n4/2024-01-opus/blob/4720e9481a4fb20f4ab4140f9cc391a23ede3817/src/core/transmuter.cairo#L351
- https://github.com/code-423n4/2024-01-opus/blob/4720e9481a4fb20f4ab4140f9cc391a23ede3817/src/core/gate.cairo#L145

ERC20 token on the codebase is done with the assumption that ERC20 tokens on the starknet mainnet used the Rust-like snake-case functions. 

However most top tokens on the Starknet mainnet including the WBTC, ETH and WstETh tokens use the Camelcase naming.

This will make the integrations done with `.balance_of(...)` and `transfer_from(...)` revert because those functions are not present in the WBTC, Eth and WstEth tokens.

an example is in the gate.enter(...) function.
```
//@audit WBtc does not have transfer_from function but transferFrom
File: gate.cairo
let success: bool = self.asset.read().transfer_from(user, get_contract_address(), asset_amt.into());
```

Another example is in the transmuter.cairo for pulling ERC20 asset

```
File: transmuter.cairo
//@audit WBtc does not have transfer_from function but transferFrom
351: let success: bool = self.asset.read().transfer_from(user, get_contract_address(), asset_amt.into());
``` 

One more example is getting the balance of an asset in transmuter.cairo

```
File: transmuter.cairo
//@audit WBtc does not have balance_of function but balanceOf
let asset_balance: u128 = asset.balance_of(get_contract_address()).try_into().unwrap();
```

Impact:
Most functions will revert and be unusable because of the assumption that the tokens implemented snake-cased balance_of and transfer_from.


Recommendation:
Consider implementing both snake-case and camel case integration for interoperability. The convention used by each token can be saved along with each trove data.

## [L-6] Emit events for useful state update
There are 5 instances of this.
- https://github.com/code-423n4/2024-01-opus/blob/4720e9481a4fb20f4ab4140f9cc391a23ede3817/src/core/abbot.cairo#L79C5-L82C6
- https://github.com/code-423n4/2024-01-opus/blob/4720e9481a4fb20f4ab4140f9cc391a23ede3817/src/core/abbot.cairo#L191C9-L212C10
- https://github.com/code-423n4/2024-01-opus/blob/4720e9481a4fb20f4ab4140f9cc391a23ede3817/src/core/absorber.cairo#L227C5-L236C6
- https://github.com/code-423n4/2024-01-opus/blob/4720e9481a4fb20f4ab4140f9cc391a23ede3817/src/core/equalizer.cairo#L84C5-L91C6
- https://github.com/code-423n4/2024-01-opus/blob/4720e9481a4fb20f4ab4140f9cc391a23ede3817/src/core/flash_mint.cairo#L66C5-L68C6
- https://github.com/code-423n4/2024-01-opus/blob/4720e9481a4fb20f4ab4140f9cc391a23ede3817/src/core/gate.cairo#L63C5-L69C6
- https://github.com/code-423n4/2024-01-opus/blob/4720e9481a4fb20f4ab4140f9cc391a23ede3817/src/core/sentinel.cairo#L99C5-L102C6
- https://github.com/code-423n4/2024-01-opus/blob/4720e9481a4fb20f4ab4140f9cc391a23ede3817/src/core/sentinel.cairo#L250C9-L269C10
- https://github.com/code-423n4/2024-01-opus/blob/4720e9481a4fb20f4ab4140f9cc391a23ede3817/src/core/transmuter.cairo#L174C5-L194C6
- https://github.com/code-423n4/2024-01-opus/blob/4720e9481a4fb20f4ab4140f9cc391a23ede3817/src/core/transmuter_registry.cairo#L65C9-L75C10
- https://github.com/code-423n4/2024-01-opus/blob/4720e9481a4fb20f4ab4140f9cc391a23ede3817/src/core/transmuter_registry.cairo#L51C5-L53C6


The construstors and some functions in the codebase update state without emitting events. For example in abbot.cairo, the contructor, deposit and withdraw functions update state without emitting events.

Emiting events helps offchain monitoring tools like [Openzeppelin Defender Sentinel](https://www.openzeppelin.com/defender) monitor activities in order to detect unusual malicious contracts state updates.


Recommendation:
Consider emitting event for important state updates.
 