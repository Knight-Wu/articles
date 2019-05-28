
### 问题
1. methods 和function 的区别
2. 短赋值的场景
3. 为什么要有指针
* Choosing a value or pointer receiver

There are two reasons to use a pointer receiver.

The first is so that the method can modify the value that its receiver points to.

The second is to avoid copying the value on each method call. This can be more efficient if the receiver is a large struct, for example.


> Written with [StackEdit](https://stackedit.io/).
<!--stackedit_data:
eyJoaXN0b3J5IjpbLTE3NTczOTgyMCwtMzEzNDI1MjE2XX0=
-->