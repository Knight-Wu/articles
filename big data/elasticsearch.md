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
```

curl -X DELETE "10.70.16.200:31200/order_realtime_25"

curl -X DELETE "10.70.16.200:31200/_template/template_order_etl"

curl -X GET "10.70.16.200:31200/_template/template_order_etl"

curl -XGET "http://10.70.16.200:31200"

list index: curl -XGET "http://10.65.229.119:9201"
create index: curl -X PUT "ip:port/index_name"

```
> Written with [StackEdit](https://stackedit.io/).
<!--stackedit_data:
eyJoaXN0b3J5IjpbLTQ3MjU0NjIyMl19
-->