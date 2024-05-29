* 论文地址
https://raft.github.io/raft.pdf

![image](https://user-images.githubusercontent.com/20329409/218662231-f6e1591f-7baa-4bac-8d1b-faecedffe7a3.png)


# raft 用在什么地方 


 多台机器需要达成一致和共识的地方, 向一个集群写入数据, 一个 leader, 多个 follower, 如何保证数据以整个集群为整体看来是一致的, 而不是以客户端的角度看来是一致的, 写入成功的数据后续肯定能读到, 哪怕 leader 发生了切换. 但是返回客户端失败也有可能实际写入成功. 例如五台机器, 只复制了 leader 和两个 follower, 如果返回给 client 的包丢失了, client 得到失败的结果, 此时 leader 挂了, 那么新的 leader 选出来之后数据会比客户端认为的要新, 这个情况也是有的, 但是整个集群数据也是一致的. 
# 如何发起选举

![image](https://user-images.githubusercontent.com/20329409/220304308-ef1e06b4-ad0e-4c5e-ac63-4d5d1d5f7a36.png)

leader 会周期性的发送appendEntity 请求给follower，是心跳也是日志append rpc，如果follower 一段时间没有收到，就转为candidate 状态，任期号加一，先给自己投一票，并发送requestVoteRpc 给其他follower，
requestVoteRpc 会携带任期号，和最新日志的序列号和日志所属的任期号，如果follower 发现 candidate 的任期号以及日志的序列号和任期号大于等于自己的，则投支持票。如果 candidate 收到大多数投票就变为leader。
其次，如果在等待投票过程中发现有其他leader 的append rpc，并且任期号大于等于自己的，则自己变为follower 状态。
如果在规定时间内没有选出新的leader，则超时，任期号加一，再次投一次。



# raft 如何避免不脑裂(同时有多个leader ?)

某个节点选举成功要得到大多数节点的投票，且每个节点在每个单独任期号能只能投一次支持票，那么只有一个节点能选举胜出，只要候选节点的任期号和日志都是最新的就可以投支持票。如果发生网络隔离, 旧的leader 失联，已经选出新的leader ，
那么旧的leader 恢复后与其他节点通信就发现自己的任期号已经比较小，那么就会变为follower, 向新的leader 请求数据。所以不会发生脑裂

# 如何保证线性读
参考：
https://blog.csdn.net/weixin_43705457/article/details/120572291
https://zhuanlan.zhihu.com/p/25367435

如果要保证线性读，一个简单的例子就是在 t1 的时间我们写入了一个值，那么在 t1 之后，我们的读一定能读到这个值，不可能读到 t1 之前的值。
有两种实现方式论文里推荐的，都是需要通过leader 来读
1 是每次读之前leader 跟多数节点通信一次确保自己是leader
2 是lease 机制，每次和多数节点心跳成功一次就更新一次leader 的lease，例如 tikv election timeout 是10 s， lease time 是9s，心跳间隔可能是几百毫秒，只要lease 没超时前就认为当前leader 是有效的，不会接受其他的
选举投票，保证了当前leader 的唯一性，lease 没超时那么从leader 读就是线性一致的。
 # 常见的分布式可能导致不一致性的问题如何解决, 
 ## 写入返回客户端成功之后, leader 挂掉  
 因为是大多数节点返回成功才最终返回 client 成功, 而且选择新 leader 选择的是有最多最新数据的节点, 所以依然能保证一致性.
 ## 返回客户端成功之前, leader 挂掉, 那么客户端会收到失败, 而可以继续尝试. 如何避免重复请求呢, 保证满足串行化语义呢, 给每个 client 分配一个 client id, 并且让 client 每个请求申请一个 uuid, leader 检测到重复就丢弃. 
 ## 如果只是单个 follower 失去去 leader 的联系
 转为 candidate, 并 increase own term , 再发送 requestVote 给其他节点, 但是只要其他节点与 leader 的append rpc 没有超时, 则不会投票给这个 candidate, candidate 转变为follower, 不管candidate 的任期号是否更大, 否则就会因为单个follower 失联导致经常发起选举, 更加证明这是一个强 leader 的系统. 
 ## 如何保证 client 每次都读取到最新的数据呢
 假设请求刚转发到旧 leader, 就发生了新旧 leader 的切换, 旧 leader 数据已经是旧的了, 例如新的客户端请求, 由于 leader 地址是缓存的有时效的, 可能访问到旧 leader, 那如何避免呢, 可以让 leader 要确保返回最新的数据之前联系一下大多数节点, 保证自己是当前的 leader; 
 或者可以维持一个和多数节点通过 append rpc 更新的一个 lease 租约 , 假设选举超时是 1s, leader 广播到所有节点并接受返回是 20-50ms, 那么可以设置 lease 超时时间为: 假设旧leader 与大多数节点通信之后是第 0s,  选出新 leader 是第 1s, 那么lease 就要小于 1s , 如果超过 1s 还没与大多数节点做过一次 append, 则认为 lease 超时

# 如何避免同时成为candidate

每个follower 等待心跳超时的值是一个随机值。

# 怎么选择最新, 最多数据的候选者作为新的 leader 呢

与旧 leader 心跳失败的 follower 都会变为 candidate, 然后发送 requestVote rpc 到其他 follower, 获得大多数票的 cadidate 变为新的 leader , 因为要获得大多数投票, 所以日志最新的节点肯定在多数节点其中, 请求中带上candidate 的 term, 最新的日志的 term 和 index , 其他接收请求的 follower 如果自己的日志比 candidate 的新, 或者 term 更大, 则不投票, 否则投票; 日志比较的原则是，如果本地的最后一条log entry的term id更大，则更加新，如果term id一样大，则日志 index更大的更新。

# raft 是否是强一致的

是强一致的，如何保证强一致呢？
>论文：Our goal for Raft is to implement linearizable seman-tics (each operation appears to execute instantaneously,exactly once, at some point between its invocation andits response)

首先客户端会把请求发送给leader，如果不是leader，会把请求拒绝，并告诉正确的leader 地址。找到了leader 如何保证能返回提交过后的数据呢，一是因为leader 都是从最新日志的副本中选出的，有所有已经提交的数据
>论文： The Leader CompletenessProperty guarantees that a leader has all committed en-tries, but at the start of its term, it may not know whichthose are. To find out, it needs to commit an entry fromits term. Raft handles this by having each leader com-mit a blankno-opentry into the log at the start of itsterm. 
为什么要提交一个blankno-opentry ? 没懂

其次如何保证leader 在接受读的时候没有被其他leader 接管呢，需要和大多数节点心跳一次看是否有新的leader 或者使用lease 租约机制。

# 如何保证客户端请求只执行了一次呢？

如果leader 在commit 之后，返回客户端响应之前挂掉，那么客户端会一直尝试请求，新的leader 因为包含了已经commit 的结果，记录了请求id，那么重试的请求因为请求id 已经重复了，就不会重新执行，而是直接返回成功。

# 日志一致性的保证

1. 如果不同日志中的两个条目有着相同的索引和任期号，则它们所存储的命令是相同的（原因：leader 最多在一个任期里的一个日志索引位置创建一条日志条目，日志条目在日志的位置从来不会改变）。


2. 如果不同日志中的两个条目有着相同的索引和任期号，则它们之前的所有条目都是完全一样的（原因：每次 RPC 发送附加日志时，leader 会把这条日志的前面的日志的下标和任期号一起发送给 follower，如果 follower 发现和自己的日志不匹配，那么就拒绝接受这条日志，这个称之为一致性检查）。

如果leader 发现和follower 日志不匹配，会强制follower 的日志修改成与leader 同步，具体步骤是不断发送AppendEntity rpc，直到找到某条日志的index，和term 一致，那么删除冲突的日志，开始同步。

# 定期备份，snapshot
![image](https://user-images.githubusercontent.com/20329409/218662381-63a55eb5-d057-42a6-b40d-6d82dcfc4db0.png)
![image](https://user-images.githubusercontent.com/20329409/218662562-7ab8790b-326e-497c-9416-6c2071826e84.png)

目的是当新的节点加入的时候，需要同步的日志很多很慢，所以定期当日志条数达到某个规模的时候，每个节点把自身的状态，以及last log index，last log term 持久化到磁盘。
做一次snapshot可能耗时过长，会影响正常日志同步。可以通过使用copy-on-write技术避免snapshot过程影响正常日志同步。

# Raft网络分区下的数据一致性怎么解决？
![image](https://user-images.githubusercontent.com/20329409/218666469-7a25fe4e-feda-415e-8dcb-3ad4842e81b6.png)

例如某个follower 和leader 失联，然后任期号增加，成为新的leader，然后同时有新旧两个leader 存在，其实没关系，哪个leader 能和大多数节点联系上那么他就是leader，并且每个节点在当前任期号的情况下只能投一次票，就保证了只有一个leader 是真正的，然后后续旧的leader 恢复过来，会以新的leader 的数据覆盖他。

## 如果某个follower 偶尔失联，并且有最新的日志，会导致反复选举吗？ 

不会的，因为其他follower 在超时时间内仍然收到leader 的心跳，确保当前leader 存在，那么这个follower 的投票会被否决，

>The third issue is that removed servers (those not inCnew) can disrupt the cluster. These servers will not re-ceive heartbeats, so they will time out and start new elec-tions. They will then send RequestVote RPCs with newterm numbers, and this will cause the current leader torevert to follower state. A new leader will eventually beelected, but the removed servers will time out again andthe process will repeat, resulting in poor availability.To prevent this problem, servers disregard RequestVoteRPCs when they believe a current leader exists. Specif-ically, if a server receives a RequestVote RPC withinthe minimum election timeout of hearing from a cur-rent leader, it does not update its term or grant its vote.This does not affect normal elections, where each serverwaits at least a minimum election timeout before startingan election. However, it helps avoid disruptions from re-moved servers: if a leader is able to get heartbeats to itscluster, then it will not be deposed by larger term num-bers.


# 成员变更



Raft两阶段成员变更过程如下：

1. Leader收到成员变更请求。
2. Leader在本地生成一个新的log entry，其内容是Cold∪Cnew，代表当前时刻新旧成员配置共存，写入本地日志，同时将该log entry复制至Cold∪Cnew中的所有副本。在此之后新的日志同步需要保证得到Cold和Cnew两个多数派的确认；
3.Follower收到Cold∪Cnew的log entry后更新本地日志，并且此时就以该配置作为自己的成员配置；
4.如果Cold和Cnew中的两个多数派确认了Cold U Cnew这条日志，Leader就提交这条log entry并切换到Cnew；
5.接下来Leader生成一条新的log entry，其内容是新成员配置Cnew，同样将该log entry写入本地日志，同时复制到Follower上；
6. Follower收到新成员配置Cnew后，将其写入日志，并且从此刻起，就以该配置作为自己的成员配置，并且如果发现自己不在Cnew这个成员配置中会自动退出；
7. Leader收到Cnew的多数派确认后，表示成员变更成功，后续的日志只要得到Cnew多数派确认即可。Leader给客户端回复成员变更执行成功。

异常分析：
1.如果Leader的Cold U Cnew尚未推送到Follower，Leader就挂了，此后选出的新Leader并不包含这条日志，此时新Leader依然使用Cold作为自己的成员配置。
2.如果Leader的Cold U Cnew推送到大部分的Follower后就挂了，此后选出的新Leader可能是Cold也可能是Cnew中的某个Follower。
3.如果Leader在推送Cnew配置的过程中挂了，那么同样，新选出来的Leader可能是Cold也可能是Cnew中的某一个，此后客户端继续执行一次改变配置的命令即可。
4.如果大多数的Follower确认了Cnew这个消息后，那么接下来即使Leader挂了，新选出来的Leader肯定位于Cnew中。
5.新加入的节点在catch up 之前不会被认为是在多数投票者里面，避免了刚加入忙于catch up 而无法投票的情况。

两阶段成员变更比较通用且容易理解，但是实现比较复杂，同时两阶段的变更协议也会在一定程度上影响变更过程中的服务可用性，因此我们期望增强成员变更的限制，以简化操作流程。
两阶段成员变更，之所以分为两个阶段，是因为对Cold与Cnew的关系没有做任何假设，为了避免Cold和Cnew各自形成不相交的多数派选出两个Leader，才引入了两阶段方案。
如果增强成员变更的限制，假设Cold与Cnew任意的多数派交集不为空，这两个成员配置就无法各自形成多数派，那么成员变更方案就可能简化为一阶段。
那么如何限制Cold与Cnew，使之任意的多数派交集不为空呢？方法就是每次成员变更只允许增加或删除一个成员。
可从数学上严格证明，只要每次只允许增加或删除一个成员，Cold与Cnew不可能形成两个不相交的多数派。

一阶段成员变更（还需要再理解）：
成员变更限制每次只能增加或删除一个成员（如果要变更多个成员，连续变更多次）。
成员变更由Leader发起，Cnew得到多数派确认后，返回客户端成员变更成功。
一次成员变更成功前不允许开始下一次成员变更，因此新任Leader在开始提供服务前要将自己本地保存的最新成员配置重新投票形成多数派确认。
Leader只要开始同步新成员配置，即可开始使用新的成员配置进行日志同步。
