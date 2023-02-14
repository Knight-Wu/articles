* 论文地址
https://raft.github.io/raft.pdf


* raft 用在什么地方 

多台机器需要达成一致和共识的地方, 向一个集群写入数据, 一个 leader, 多个 follower, 如何保证数据在业务看来是一致的, 写入成功的数据后续肯定能读到, 哪怕 leader 发生了切换. 

* 如何发起选举

leader 会周期性的发送

* 常见的分布式故障如何解决,

写入返回客户端成功之后, leader 挂掉: 因为是大多数节点返回成功才最终返回 client 成功, 而且选择新 leader 选择的是有最多数据的节点, 所以依然能保证一致性.

* 怎么选择最新, 最多数据的候选者作为新的 leader 呢

与旧 leader 心跳失败的 follower 都会变为 candidate, 然后发送 requestVote rpc 到其他 follower, 获得大多数票的 cadidate 变为新的 leader , 请求中带上candidate 的 term, 最新的日志的 term 和 index , 其他接收请求的 follower 如果自己的日志比 candidate 的新, 或者 term 更大, 则不投票, 否则投票; 得到大多数投票的 follower 或者接收到新的 leader 的 append 请求(只要新的 leader 的 term id 大于自己的)就自动变为 follower.

* raft 是否是强一致的

是强一致的，如何保证强一致呢？
>论文：Our goal for Raft is to implement linearizable seman-tics (each operation appears to execute instantaneously,exactly once, at some point between its invocation andits response)

首先客户端会把请求发送给leader，如果不是leader，会把请求拒绝，并告诉正确的leader 地址。找到了leader 如何保证能返回提交过后的数据呢，一是因为leader 都是从最新日志的副本中选出的，有所有已经提交的数据
>论文： The Leader CompletenessProperty guarantees that a leader has all committed en-tries, but at the start of its term, it may not know whichthose are. To find out, it needs to commit an entry fromits term. Raft handles this by having each leader com-mit a blankno-opentry into the log at the start of itsterm. 
为什么要提交一个blankno-opentry ? 没懂

其次如何保证leader 在接受读的时候没有被其他leader 接管呢，需要和大多数节点心跳一次看是否有新的leader，

* 如何保证客户端请求只执行了一次呢？

如果leader 在commit 之后，返回客户端响应之前挂掉，那么客户端会一直尝试请求，新的leader 因为包含了已经commit 的结果，记录了请求id，那么重试的请求因为请求id 已经重复了，就不会重新执行，而是直接返回成功。