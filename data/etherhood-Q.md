### Zero address check not present
- https://github.com/code-423n4/2024-01-opus/blob/main/src/core/caretaker.cairo#L107
- https://github.com/code-423n4/2024-01-opus/blob/main/src/core/shrine.cairo#L388
- https://github.com/code-423n4/2024-01-opus/blob/main/src/core/flash_mint.cairo#L66
- https://github.com/code-423n4/2024-01-opus/blob/main/src/core/abbot.cairo#L79
- https://github.com/code-423n4/2024-01-opus/blob/main/src/core/absorber.cairo#L227
- https://github.com/code-423n4/2024-01-opus/blob/main/src/core/controller.cairo#L91
- https://github.com/code-423n4/2024-01-opus/blob/main/src/core/equalizer.cairo#L84
- https://github.com/code-423n4/2024-01-opus/blob/main/src/core/gate.cairo#L63
- https://github.com/code-423n4/2024-01-opus/blob/main/src/core/purger.cairo#L134
- https://github.com/code-423n4/2024-01-opus/blob/main/src/core/seer.cairo#L104
- https://github.com/code-423n4/2024-01-opus/blob/main/src/core/sentinel.cairo#L99

### Invalid frequency can be set during contract deployment
- https://github.com/code-423n4/2024-01-opus/blob/main/src/core/seer.cairo#L108
In `set_update_frequency` it is checked that frequency is in range, whereas during contract deployment, it is not checked in constructor, which can lead to invalid frequency being set

### Add `charge()` in redistribute
- https://github.com/code-423n4/2024-01-opus/blob/main/src/core/shrine.cairo#L921
In redistribute function `charge` is not called to compound debt and attribute redistributed collateral to troves. It is working now, but in future if purger is updated, it will always rely on purger to have called `charge` which might not be the case