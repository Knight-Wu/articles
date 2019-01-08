## hdfs lease recovery in process exception

hive on spark 出现这个异常, 如下图, hive是apache hive-2.2, spark 是cdh 1.6 版本, 在hdfs startFile 这一步出现了异常. 原因是有其他客户端占用了lease, 其他客户端检查到这个lease 已经被占用, 且超过softlimit 设置的时间, 故进行抢占, 进行lease recovery, 再进行lease recovery 的时候, 发现文件的最后一个块不是 COMPLETED , 也就是这个块的所有副本有一定数量的FINALIZED replica, (一定数量根据min-replica 来控制), 


> Written with [StackEdit](https://stackedit.io/).
<!--stackedit_data:
eyJoaXN0b3J5IjpbLTI1OTg2OTMzMyw3MzA5OTgxMTZdfQ==
-->