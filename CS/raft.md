* 论文地址
https://raft.github.io/raft.pdf

![image](https://user-images.githubusercontent.com/20329409/218662231-f6e1591f-7baa-4bac-8d1b-faecedffe7a3.png)


* raft 用在什么地方 

多台机器需要达成一致和共识的地方, 向一个集群写入数据, 一个 leader, 多个 follower, 如何保证数据在业务看来是一致的, 写入成功的数据后续肯定能读到, 哪怕 leader 发生了切换. 

* 如何发起选举

leader 会周期性的发送appendEntity 请求给follower，是心跳也是日志append rpc，如果follower 一段时间没有收到，就转为candidate 状态，任期号加一，先给自己投一票，并发送requestVoteRpc 给其他follower，
requestVoteRpc 会携带任期号，和最新日志的序列号和日志所属的任期号，如果接受者发现候选者的任期号以及日志的序列号和任期号大于等于自己的，则投支持票。如果收到大多数投票就变为leader。
其次，如果在等待投票过程中发现有其他leader 的append rpc，并且任期号大于等于自己的，则自己变为follower 状态。
最后如果一次投票时间内没有选出新的leader，则超过，任期号加一，再次投一次。

* 如何避免同时成为candidate

每个follower 等待心跳超时的值是一个随机值。

* 常见的分布式故障如何解决,

写入返回客户端成功之后, leader 挂掉: 因为是大多数节点返回成功才最终返回 client 成功, 而且选择新 leader 选择的是有最多数据的节点, 所以依然能保证一致性.

* 怎么选择最新, 最多数据的候选者作为新的 leader 呢

与旧 leader 心跳失败的 follower 都会变为 candidate, 然后发送 requestVote rpc 到其他 follower, 获得大多数票的 cadidate 变为新的 leader , 请求中带上candidate 的 term, 最新的日志的 term 和 index , 其他接收请求的 follower 如果自己的日志比 candidate 的新, 或者 term 更大, 则不投票, 否则投票; 日志比较的原则是，如果本地的最后一条log entry的term id更大，则更加新，如果term id一样大，则日志 index更大的更新。

* raft 是否是强一致的

是强一致的，如何保证强一致呢？
>论文：Our goal for Raft is to implement linearizable seman-tics (each operation appears to execute instantaneously,exactly once, at some point between its invocation andits response)

首先客户端会把请求发送给leader，如果不是leader，会把请求拒绝，并告诉正确的leader 地址。找到了leader 如何保证能返回提交过后的数据呢，一是因为leader 都是从最新日志的副本中选出的，有所有已经提交的数据
>论文： The Leader CompletenessProperty guarantees that a leader has all committed en-tries, but at the start of its term, it may not know whichthose are. To find out, it needs to commit an entry fromits term. Raft handles this by having each leader com-mit a blankno-opentry into the log at the start of itsterm. 
为什么要提交一个blankno-opentry ? 没懂

其次如何保证leader 在接受读的时候没有被其他leader 接管呢，需要和大多数节点心跳一次看是否有新的leader，

* 如何保证客户端请求只执行了一次呢？

如果leader 在commit 之后，返回客户端响应之前挂掉，那么客户端会一直尝试请求，新的leader 因为包含了已经commit 的结果，记录了请求id，那么重试的请求因为请求id 已经重复了，就不会重新执行，而是直接返回成功。

* 日志一致性的保证

1. 如果不同日志中的两个条目有着相同的索引和任期号，则它们所存储的命令是相同的（原因：leader 最多在一个任期里的一个日志索引位置创建一条日志条目，日志条目在日志的位置从来不会改变）。


2. 如果不同日志中的两个条目有着相同的索引和任期号，则它们之前的所有条目都是完全一样的（原因：每次 RPC 发送附加日志时，leader 会把这条日志的前面的日志的下标和任期号一起发送给 follower，如果 follower 发现和自己的日志不匹配，那么就拒绝接受这条日志，这个称之为一致性检查）。

如果leader 发现和follower 日志不匹配，会强制follower 的日志修改成与leader 同步，具体步骤是不断发送AppendEntity rpc，直到找到某条日志的index，和term 一致，那么删除冲突的日志，开始同步。

* 定期备份，snapshot
![image](https://user-images.githubusercontent.com/20329409/218662381-63a55eb5-d057-42a6-b40d-6d82dcfc4db0.png)
![image](https://user-images.githubusercontent.com/20329409/218662562-7ab8790b-326e-497c-9416-6c2071826e84.png)

目的是当新的节点加入的时候，需要同步的日志很多很慢，所以定期当日志条数达到某个规模的时候，每个节点把自身的状态，以及last log index，last log term 持久化到磁盘。
做一次snapshot可能耗时过长，会影响正常日志同步。可以通过使用copy-on-write技术避免snapshot过程影响正常日志同步。

* Raft网络分区下的数据一致性怎么解决？
![image](https://user-images.githubusercontent.com/20329409/218666469-7a25fe4e-feda-415e-8dcb-3ad4842e81b6.png)

例如某个follower 和leader 失联，然后任期号增加，成为新的leader，然后同时有新旧两个leader 存在，其实没关系，哪个leader 能和大多数节点联系上那么他就是leader，并且每个节点在当前任期号的情况下只能投一次票，就保证了只有一个leader 是真正的，然后后续旧的leader 恢复过来，会以新的leader 的数据覆盖他。
