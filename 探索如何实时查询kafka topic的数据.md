#### 需求
在做实时变量分析的时候, 需要读取kafka 消息, 但是谁也不能精确给出topic 中具体的数据格式, 例如复杂嵌套和需要关联多个topic 的时候

#### 目的
寻找一个技术, 能交互式查询生产kafka topic数据, 最好能支持sql, 轻量化, 查询多个topic, 实时查询

#### 结果
ksq 目前是可以支持sql 的查询, 但是必须要使用[Confluent Platform(兼容目前公司的kafka 的版本)](https://docs.confluent.io/3.2.4/platform.html) 来部署kafa和zookeeper, 

* Confluent Platform
可以简单理解为hadoop 的企业版, 例如cdh , 简要介绍Confluent Platform的几个功能
  * [ksql](https://docs.confluent.io/current/ksql/docs/quickstart.html)
可以使用sql 实时查询 kafka 多个topic 的消息, 并支持持久化. 
  * control center 
 可以理解为kafka manager的地位, 但是有更多的功能, 比如可以在web 页面上直接发送, 接受kafka 消息, 


> Written with [StackEdit](https://stackedit.io/).
<!--stackedit_data:
eyJoaXN0b3J5IjpbLTMwMDU4OTY0LC01NjA3MTQ4MjJdfQ==
-->