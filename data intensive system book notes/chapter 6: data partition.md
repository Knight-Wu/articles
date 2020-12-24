### 重点
Cassandra 可以将多个列作为复合主键, 第一个列用作 hash 分区的 key, 后几列用作 range 分区的 key, 当第一列指定的时候, 可以对后几列做范围查询, 例如 key: {userId, update_timestamp} 就很适用


> Written with [StackEdit](https://stackedit.io/).
<!--stackedit_data:
eyJoaXN0b3J5IjpbMTk1NjMyMTc3Nl19
-->