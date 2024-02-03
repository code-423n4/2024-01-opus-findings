## [QA-1] `TODO`s are still left in the codebase
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