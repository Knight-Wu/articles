## 问题
 * 一个shard 有多个segment 组成，如何搜索
### es 选主， es 会不会出现脑裂
es 在 7.X 版本之前有可能会出现脑裂的情况，因为minimum_master_nodes 配置不正确，默认值是1，推荐配置的是有master 资格的节点数/2 +1,例如 3/2 +1 = 2 , 如果没有按照推荐配置，那么如果有三个master 资格节点，
配置默认值1，那么可能会出现三个master 的情况，因为只需要投一票就可以了。
7.X 版本之后基于raft 实现了一套选举方法，有些不同于raft。
* ES实现的Raft算法选主流程

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

* 如何维护选举节点列表

现在的做法不再记录“quorum” 的具体数值，取而代之的是记录一个节点列表，这个列表中保存所有具备 master 资格的节点，称为 VotingConfiguration，他会持久化到集群状态中
默认情况下，ES 自动维护VotingConfiguration，有新节点加入的时候比较好办，但是当有节点离开的时候，他可能是暂时的重启，也可能是永久下线。你也可以人工维护 VotingConfiguration，配置项为：cluster.auto_shrink_voting_configuration ，当你选择人工维护时，有节点永久下线，需要通过 voting exclusions API 将节点排除出去。如果使用默认的自动维护VotingConfiguration，也可以使用 voting exclusions API 来排除节点，例如一次性下线半数以上的节点。

如果在维护VotingConfiguration时发现节点数量为偶数，ES 会将其中一个排除在外，保证 VotingConfiguration是奇数。因为当是偶数的情况下，网络分区将集群划分为大小相等的两部分，那么两个子集群都无法达到“多数”的条件。

### dynamic mapping
es 能够识别新加字段，以及自动推断新加字段的索引，但是会触发一个mapping task，由master 同步到所有节点，如果很频繁会很耗性能。
### 写入
![image](https://user-images.githubusercontent.com/20329409/220523703-d451d112-030f-4b8f-b291-1d4f986bb75c.png)

![image](https://user-images.githubusercontent.com/20329409/220523717-2faf81a9-6d00-4208-8400-d3846aa00ba0.png)
1. 首先随机挑选一个节点作为协调节点转发请求，然后primary shard 所在节点写入，之后再将数据同步到副本，协调节点等primary shard 和secondary shard 都执行完成之后再返回成功。
2. 在某个节点写入的时候，先写memo table，再写log，memo table 会隔默认一秒refresh 成segment 才构建lucene 索引，才可以搜索；在内存中还没有refresh 之前可以根据docid 来搜索；直到segment 持久化到磁盘之后才会把log 清空。
3. segment 会定期合并，整个写入类似 LSM 

### 某些字段不需要直接查询, 从而关闭 index, 减少空间使用



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

### curator 删除过期的 indexer
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

#### 常用命令
* template
curl -X DELETE "ip:port/_template/templateName"
curl -X GET "ip:port/_template/templateName"
* es version
 curl -XGET "ip:port"



* index 
curl -X PUT "ip:port/indexName"
curl -X DELETE "ip:port/indexName"

### documation 重点, 可以作为ppt 的内容
1. request 和response的json的格式, 如何快速查询, 而不是每次都google 
应该不需要用query json来查, 毕竟写json 还是太麻烦了, 另外 :`_score` field in the search results , 这个字段是啥意思, 

2. how to use sql in ES. 

### master elasticSearch
1. lucene 和 es 的关系
2. Apache Lucene architecture, 四个重要概念

* Getting deeper into Lucene index

NormsA norm is a factor associated with each indexed document and  stores normalization factorsused to compute the **score** relative to the query.
Term vectors
是一个document 维度的倒排索引, 由term 和他出现的频率决定, 并包括term 的position 

Posting formats
控制着index file 如何被写入磁盘的

doc values
Lucene index is the so-called inverted index. However, forcertain features, such as faceting or aggregations, such architecture is not the best one. 因为faceting 和aggregation operate on the document level, 而不是term level, 所以需要doc value存储着  uninvert the field 用于上面的那些操作, 


* Lucene query language (可以简单讲下查询的语法)
fuzzy search
if we would run a query, such as writer~2, both the terms writer and writers would be considered a match
For example, let’s take the followingquery:title:"mastering Elasticsearch"It would match the document with the title field containing mastering Elasticsearch,but not mastering book Elasticsearch. However, if we would run a query, such astitle:"mastering Elasticsearch"~2, it would result in both example documentsmatched.

We can also use boosting to increase our term importance by using the ^ character andproviding a float number. Boosts lower than one would result in decreasing the documentimportance. Boosts higher than one will result in increasing the importance. The defaultboost value is 1

查询特殊字符:
In case you want to search for one of the special characters (which are +, -, &&, ||, !, (, ),{ }, [ ], ^, ", ~, *, ?, :, \, /), you need to escape it with the use of the backslash (\)character
### es basic concepts
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

* segment 
elasticsearch中的每个分片包含多个segment，每一个segment都是一个倒排索引；在查询的时，会把所有的segment查询结果汇总归并后最为最终的分片查询结果返回；
* Replica
每一个 shard 都有多个副本3. 倒排索引, reversed index

#### Apache Lucene scoring
A score is a factor that describes how well the document matched the query. 等于说要提高query 的准确性, 准确匹配到我们想要的结果的话, 就需要了解score 的计算原理
scoring mechanism: the TF/IDF(term frequency/inverse document frequency) algorithm
* In order to calculate the score property for adocument, multiple factors are taken into account, which are as follows( ignore)
* What you should be aware of is what matters when it comes to document score. Basically,there are a few rules
1. term 越稀有, doc 的 score 越高
2.  doc 的 fields 越少, score 越高
3. 设置的权重, (索引和搜索时设置的), 越大, score 越高

可以讲下这个例子, An example . Till now we’ve seen how scoring works. Now we would like to show you a simpleexample of how the scoring works in real life. To do this, we will create a new indexcalled scoring.

> Written with [StackEdit](https://stackedit.io/).
<!--stackedit_data:
eyJoaXN0b3J5IjpbMTk0NzA3MTgwM119
-->
