#### 需求
在做实时变量分析的时候, 需要读取kafka 消息, 但是谁也不能精确给出topic 中具体的数据格式, 例如复杂嵌套和需要关联多个topic 的时候

#### 目的
寻找一个技术, 能交互式查询生产kafka topic数据, 最好能支持sql, 轻量化, 查询多个topic, 实时查询

#### 结果
ksql https://docs.confluent.io/current/ksql/docs/tutorials/basics-local.html目前是可以支持sql 的查询,


> Written with [StackEdit](https://stackedit.io/).
<!--stackedit_data:
eyJoaXN0b3J5IjpbLTgwMzAwMjc0XX0=
-->