http://www.usenix.org/events/usenix10/tech/full_papers/Hunt.pdf
论文笔记, 注重的是思想
### 4. ZooKeeper Implementation
所有的写都会交给leader , folower 只负责转递请求和读, 写会触发一个数据广播的过程, 多台服务器的数据同步, 每次写先保证write ahead log persist to disk 之后, 再去更新内存的数据库, 满足读的高性能. 

写是如何保证幂等的

#### 4.2 Atomic Broadcast
是强顺序一致性的, 广播数据的顺序和客户端写的顺序保持一致

#### 4.3 Replicated Database
跟hdfs 一样有一个周期性的checkpoint 功能, 将内存数据 dump 到磁盘上, 为了故障恢复, 但是这个snapshot 跟实际内存数据还是会有一些差别, 跟hdfs 一样, 恢复的时候会把这个snapshot 和最新的修改合并. 那么最新的修改是怎么存储的, 单节点问题如何恢复. 

#### 4.4 Client-Server Interactions
读请求不一定能获取到最近更新的写请求的结果, 因为写的结果可能还没同步到其他server 的内存中, 若要保障读请求的实时性, zk 实现了sync, 
使用FIFO l
> Written with [StackEdit](https://stackedit.io/).
<!--stackedit_data:
eyJoaXN0b3J5IjpbMTM3ODc2NTE5MiwxNzI2NDkxMzUxLDExOD
k1NDQxMDEsNzQ5NzMwMTkyLC0xODU3MjI4NjM3XX0=
-->