## 1、The recursion length can be modified by changing the parameter
The recursion length can be modified by changing the parameter
This is a recursive call. You can shorten the recursion length by changing the parameter provision.shares to adjusted_shares so that the previous result is the start of the next operation.
https://github.com/code-423n4/2024-01-opus/blob/4720e9481a4fb20f4ab4140f9cc391a23ede3817/src/core/absorber.cairo#L853-L854

## 2、You can judge when the function comes in, if provision.shares == 0, it will return directly, no need to run to the sub-function, so as to save gas.
https://github.com/code-423n4/2024-01-opus/blob/4720e9481a4fb20f4ab4140f9cc391a23ede3817/src/core/absorber.cairo#L768-L776

## 3、Put the zero judgment in the front to reduce the data reading
You can put (*recipient_yang_balance.amount).is_zero() in front of it to reduce the read of yang_to_yang_redistribution

https://github.com/code-423n4/2024-01-opus/blob/4720e9481a4fb20f4ab4140f9cc391a23ede3817/src/core/shrine.cairo#L2112-L2125