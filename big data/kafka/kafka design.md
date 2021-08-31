* 如何使用 page cache 的, 能否精确控制, 只是通过调参数? 
* b tree 支持随机查找? 
per-consumer queue with an associated BTree or other general-purpose random access data structures to maintain metadata about messages
* 如果用 B tree 作为 metadata 的数据结构, 咋看时间复杂度 O(lgN) , 接近线性, 但是放到磁盘上却代价很大, 因为单块 ssd 因为磁头的缘故查找是串行的, 假设存储的数据翻倍了, 但是因为造成了很多磁盘的操作, 查找的时间就不止翻倍了; 所以用 append , 读写不冲突, 写缓存效率很高, 而且存储用 HDD 性价比高, 可以支持按时间删除数据等特性. 

* producer 为啥用 push 
相比 store and forward, 需要在 producer 持久化, 不可控

### consumer 为啥用 pull 
1. push 消费速率难控制, 速率要不由 broker 控制, 要不就需要来回协调, 还不如pull 由 consumer 控制速率, 以他最大的能力消费; 
2. push 消息延迟难取舍, 若 push , 因为不知道消费的速率则在低延迟的需求下, 需要小批量的发送消息到 consumer, 但是如果 pull , consumer 以尽可能的方式 pull, 由 consumer 具体场景决定了延迟, 而不需要 broker 决定消息的延迟.

### kafka exactly once
1. 消费和发送都是在 kafka topic 间, kafka 有 transaction 支持, 并且能够设置消息的隔离级别, 否则如果是其他目标系统, 则需要目标系统支持两阶段提交. 或者可以将consume offset 和输出结果一起保存, 也可以实现 exactly once , 例如讲 offset 保留到输出文件的文件名. 
2. consumer, commit offset 设置在消息处理的前后能分别达到 at most once 和 at least once. 
3. producer , 当有 retry 的情况是 at least once, 不 retry 就是 at most once, 设置了幂等(需要 ack=all, retry) 则是 exactly once. 

### kafka replication
通过 ISR (in sync replica) , 假设 ISR 有 n+1 个 包括一个 leader, 那么只到 ISR 里面所有机器都 copy 了 message , message 才能当做提交了; 能容忍n 个错误; 相比于 majoritory vote(2n+1个副本), 副本所带来的的写压力大大降低, 但是 majoritory vote 的好处是不需要等待最慢的机器 ack . 
> Written with [StackEdit](https://stackedit.io/).
<!--stackedit_data:
eyJoaXN0b3J5IjpbLTEyNTgzMjIxODMsLTU3MTk1MDM2OCwtMT
gxOTcyNzEzMCwtMTMyOTY0NjEzNCwxMTY1OTkzOTQ0LC0yMTQ0
ODI3NTc2LDE2Njk1NzAxMTEsMTMyMDA5NTI2NywtOTI4MjY4ND
k2XX0=
-->