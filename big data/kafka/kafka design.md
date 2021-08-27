* 如何使用 page cache 的, 能否精确控制, 只是通过调参数? 
* b tree 支持随机查找? 
per-consumer queue with an associated BTree or other general-purpose random access data structures to maintain metadata about messages
* 如果用 B tree 作为 metadata 的数据结构, 咋看时间复杂度 O(lgN) , 接近线性, 但是放到磁盘上却代价很大, 因为单块 ssd 因为磁头的缘故查找是串行的, 假设存储的数据翻倍了, 但是因为造成了很多磁盘的操作, 查找的时间就不止翻倍了; 所以用 append , 读写不冲突, 写缓存效率很高, 而且存储用 HDD 性价比高, 可以支持按时间删除数据等特性. 

* zero copy 
如果不用 zero copy, 那么从读文件到通过网络发送需要经过以下四个步骤
1. 操作系统从磁盘读文件到内核空间的 page cache
2. 应用从内核空间的 page cache 读到用户
> Written with [StackEdit](https://stackedit.io/).
<!--stackedit_data:
eyJoaXN0b3J5IjpbLTE3NzI0Njc2NDAsMTE2NTk5Mzk0NCwtMj
E0NDgyNzU3NiwxNjY5NTcwMTExLDEzMjAwOTUyNjcsLTkyODI2
ODQ5Nl19
-->