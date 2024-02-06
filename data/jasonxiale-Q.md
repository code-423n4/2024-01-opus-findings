## [L-01] `abbot.deposit` and `abbot.melt` lack of check the caller is the owner of the trove
File:
https://github.com/code-423n4/2024-01-opus/blob/4720e9481a4fb20f4ab4140f9cc391a23ede3817/src/core/abbot.cairo#L191-L200
https://github.com/code-423n4/2024-01-opus/blob/4720e9481a4fb20f4ab4140f9cc391a23ede3817/src/core/abbot.cairo#L222-L226
Both of `deposit` and `melt` lack of check the caller is the own of trove, if a user call [abbot.deposit](https://github.com/code-423n4/2024-01-opus/blob/4720e9481a4fb20f4ab4140f9cc391a23ede3817/src/core/abbot.cairo#L191-L200) with wrong trove_id, he/she will lost the tokens. And [abbot.melt](https://github.com/code-423n4/2024-01-opus/blob/4720e9481a4fb20f4ab4140f9cc391a23ede3817/src/core/abbot.cairo#L222-L225) has the same issue.

## [L-02] `absorber.transfer_assets` doesn't check `transfer's return value`
File:
https://github.com/code-423n4/2024-01-opus/blob/4720e9481a4fb20f4ab4140f9cc391a23ede3817/src/core/absorber.cairo#L876C12-L888
In `transfer` is used in [absorber.cairo#L882], but its return value is not checked.

## [L-03] stale price might be used in `shrine`
File:
https://github.com/code-423n4/2024-01-opus/blob/4720e9481a4fb20f4ab4140f9cc391a23ede3817/src/core/seer.cairo#L135-L154
https://github.com/code-423n4/2024-01-opus/blob/4720e9481a4fb20f4ab4140f9cc391a23ede3817/src/core/seer.cairo#L193-L239
https://github.com/code-423n4/2024-01-opus/blob/4720e9481a4fb20f4ab4140f9cc391a23ede3817/src/core/sentinel.cairo#L174-L214
Impact:
In current, new yang can be added by [sentinel.add_yang](https://github.com/code-423n4/2024-01-opus/blob/4720e9481a4fb20f4ab4140f9cc391a23ede3817/src/core/sentinel.cairo#L174-L214), and the oracles can be set by calling [seer.set_oracles](https://github.com/code-423n4/2024-01-opus/blob/4720e9481a4fb20f4ab4140f9cc391a23ede3817/src/core/seer.cairo#L134-L154), and finally the yang price can be updated by [seer.update_prices_internal](https://github.com/code-423n4/2024-01-opus/blob/4720e9481a4fb20f4ab4140f9cc391a23ede3817/src/core/seer.cairo#L193-L239)
There is a flaw in seer is that **Both set_oracles and update_prices_internal doesn't check if all yangs has properly oracle**.
So there might be a case that:
1. the admin calls `sentinel.add_yang` to add a new yang
1. the oracles aren't updated for the new yang
1. even though all the yangs' price will be updated with time gap `update_frequency`, the new added yang's price won't be updated.
1. in such case, the newly added yang will keep using the stale price.

## [L-04] `unsuspend_yang` should check if get_yang_suspension_status returns `YangSuspensionStatus::Temporary` instead of `YangSuspensionStatus::Permanent`
File:
    https://github.com/code-423n4/2024-01-opus/blob/4720e9481a4fb20f4ab4140f9cc391a23ede3817/src/core/shrine.cairo#L640
Because `shrine.unsuspend_yang` will emit an `YangUnsuspended` event in [shrine.cairo#L644](https://github.com/code-423n4/2024-01-opus/blob/4720e9481a4fb20f4ab4140f9cc391a23ede3817/src/core/shrine.cairo#L644), it'll make more sense that the function checks if `get_yang_suspension_status` return value equal to `YangSuspensionStatus::Temporary`
```diff
diff --git a/src/core/shrine.cairo b/src/core/shrine.cairo
index 0236e95..62b9829 100644
--- a/src/core/shrine.cairo
+++ b/src/core/shrine.cairo
@@ -637,7 +637,7 @@ mod shrine {
             self.access_control.assert_has_role(shrine_roles::UPDATE_YANG_SUSPENSION);

             assert(
-                self.get_yang_suspension_status(yang) != YangSuspensionStatus::Permanent, 'SH: Suspension is permanent'
+                self.get_yang_suspension_status(yang) == YangSuspensionStatus::Temporary, 'SH: Suspension is permanent'
             );

             self.yang_suspension.write(self.get_valid_yang_id(yang), 0);
```