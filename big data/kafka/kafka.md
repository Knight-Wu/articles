# kafka consume protocol
## 新建一个 consumer group 或者 rebalance 时候的步骤

1. 找到消费组 group 所属的 corrdinator
![image](https://user-images.githubusercontent.com/20329409/235361432-ac7d1a9c-ab80-4c0c-a386-c69c205473ec.png)

1.1 消费者 client 发送 findCorrdinator request, 携带 groupId 到任何一个 broker. 
1.2 hash(groupId) 对 __consumer_offset 的 partition 数量取模, partition 所属的 leader broker 即为 group corrdinator. 并返回 group Corrdinator 的地址给 client.


2. members join(成员变更)
![image](https://user-images.githubusercontent.com/20329409/235361604-972e9bf7-bb01-4181-927c-b6a07befc0c9.png)
2.1 选取第一个加入的 consumer 作为 consumer leader.
2.2 group Corrdinator 会返回所有消费者实例信息给 consumer leader, 然后 leader 就可以根据策略去生成 partition 分配的规则, 哪个 partition 分配给哪个 consumer.


3. Partitions Assigned
![image](https://user-images.githubusercontent.com/20329409/235362069-32348847-06ac-46b6-a27a-82208ed1b8ba.png)

3.1 consumer leader 会发起 SyncGroupRequest , 把partition 分配的规则发给 group Corrdinator. 
3.2 group Corrdinator 就把具体的分配信息发送给对应的 consumer, 每个 consumer 只知道自己需要消费的 partition, 然后 consumer 就开始消费了. 

## commitOffsetRequest(提交 offset )

![image](https://user-images.githubusercontent.com/20329409/235362305-6eae5f00-3e70-47f5-8f31-2b815ba303bc.png)

## offsetFetchRequest
![image](https://user-images.githubusercontent.com/20329409/235390137-30a6b129-b46d-40b5-96d9-f06ee4da1c19.png)

##  Static Group Membership 可以避免 rebalance
![image](https://user-images.githubusercontent.com/20329409/235407997-b35f3467-5bd5-4fe8-847e-58fce6cfc4b9.png)

* 某个 consumer 只要制定了 group.instance.id, 如果重启了在session.timeout.ms 之前恢复都不会触发 rebalance, 当然这个过程中consumer 所消费的 partition 是不会被消费的
* 当有新成员加入时肯定会触发Rebalance重新分配分区
# kafka transaction 事务
## 大体流程
![image](https://user-images.githubusercontent.com/20329409/222062903-b0b174d1-a94f-4a04-8809-9cb798efb59e.png)

1. FindCoordinatorRequest , 根据transactionId 找对应的transaction corrdinator, transactionId hash 之后对 __transaction_state topic 的partition 取余, 然后partition 所在的leader 就是 transaction corrdinator
2. 根据 transaction id 返回一个固定的producer id, 用于继续执行或者回滚之前的遗留事务.
3. producer begin transaction, transaction corrdinator 记录 发送的topicpartition 到 transaction log.
4. producer 发送信息到各个partition
5. producer 发送 EndTxnRequest, tc 先写 prepare commit 日志(5.1a)
6. transaction coordinator 发送 transaction marker 到 leader partition 标志这些消息可见或清除, leader 并把这些消息记录在partition 中.
7. transaction coordinator  最后写事务结果 到 transaction log.
8. consumer 根据事务级别是 read committed, 还是 read uncommited, 判断这些消息是否可见. 如果是 read committed ,要发送了success transaction marker 之后这些消息就才能被 consumer 读到. 

## 详细流程
![image](https://user-images.githubusercontent.com/20329409/222064514-928c32aa-f79a-4456-b564-4a3402a8a25f.png)

</br>
1. 找transaction coordinator -- FindCoordinatorRequest , 根据transactionId 找对应的transaction corrdinator, transactionId hash 之后对 __transaction_state topic 的partition 取余, 然后partition 所在的leader 就是 transaction corrdinator
</br>
2. 获取producerId -- InitPidRequest, 每一个transactionId 返回一个固定的 pid, 用于继续执行或者回滚之前的遗留事务.
</br>
3. 开始事务 --  The beginTransaction() API
</br>
4. 如果是先消费再producer 的场景会包括多种请求:
</br>
4.1 图中4.1a, AddPartitionsToTxnRequest, 事务当中每次新写一个topic partition, 就写 tp 的信息到transaction log, 用于后续写transaction marker. 
</br>
4.2 producer 发送消息到broker -- ProduceRequest
</br>
4.3 AddOffsetCommitsToTxnRequest, 图中 4.3a, 如果是producer 先消费一个topic 再写另一个topic 的场景, 会记录消费的offset 到 transaction log
</br>
4.4 TxnOffsetCommitRequest, 图中 4.4 a, producer 会发送消费的offset 信息到consumer coordinator 持久化, cc 会校验 pid 和 epoch(版本号), 排除那些旧的producer, 除非事务提交不然offset 的变化是不可见的.
</br>
5. 提交或中止事务
</br>
5.1 producer 发送 EndTxnRequest, 先写准备日志(5.1a), 再发送 transaction marker 到 partition, 最后写 commit 或 abort 到 transaction log.
</br>
5.2 WriteTxnMarkerRequest, transaction coordinator 发送 WriteTxnMarkerRequest 给 leader partition 来让这些消息可见或者清除, 图中 5.2a
</br>
5.3 写最后的commit 结果, 成功或者回滚, 写到transaction log, 图中 5.3 , 然后transaction log 就可以清除了. 


### 参考
1. https://juejin.cn/post/7122295644919693343
2. https://cwiki.apache.org/confluence/display/KAFKA/KIP-98+-+Exactly+Once+Delivery+and+Transactional+Messaging#KIP98ExactlyOnceDeliveryandTransactionalMessaging-DataFlow
# log end offset and high watermark
每个partition 的每个副本都有自己的LEO, 标识当前写入的最后一条消息. 
每个partition 的每个副本都有自己 HW(高水位), 每个partition 的leader的 HW = max(current HW,min(all replicas LEO)), 可以理解为在HW 之前的消息才可读, 不会出现幻读等场景. 
replica 的 HW = min(fetch 消息的时候leader 发送的自己的HW, replica 自己的LEO)

* 0.10 版本的kafka 在采用HW 作为replica 恢复时有可能会造成数据不一致. 

* 例如第一种情况 

1. client 向 leader 写入一条消息 m0( offset 0), leader LEO=1, 
2. replica 向 leader l 拉取消息, 发送自己的LEO=0, l 返回 消息m0, HW = 0.
3. client 继续写入一条消息 m1, leader LEO = 2,
4. r 向l 拉取消息, 发送自己LEO=1, l 回复 消息m1, 并更新 l HW = 1, 返回 HW=1, 意味着小于offset 1 的消息都被所有replica 提交, 对消费者可读取. r 的HW = min(r LEO, l HW) = 1, r 写消息m1 进入本地文件系统cache.
5. r 继续向l 拉取消息, 发送 LEO=2, l 此时没有新消息回复, 更新 l HW=2, 但此时r 在收到此回复之前下线再重启, 然后此时 r HW = 1, offset 小于1的消息才是提交的,
6. 此时r 向l fetch 消息就能更新自己的HW 与l 保持一致, 但此时l 挂了, 那么r 只能根据HW=1, truncate offset=1 的消息m1
7. 所以之前m1 消息在l 已经被提交了, 可能被读取了, 但此时 m1 又不存在了, 造成了数据的不一致. 

* 另一种情况
在上述情况的第五步之后, l 和 r 都挂了, 由于 r 的m1 只写入了cache, 彻底丢失
1. 然后 r 先回复, 设置了unclean.leader.enable = true, r 被选为leader, 新写入一条消息 m2, 此时r offset=0 是m0, offset=1 是m2.
2. 此时 l 回复 , HW = 2, 那么 从 r(此时的leader) 发送fetch req, LEO = 2, 从offset=2 开始返回, 那么 offset=1 这个位置两个副本就产生了不一致的消息, 一个是m2, 一个是 m1. 

* 数据不一致的根本原因
 
replica 的HW 是在下次fetch 请求才会拉取leader 的HW , 所以leader 的HW 和replica 的HW 是有可能出现某些时间上是不一致的. 
根据replica 的HW 来恢复是不可靠的. 


* 那么kafka 是如何解决的呢


加入一个leader epoch, 类似leader 的版本号, 和当前epoch 下的第一条消息的offset, 和raft 很类似. 
第一种情况, r 重启后去 l 发送 leaderEpochRequest, 如果l 没挂会返回对应的start offset, 就不会截断, 如果l 挂了, 那么r 被选为leader ,也不会截断, 但是如果 r 在数据落盘之前就挂了呢, 如果能接受unclean.leader, 而且此时是 l 和r 一起挂, 是能接受理论上数据丢失的. 

第二种情况, r 先恢复, r offset=0 是m0, offset=1 是m2, 变为新的l, l 恢复后会去 r 发起leaderEpochReq, 返回更大的epoch, 和startOffset=1, 那么 旧的l 就会截断1 开始的日志, 并从LEO = 1 开始fetch. 

* leader epoch 的步骤

如果一个replica 变成了leader, 就增加epoch, 并把LEO 作为startOffset, 并刷盘. 
如果一个r 变成了follower, 就会把当前磁盘的epoch 发给 leader, leader , 如果epoch 和leader 相同, 就返回leader 的LEO, 如果大于LEO, 就截断, 否则直接以r 的LEO fetch;
如果epoch 小于leader, 则返回新的epoch, 和startOffset, 大于startOffset 的消息开始截断, 并从截断后的LEO 开始fetch.

* 如何选leader
多个broker 一起监听 zk 事件, 由zk 保证只有某个 broker 竞争成功成为controller, 然后由这个去轮训分发每个partition, 哪个replica 所在的broker 为leader. 当然需要一些负载均衡策略. 
### 手动计算consume 和 produce 的速度
date;./kafka-consumer-s.sh --describe --group groupid  --broker:9092| sort > lag.msg
执行这个命令两次, 记录时间, 然后将两个 lag.msg 导入到 google sheet, 但是初始数据都在一列, 此时 选择 Data 菜单下" split text to columns " , 就可以分成多列, 通过两个时间点内的 current-offset 的相差计算consume 速度, log end offset(是最新的一条消息进入到 log 里面的位置) 计算 produce 的速度. 

* 如何使用 page cache 的, 能否精确控制, 只是通过调参数? 
* b tree 支持随机查找? 
per-consumer queue with an associated BTree or other general-purpose random access data structures to maintain metadata about messages
* 如果用 B tree 作为 metadata 的数据结构, 咋看时间复杂度 O(lgN) , 接近线性, 但是放到磁盘上却代价很大, 因为单块 ssd 因为磁头的缘故查找是串行的, 假设存储的数据翻倍了, 但是因为造成了很多磁盘的操作, 查找的时间就不止翻倍了; 所以用 append , 读写不冲突, 写缓存效率很高, 而且存储用 HDD 性价比高, 可以支持按时间删除数据等特性. 

* producer 为啥用 push 
相比 store and forward, 需要在 producer 持久化, 不可控

### consumer 为啥用 pull
1. push 消费速率难控制, 速率要不由 broker 控制, 要不就需要来回协调, 还不如pull 由 consumer 控制速率, 以他最大的能力消费; 
2. push 消息延迟难取舍, 若 push , 因为不知道消费的速率则在低延迟的需求下, 需要小批量的发送消息到 consumer, 但是如果 pull , consumer 以尽可能的方式 pull, 由 consumer 具体场景决定了延迟, 而不需要 broker 决定消息的延迟.

### message exactly once
1. 消费和发送都是在 kafka topic 间, kafka 有 transaction 支持, 并且能够设置消息的隔离级别, 否则如果是其他目标系统, 则需要目标系统支持两阶段提交. 或者可以将consume offset 和输出结果一起保存, 也可以实现 exactly once , 例如讲 offset 保留到输出文件的文件名. 
2. consumer, commit offset 设置在消息处理的前后能分别达到 at most once 和 at least once. 
3. producer , 当有 retry 的情况是 at least once, 不 retry 就是 at most once, 设置了幂等(需要 ack=all, retry) 则是 exactly once. 

### replication
通过 ISR (in sync replica) , 假设 ISR 有 n+1 个 包括一个 leader, 那么只到 ISR 里面所有机器都 copy 了 message , message 才能当做提交了, 能容忍n 个错误; 相比于 majority vote(2n+1个副本), 副本所带来的的写压力大大降低, 但是majority  vote 的好处是不需要等待最慢的机器 ack . 

### leader election
how 

### log compaction
message 由 key 和 value 组成, 每隔一段时间会对 log entry 进行 compact, 相同 key 只保留最新的那个 keyVal, 
> Written with [StackEdit](https://stackedit.io/).
<!--stackedit_data:
eyJoaXN0b3J5IjpbODUxMDg1Mjg3LDE2MDY0NTE3OTgsMTA3Nj
c2NTA1NiwtNTcxOTUwMzY4LC0xODE5NzI3MTMwLC0xMzI5NjQ2
MTM0LDExNjU5OTM5NDQsLTIxNDQ4Mjc1NzYsMTY2OTU3MDExMS
wxMzIwMDk1MjY3LC05MjgyNjg0OTZdfQ==
-->

### __consumer_offset topic
* 消息种类

__consumer_offsets中保存的记录是普通的Kafka消息，只是它的格式完全由Kafka来维护，用户不能干预。严格来说，__consumer_offsets中保存三类消息，分别是：
Consumer group组元数据消息
Consumer group位移消息
Tombstone消息
* Consumer group组元数据消息

我们都知道__consumer_offsets是保存位移的，但实际上每个消费者组的元数据信息也保存在这个topic。这些元数据包括：
这里不详细展开组元数据各个字段的含义。我们只需要知道组元数据消息也是保存在__consumer_offsets中即可。值得一提的是， 如果用户使用standalone consumer（即consumer.assign(****)方法），那么就不会写入这类消息，毕竟我们使用的是独立的消费者，而没有使用消费者组。
这类消息的key是一个二元组，格式是【版本+groupId】，这里的版本表征这类消息的版本号，无实际用途；而value就是上图所有这些信息打包而成的字节数组。

* Consumer group组位移提交消息

如果只允许说出__consumer_offsets的一个功能，那么我们就记住这个好了：__consumer_offsets保存consumer提交到Kafka的位移数据。这句话有两个要点：1. 只有当consumer group向Kafka提交位移时才会向__consumer_offsets写入这类消息。如果你的consumer压根就不提交位移，或者你将位移保存到了外部存储中(比如Apache Flink的检查点机制或老版本的Storm Kafka Spout)，那么__consumer_offsets中就是无位移数据；2. 这句话中的consumer既包含consumer group也包含standalone consumer。也就是说，只要你向Kafka提交位移，不论使用哪种java consumer，它都是向__consumer_offsets写消息。
这类消息的key是一个三元组，格式是【groupId + topic + 分区号】，value则是要提交的位移信息，如下图所示：
位移就是待提交的位移，提交时间是提交位移时的时间戳，而过期时间则是用户指定的过期时间。由于目前consumer代码在提交位移时并没有明确指定过期间隔，故broker端默认设置过期时间为提交时间+offsets.retention.minutes参数值，即提交1天之后自动过期。Kafka会定期扫描__consumer_offsets中的位移消息并删除掉那些过期的位移数据。
上图中还有个“自定义元数据”，实际上consumer允许用户在提交位移时指定一些特殊的自定义信息。我们不对此进行详细展开，因为java consumer根本就没有使用到它。相反地，Kafka Streams利用该字段来完成某些定制任务。

* tombstone消息或Delete Mark消息

 第三类消息成为tombstone消息或delete mark消息。这类消息只出现在源码中而不暴露给用户。它和第一类消息很像，key都是二元组【版本+groupId】，唯一的区别在于这类消息的消息体是null，即空消息体。何时写入这类消息？前面说过了，Kafka会定期扫描过期位移消息并删除之。一旦某个consumer group下已没有任何active成员且所有的位移数据都已被删除时，Kafka会将该group的状态置为Dead并向__consumer__offsets对应分区写入tombstone消息，表明要彻底删除这个group的信息。简单来说，这类消息就是用于彻底删除group信息的。

* 何时写入

第一类消息是在组rebalance时写入的；第二类消息是在提交位移时写入的；第三类消息是在Kafka后台线程扫描并删除过期位移或者__consumer_offsets分区副本重分配时写入的。

* 消息留存策略

__consumer_offsets目前的留存策略是compact，__consumer_offsets会定期对消息内容进行compact操作——用户也可以同时启用两种留存策略来减少该topic所占的磁盘空间，不过要承担可能丢失位移数据的风险。

* 副本因子
__consumer_offest不受server.properties中num.partitions和default.replication.factor参数的制约。相反地，它的分区数和备份因子分别由offsets.topic.num.partitions和offsets.topic.replication.factor参数决定。这两个参数的默认值分别是50和1，表示该topic有50个分区，副本因子是1。鉴于位移和group元数据等信息都保存在该topic中，实际使用过程中很多用户都会将offsets.topic.replication.factor设置成大于1的数以增加可靠性，这是推荐的做法。不过在0.11.0.0之前，这个设置是有缺陷的：假设你设置了offsets.topic.replication.factor = 3，只要Kafka创建该topic时可用broker数<3，那么创建出来的__consumer_offsets的备份因子就是2。也就是说Kafka并没有尊重我们设置的offsets.topic.replication.factor参数。好在这个问题在0.11.0.0版本得到了解决，现在用户在使用时，一旦需要创建__consumer_offsets了Kafka一定会保证凑齐足量的broker才会开始创建，否则就抛出异常给用户。
日常使用中，另一个常见的问题是如何扩展该topic的副本因子。由于它依然是一个Kafka topic，因此我们可以调用bin/kafka-reassign-partitions.sh(bat)脚本来扩展replication factor。做法如下：
1. 构造一个json文件，如下所示，其中1，2，3表示3台broker的ID

{"version":1, "partitions":[
{"topic":"__consumer_offsets","partition":0,"replicas":[1,2,3]},
{"topic":"__consumer_offsets","partition":1,"replicas":[2,3,1]},
{"topic":"__consumer_offsets","partition":2,"replicas":[3,1,2]},
{"topic":"__consumer_offsets","partition":3,"replicas":[1,2,3]},
...
{"topic":"__consumer_offsets","partition":49,"replicas":[2,3,1]}
]}
2. 运行bin/kafka-reassign-partitions.sh --zookeeper localhost:2181 --reassignment-json-file reassign.json --execute
如果一切正常，你会发现__consumer_offsets的replication factor已然被扩展为3。

* 如何删除group信息
首先明确一点，Kafka是会删除consumer group信息的，既包括位移信息，也包括组元数据信息。对于位移信息而言，前面提到过每条位移消息都设置了过期时间。每个Kafka broker在后台会启动一个线程，定期(由offsets.retention.check.interval.ms确定，默认10分钟)扫描过期位移，并删除之。而对组元数据而言，删除它们的条件有两个：1. 这个group下不能存在active成员，即所有成员都已经退出了group；2. 这个group的所有位移信息都已经被删除了。当满足了这两个条件后，Kafka后台线程会删除group运输局信息。
好了， 我们总说删除，那么Kafka到底是怎么删除的呢——正是通过写入具有相同key的tombstone消息。我们举个例子，假设__consumer_offsets当前保存有一条位移消息，key是【testGroupid，test, 0】（三元组），value是待提交的位移信息。无论何时，只要我们向__consumer_offsets相同分区写入一条key=【testGroupid，test, 0】，value=null的消息，那么Kafka就会认为之前的那条位移信息是可以删除的了——即相当于我们向__consumer_offsets中插入了一个delete mark。
再次强调一下，向__consumer_offsets写入tombstone消息仅仅是标记它之前的具有相同key的消息是可以被删除的，但删除操作通常不会立即开始。真正的删除操作是由log cleaner的Cleaner线程来执行的。
### consume from __consumer_offset topic
``` kafka-console-consumer.sh --zookeeperzoo1.example.com:2181/kafka-cluster --topic __consumer_offsets--formatter 'kafka.coordinator.GroupMetadataManager$OffsetsMessageFormatter' --max-messages 1[my-group-name,my-topic,0]::[OffsetMetadata[481690879,NO_METADATA],CommitTime 1479708539051,ExpirationTime 1480313339051] ```

### leader election
A common approach to this tradeoff is to use a majority vote for both the commit decision and the leader election. This is not what Kafka does, but let's explore it anyway to understand the tradeoffs. Let's say we have 2_f_+1 replicas. If  _f_+1 replicas must receive a message prior to a commit being declared by the leader, and if we elect a new leader by electing the follower with the most complete log from at least  _f_+1 replicas, then, with no more than  _f_  failures, the leader is guaranteed to have all committed messages. This is because among any  _f_+1 replicas, there must be at least one replica that contains all committed messages. That replica's log will be the most complete and therefore will be selected as the new leader. There are many remaining details that each algorithm must handle (such as precisely defined what makes a log more complete, ensuring log consistency during leader failure or changing the set of servers in the replica set) but we will ignore these for now.

This majority vote approach has a very nice property: the latency is dependent on only the fastest servers. That is, if the replication factor is three, the latency is determined by the faster follower not the slower one.

There are a rich variety of algorithms in this family including ZooKeeper's  [Zab](http://web.archive.org/web/20140602093727/http://www.stanford.edu/class/cs347/reading/zab.pdf),  [Raft](https://www.usenix.org/system/files/conference/atc14/atc14-paper-ongaro.pdf), and  [Viewstamped Replication](http://pmg.csail.mit.edu/papers/vr-revisited.pdf). The most similar academic publication we are aware of to Kafka's actual implementation is  [PacificA](http://research.microsoft.com/apps/pubs/default.aspx?id=66814)  from Microsoft.

The downside of majority vote is that it doesn't take many failures to leave you with no electable leaders. To tolerate one failure requires three copies of the data, and to tolerate two failures requires five copies of the data. In our experience having only enough redundancy to tolerate a single failure is not enough for a practical system, but doing every write five times, with 5x the disk space requirements and 1/5th the throughput, is not very practical for large volume data problems. This is likely why quorum algorithms more commonly appear for shared cluster configuration such as ZooKeeper but are less common for primary data storage. For example in HDFS the namenode's high-availability feature is built on a  [majority-vote-based journal](http://blog.cloudera.com/blog/2012/10/quorum-based-journaling-in-cdh4-1), but this more expensive approach is not used for the data itself.

Kafka takes a slightly different approach to choosing its quorum set. Instead of majority vote, Kafka dynamically maintains a set of in-sync replicas (ISR) that are caught-up to the leader. Only members of this set are eligible for election as leader. A write to a Kafka partition is not considered committed until  _all_  in-sync replicas have received the write. This ISR set is persisted to ZooKeeper whenever it changes. Because of this, any replica in the ISR is eligible to be elected leader. This is an important factor for Kafka's usage model where there are many partitions and ensuring leadership balance is important. With this ISR model and  _f+1_  replicas, a Kafka topic can tolerate  _f_  failures without losing committed messages.

For most use cases we hope to handle, we think this tradeoff is a reasonable one. In practice, to tolerate  _f_  failures, both the majority vote and the ISR approach will wait for the same number of replicas to acknowledge before committing a message (e.g. to survive one failure a majority quorum needs three replicas and one acknowledgement and the ISR approach requires two replicas and one acknowledgement). The ability to commit without the slowest servers is an advantage of the majority vote approach. However, we think it is ameliorated by allowing the client to choose whether they block on the message commit or not, and the additional throughput and disk space due to the lower required replication factor is worth it.

https://kafka.apache.org/documentation/#replication

kafka 采用的 ISR 的 replication, 而不是 majority vote, 因为容忍 f 个错误, ISR 需要的 replica 只要 f+1, 而 majority vote 需要 2f+1 replica, 但是 majority vote 不需要等待最慢的机器返回, 只会等待较快的一批机器, 而 ISR 比较省存储, 而且吞吐量会更高, 因为 replica 越多写请求返回越慢. 所以一些集群配置的同步一般用 majority vote, 而大数据量的同步用 ISR. 
> This is likely why quorum algorithms more commonly appear for shared cluster configuration such as ZooKeeper but are less common for primary data storage. For example in HDFS the namenode's high-availability feature is built on a [majority-vote-based journal](http://blog.cloudera.com/blog/2012/10/quorum-based-journaling-in-cdh4-1), but this more expensive approach is not used for the data itself.


### 如何手动调整 leader 呢
手动分配 partitions, 让 prefered leader 的比例低于 10 %, 然后run prefered leader election. 

> Written with [StackEdit](https://stackedit.io/).
<!--stackedit_data:
eyJoaXN0b3J5IjpbLTE3MjgzNjE3ODBdfQ==
-->
