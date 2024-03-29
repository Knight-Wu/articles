

# Pic show the arch of lucene and ES
![image](https://github.com/Knight-Wu/articles/assets/20329409/3f7d8e48-d5f5-46ee-8f11-8e2163c878ce)
# es dynamic mapping

When Elasticsearch detects a new field in a document, it dynamically adds the field to the type mapping by default. The dynamic parameter controls this behavior.

# data tier

## 如何控制分层策略, 多久到下一层
停留在某一层的由容量和时间同时控制

https://www.elastic.co/guide/en/elasticsearch/reference/current/set-up-lifecycle-policy.html

## 分哪几层
https://www.elastic.co/guide/en/elasticsearch/reference/current/data-tiers.html

# es shard
## shard size recommendation
Aim for shards of up to 200M documents, or with sizes between 10GB and 50GB

## Reduce a cluster’s shard count
### Create indices that cover longer time periods
### Force merge during off-peak hoursedit
If you no longer write to an index, you can use the force merge API to merge smaller segments into larger ones. This can reduce shard overhead and improve search speeds.
### Shrink an existing index to fewer shards
If you no longer write to an index, you can use the shrink index API to reduce its shard count.

### Combine smaller indices
例如可以把多个小业务的 index 合在一起, 但是查询和写入都会互相影响, 可以把不写入小 index 的合在一起, 查询的 tps 很小的. 

# Getting deeper into Lucene index

NormsA norm is a factor associated with each indexed document and  stores normalization factorsused to compute the **score** relative to the query.
Term vectors
是一个document 维度的倒排索引, 由term 和他出现的频率决定, 并包括term 的position 

## Posting formats
控制着index file 如何被写入磁盘的

## doc values
Lucene index is the so-called inverted index. However, forcertain features, such as faceting or aggregations, such architecture is not the best one. 因为faceting 和aggregation operate on the document level, 而不是term level, 所以需要doc value存储着  uninvert the field 用于上面的那些操作, 


## Lucene query language (可以简单讲下查询的语法)
fuzzy search
if we would run a query, such as writer~2, both the terms writer and writers would be considered a match
For example, let’s take the followingquery:title:"mastering Elasticsearch"It would match the document with the title field containing mastering Elasticsearch,but not mastering book Elasticsearch. However, if we would run a query, such astitle:"mastering Elasticsearch"~2, it would result in both example documentsmatched.

We can also use boosting to increase our term importance by using the ^ character andproviding a float number. Boosts lower than one would result in decreasing the documentimportance. Boosts higher than one will result in increasing the importance. The defaultboost value is 1

查询特殊字符:
In case you want to search for one of the special characters (which are +, -, &&, ||, !, (, ),{ }, [ ], ^, ", ~, *, ?, :, \, /), you need to escape it with the use of the backslash (\)character
# es basic concepts
* index 
可以理解为database
与lucene 的关系,  Elasticsearch uses Apache Lucene library to write and read the data from the index. What you should remember is that a single Elasticsearch index may be built of more than a single Apache Lucene index—by using shards.

* type 
This allows us to store variousdocument types in one index and have different mappings for different document types, 类 table
* Mapping
控制着text 如何被解析成token, 例如 field 的type, (field 由key 和val组成 df) 
* node 
分为data node, master, 和tribe node(用于多个cluster 的协调, 使得用起来好像在一个 cluster )
* Shard（分片） 
一个Shard就是一个Lucene实例，是一个完整的搜索引擎。一个索引可以只包含一个Shard，只是一般情况下会用多个分片，可以拆分索引到不同的节点上，分担索引压力。

* lucene segment 
elasticsearch中的每个shard 是一个 lunene index, 包含多个segment，每个 seg 和他的索引都是不可变的, 理念与 LSM 类似, 只能 merge, 每一个segment都有其对应的一系列索引；在查询的时，会把所有的segment查询结果汇总归并后最为最终的分片查询结果返回；
* Replica
每一个 shard 都有多个副本3.

# Apache Lucene scoring
A score is a factor that describes how well the document matched the query. 等于说要提高query 的准确性, 准确匹配到我们想要的结果的话, 就需要了解score 的计算原理
scoring mechanism: the TF/IDF(term frequency/inverse document frequency) algorithm
* In order to calculate the score property for adocument, multiple factors are taken into account, which are as follows( ignore)
* What you should be aware of is what matters when it comes to document score. Basically,there are a few rules
1. term 越稀有, doc 的 score 越高
2.  doc 的 fields 越少, score 越高
3. 设置的权重, (索引和搜索时设置的), 越大, score 越高

可以讲下这个例子, An example . Till now we’ve seen how scoring works. Now we would like to show you a simpleexample of how the scoring works in real life. To do this, we will create a new indexcalled scoring.
## 问题
 * 一个shard 有多个segment 组成，如何搜索
 * 写入要写几个副本才成功，读呢，有可能的异常情况。
 * docId 不是默认的路由规则吗，如何保证跟其他shard 不会重复的。

# es 版本升级
## overview
https://www.elastic.co/guide/en/elasticsearch/reference/7.17/setup-upgrade.html
* 直接降版本是不支持的, 可以先保留一个老版本的snapshot
* 旧版本的index 是可以在新版本中被读取的

## 滚动升级
将master 和 非master 节点分成两组升级, 先升级非master 节点, 再升级master 节点, 新版本的非master 节点可以加入旧
master 的集群, 反之不能
1. 看deprecation log，看是否在新版本中有功能不支持，需要代码改动
2. 在测试环境先完整升级演练一遍。 
3. 可以用snapshot 备份数据。
4. 防止重启过程副本在其他节点重建
5. 停止节点
6. 升级节点文件
7，启动
8，恢复第四部的配置
9. cat/health 看集群是否变绿
10. 在其他节点重复升级

* 如果是整个集群的重启，先启动master 节点，直到选出新的leader，再重启数据节点。

## deprecation log 
https://www.elastic.co/guide/en/elasticsearch/reference/7.17/logging.html#deprecation-logging

如果是大版本的升级, major 版本, 则可以看deprecation log 会提示在以后的版本中哪些功能会不支持或者改动

# es 选主， es 会不会出现脑裂
es 在 7.X 版本之前有可能会出现脑裂的情况，因为minimum_master_nodes 配置不正确，默认值是1，推荐配置的是有master 资格的节点数/2 +1,例如 3/2 +1 = 2 , 如果没有按照推荐配置，那么如果有三个master 资格节点，
配置默认值1，那么可能会出现三个master 的情况，因为只需要投一票就可以了。
7.X 版本之后基于raft 实现了一套选举方法，有些不同于raft。
## ES实现的Raft算法选主流程

ES 实现的 Raft 中，选举流程与标准的有很多区别：

1. 初始为 Candidate状态
2. 执行 PreVote 流程，并拿到 maxTermSeen
3. 准备 RequestVote 请求（StartJoinRequest），基于maxTermSeen，将请求中的 term 加1（尚未增加节点当前 term）
4. 并行发送 RequestVote，异步处理。目标节点列表中包括本节点。

ES 实现中，候选人不先投自己，而是直接并行发起 RequestVote，这相当于候选人有投票给其他候选人的机会。这样的好处是可以在一定程度上避免3个节点同时成为候选人时，都投自己，无法成功选主的情况。

ES 不限制每个节点在某个 term 上只能投一票，节点可以投多票，这样会产生选出多个主的情况：
![image](https://user-images.githubusercontent.com/20329409/220305744-9fbf106f-bd6f-4d62-864a-2d17f42aa187.png)



Node2被选为主：收到的投票为：Node2,Node3
Node3被选为主：收到的投票为：Node3,Node1

对于这种情况，ES 的处理是让最后当选的 Leader 成功。作为 Leader，如果收到 RequestVote请求，他会无条件退出 Leader 状态。在本例中，Node2先被选为主，随后他收到 Node3的 RequestVote 请求，那么他退出 Leader 状态，切换为CANDIDATE，并同意向发起 RequestVote候选人投票。因此最终 Node3成功当选为 Leader。

## 如何维护选举节点列表

现在的做法不再记录“quorum” 的具体数值，取而代之的是记录一个节点列表，这个列表中保存所有具备 master 资格的节点，称为 VotingConfiguration，他会持久化到集群状态中
默认情况下，ES 自动维护VotingConfiguration，有新节点加入的时候比较好办，但是当有节点离开的时候，他可能是暂时的重启，也可能是永久下线。你也可以人工维护 VotingConfiguration，配置项为：cluster.auto_shrink_voting_configuration ，当你选择人工维护时，有节点永久下线，需要通过 voting exclusions API 将节点排除出去。如果使用默认的自动维护VotingConfiguration，也可以使用 voting exclusions API 来排除节点，例如一次性下线半数以上的节点。

如果在维护VotingConfiguration时发现节点数量为偶数，ES 会将其中一个排除在外，保证 VotingConfiguration是奇数。因为当是偶数的情况下，网络分区将集群划分为大小相等的两部分，那么两个子集群都无法达到“多数”的条件。

# dynamic mapping
es 能够识别新加字段，以及自动推断新加字段的索引，但是会触发一个mapping task，由master 同步到所有节点，如果很频繁会很耗性能。
# 写入
![image](https://user-images.githubusercontent.com/20329409/220523703-d451d112-030f-4b8f-b291-1d4f986bb75c.png)

![image](https://user-images.githubusercontent.com/20329409/220523717-2faf81a9-6d00-4208-8400-d3846aa00ba0.png)
1. 首先随机挑选一个节点作为协调节点转发请求，然后primary shard 所在节点写入，之后再将数据同步到副本，协调节点等primary shard 和secondary shard 都执行完成之后再返回成功。
2. 在某个节点写入的时候，先写memo table，再写log，memo table 会隔默认一秒refresh 成segment 才构建lucene 索引，才可以搜索；在内存中还没有refresh 之前可以根据docid 来搜索；直到segment 持久化到磁盘之后才会把log 清空。
3. segment 会定期合并，整个写入类似 LSM 
4. 删除和更新都是写操作，但是由于 Elasticsearch 中的文档是不可变的，因此不能被删除或者改动以展示其变更；所以 ES 利用 .del 文件 标记文档是否被删除，磁盘上的每个段都有一个相应的.del 文件
（1）如果是删除操作，文档其实并没有真的被删除，而是在 .del 文件中被标记为 deleted 状态。该文档依然能匹配查询，但是会在结果中被过滤掉。
（2）如果是更新操作，就是将旧的 doc 标识为 deleted 状态，然后创建一个新的 doc
# 搜索
query then fetch
1. query：先由协调节点广播查询请求到所有shard，primary shard 和 replica shard 都可以接受请求，然后查询索引，得到docId
2. 再由docId 进行路由，请求转发到shard，进行查询，最后结果聚合到协调节点再返回。
# es 数据一致性
总而言之是个最终一致性的系统，读可能读到未写完的数据（因为先写primary local 再写replica），可能读到旧的数据（primary 在与replica 和master 交互前不能保证是当前的primary），可能之前读到同一个doc 新的数据又读
到旧的数据（先从primary 读到但是replica 写失败，master还没把replica 从in-sync replica去掉的时候，又从replica 读到了旧的数据）

主备同步的模型，由primary 负责写入，然后再复制到其他replica，参考了 PacificA paper of Microsoft Research，每一个shard 和他的副本称作 replication group，
## 写入

主副本负责接受协调节点的写入请求，并做校验，然后写入本地，再把请求并行转发到其他replica，wait_for_active_shard 配置控制等待几个副本写入成功才返回，如果达不到这个配置的参数会一直重试等待，所以响应时间会受最慢的副本影响。但是好处是in-sync 副本都是一致的，读任何一个副本就可以了，提升了读的性能。

## 写入过程中错误处理

1. 如果是primary shard 挂了，会等一分钟，如果没响应就写到其他replica，
2. 如果是primary 成功，但是写入replica 失败，不管是返回失败或者是没有响应等任何非正常的情况，primary shard 都会报告master ，将失败shard 从in-sync replicas 中移除，直到master 响应之后才会返回客户端响应，并且master 会同时新建一个新的replica。那么就可以保证in-sync replicas 都是和primary 数据一致的。
3. primary 在同步请求给replica 过程中也会校验自己是否还是primary，因为可能因为网络分区的原因primary 不知道已经有新的primary 了，如果不是primary 了，就会把请求转发到真正的primary。与此同时为了避免没有写入，导致没有跟其他节点通信不知道自己是否是真正的primary，primary 还会每隔一秒心跳master 一次检查自己是否是primary。

## 搜索

1. 因为数据先写入primary 本地，有可能还没写入返回成功就被搜索到（类似read uncommitted）
2. 只要 N+1 个副本就能忍受 N 个副本故障。
3. 牺牲了写性能换取读性能：in-sync replicas 可以认为都是一致的，那么读取只要读任何一个就可以了，代价就是写入要写多个，并且最慢的那个返回之后才能返回客户端成功。
4. es 也会发生dirty read，因为在与replica 和master 通信前是无法知道是不是真正的primary shard，那么就有可能读到旧数据。

## 更新doc
对于更新操作：可以通过版本号使用乐观并发控制，以确保新版本不会被旧版本覆盖
每个文档都有一个_version 版本号，这个版本号在文档被改变时加一。类似cas，只有当当前版本和期待版本一致，才能成功，其实最后写入的时候也是要加锁的。
## ES在高并发下如何保证读写一致性，这个出处有待考证？
（2）对于写操作，一致性级别支持 quorum/one/all，默认为 quorum，即只有当大多数副本可用时才允许写操作。但即使大多数可用，也可能存在因为网络等原因导致写入副本失败，这样该副本被认为故障，副本将会在一个不同的节点上重建。
</br>
one：写操作只要有一个primary shard是active活跃可用的，就可以执行
</br>
all：写操作必须所有的primary shard和replica shard都是活跃可用的，才可以执行
</br>
quorum：默认值，要求ES中大部分的副本是活跃可用的，才可以执行写操作
（3）对于读操作，正常情况是replica 和primary 是保持同步的，但是如果replica 写入异常，会把错误存储到元数据中 cluster state，搜索时跳过这个replica，但在此之前有可能读到旧的数据，但是鉴于es 是个近实时搜索的系统，数据必须要refresh 才能搜到，也是可以接受的。
可以设置 replication 为 sync(默认)，数据要同时写入主和副分片之后才能被搜索到，避免了主搜索到挂掉之后副本搜索不到的不一致的情况，但是如果副副本写入失败，也还是会搜索到旧的数据；如果设置replication 为 async 时，也可以通过设置搜索请求参数 _preference 为 primary 来查询主分片。
## ClusterState commit 之后如何保证不回退
这里是为了解决这样一个问题，我们知道，新的Meta一旦在某个节点上commit，那么这个节点就会执行相应的操作，比如删除某个Shard等，这样的操作是不可回退的。而假如此时Master节点挂掉了，新产生的Master一定要在新的Meta上进行更改，不能出现回退，否则就会出现Meta回退了但是操作无法回退的情况。本质上就是Meta更新没有保证一致性。

早期的ES版本没有解决这个问题，后来引入了两阶段提交的方式(Add two phased commit to Cluster State publishing)。所谓的两阶段提交，是把Master发布ClusterState分成两步，第一步是向所有节点send最新的ClusterState，当有超过半数的master节点返回ack时，再发送commit请求，要求节点commit接收到的ClusterState。如果没有超过半数的节点返回ack，那么认为本次发布失败，同时退出master状态，执行rejoin重新加入集群。

## 问题

但是没法解决这个问题：
如果master在commit阶段，只commit了少数几个节点就出现了网络分区，将master与这几个少数节点分在了一起，其他节点可以互相访问。此时其他节点构成多数派，会选举出新的master，由于这部分节点中没有任何节点commit了新的ClusterState，所以新的master仍会使用更新前的ClusterState，造成Meta不一致。

貌似raft 也无法解决？如果集群五个节点，leader 只成功复制到了一个节点A，然后leader 和节点A 就和其他三个节点通信不了了，然后三个节点选出了新的leader，然后这次更新就丢失了，除非有个client 一直重试遍历所有可达节点。这个client 可以是内部提交请求的线程，一直轮训重试，最后还是可能结果不符合预期，这个可以由用户发现并自行重试。

# 某些字段不需要直接查询, 从而关闭 index, 减少空间使用



```
# template 指定的是 indexer 的名字, es-host 指定 ESserver:port
# mapping 可以从chrome plugin: elasticSearch 获取, properties 为存的字段, 
# 关闭索引 index:false

curl -XPUT -H 'Content-Type: application/json' ES-host/_template/template_order_etl?pretty -d '{
  "template": "order_realtime_*",
  "settings": {
    "number_of_shards": 10,
    "number_of_replicas": 2
  },
   "mappings": {
			"table": {
			"properties": {
			"itemid": {
			"type": "long"
			},
			"hour": {
			"type": "long"
			},
			"l2_cat": {
			"type": "long"
			},
			"orderid": {
			"type": "long"
			},
			"l1_cat": {
			"type": "long"
			},
			"l3_cat": {
			"type": "long"
			},
			"shopid": {
			"type": "long"
			},
			"units": {
			"type": "long",
			"index": false
			},
			"buyer_userid": {
			"type": "long"
			},
			"event": {
			"type": "text",
			"fields": {
			"keyword": {
			"ignore_above": 256,
			"type": "keyword"
			}
			}
			},
			"sales": {
			"type": "long",
            "index": false
			}
		}
	}
}
}'
```

# curator 删除过期的 indexer
```
$ cat curator-cfg.yml

client:

hosts:

- 10.65.229.119

port: 9201

$ cat curator-action-delete.yml

actions:

1:

action: delete_indices

description: >-

Delete old indexes.

options:

ignore_empty_list: True

timeout_override:

continue_if_exception: False

disable_action: False

filters:

- filtertype: pattern

kind: prefix

value: order_realtime_ 
(指定 index 的名字, 后面是 %Y%m%d 格式的时间字符串)
exclude:

- filtertype: age

source: name

direction: older

timestring: '%Y%m%d'

unit: days

unit_count: 2
(删除今天的两天之前的 indexer)
$ curator  --config curator-cfg.yml curator-action-delete.yml 

```

# 常用命令
## template
curl -X DELETE "ip:port/_template/templateName"
curl -X GET "ip:port/_template/templateName"
## es version
 curl -XGET "ip:port"



## index 
curl -X PUT "ip:port/indexName"
curl -X DELETE "ip:port/indexName"



> Written with [StackEdit](https://stackedit.io/).
<!--stackedit_data:
eyJoaXN0b3J5IjpbMTk0NzA3MTgwM119
-->
