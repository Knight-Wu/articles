http://www.usenix.org/events/usenix10/tech/full_papers/Hunt.pdf
论文笔记, 注重的是思想
### 4. ZooKeeper Implementation
所有的写都会交给leader , folower 只负责转递请求和读, 写会触发一个数据广播的过程, 多台服务器的数据同步, 每次写先保证write ahead log persist to disk 之后, 再去更新内存的数据库, 满足读的高性能. 

写是如何保证幂等的

#### 4.2 Atomic Broadcast
是强顺序一致性的, 广播数据的顺序和客户端写的顺序保持一致

#### 4.3 Replicated Database
跟hdfs 
> Written with [StackEdit](https://stackedit.io/).
<!--stackedit_data:
eyJoaXN0b3J5IjpbLTQ4MTg1MTE1OCwxMTg5NTQ0MTAxLDc0OT
czMDE5MiwtMTg1NzIyODYzN119
-->