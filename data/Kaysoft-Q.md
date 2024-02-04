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