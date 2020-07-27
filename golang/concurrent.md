*  waitgroup add 必须要在 go func() 的外面, 保证 happen before waitgroup wait().
* break in select means break out in one loop, 如果外面还有 for loop, 并不能跳出, 需要一个标志位. 

> Written with [StackEdit](https://stackedit.io/).
<!--stackedit_data:
eyJoaXN0b3J5IjpbLTQxNTA2ODI0MywyMDM0MTQzMTY4XX0=
-->