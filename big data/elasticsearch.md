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
2.   Apache Lucene architecture, 四个重要概念

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
if we would run a query, such aswriter~2, both the terms writer and writers would be considered a match
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
* shard
Elasticsearch divide index data to several physical Lucene indices, every lucene indice is called shard
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
eyJoaXN0b3J5IjpbLTkwODU0ODUwMSwtNDAyNzQyNDYwLDgzNj
Q4NzUyNywtOTkxMTU2NDUxLC0xODYwOTg1MTIsMjAyMTY1NzY0
LDEyNjcyOTMwMDEsLTE2MzA5OTMxODEsMTc1NTA3NjAxOCwtMT
A5NjkwNjcwMSwtMTcwNTc5MzcwMywtMzU4MzM5MTc2LC04MTkx
OTQ1MTksMTgzMDQzMTk5OSwtNDc3OTg4MjA2LC0xNDYyNTA1MD
M1LDc3MzA4MzUzNiwtMTkzODc3NTMxOCwtMTMxOTUyODY0NCwy
MDI1MTI1NjUzXX0=
-->