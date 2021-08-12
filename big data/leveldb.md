### memoreStore
#### 跳表
说得很清楚: https://www.jianshu.com/p/9d8296562806
查询的时间复杂度是 O(lgn), n 为元素个数, 近似为二分查找, 等同于跳表的高度, 
插入和删除的时间复杂度其实也是 O(lgn), 但是比查询多了一点, 因为需要构建和删除索引, 
范围查询的时间复杂度是 查询到首节点( O(lgn)) + 后续链表遍历的时间复杂度(可以当做常数) , 
适用于 LSM 类存储引擎(Hbase, levelDB)做 memStore 的数据结构, 以及 redis zset , 因为插入, 查询, 删除的时间复杂度都是 O(lgn) , 而且底层链表又是有序的, 可以直接持久化做 SSTable, 

### Compaction

* minor compaction
当 memoryStore 增长到一定大小, 会变成 immutable memoryStore, ran'h
> Written with [StackEdit](https://stackedit.io/).
<!--stackedit_data:
eyJoaXN0b3J5IjpbMTcwMzMzOTY1XX0=
-->