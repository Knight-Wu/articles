* 如何使用 page cache 的, 能否精确控制, 只是通过调参数? 
* b tree 支持随机查找? 
per-consumer queue with an associated BTree or other general-purpose random access data structures to maintain metadata about messages
* 如果用 B tree 作为 metadata 的数据结构, 咋看时间复杂度 O(lgN) , 接近线性, 但是放到磁盘上却代价很大, 因为单块 ssd 因为磁头的缘故查找是串行的, 

> Written with [StackEdit](https://stackedit.io/).
<!--stackedit_data:
eyJoaXN0b3J5IjpbMjA2Nzk1NDk4NCwxNjY5NTcwMTExLDEzMj
AwOTUyNjcsLTkyODI2ODQ5Nl19
-->