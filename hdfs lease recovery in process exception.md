## hdfs lease recovery in process exception

hive on spark 出现这个异常, 如下图, hive是apache hive-2.2, spark 是cdh 1.6 版本, 在hdfs startFile 这一步出现了异常. 原因是有其他客户端占用了lease, 其他客户端检查到这个lease 已经被占用, 且超过softlimit 设置的时间, 故进行抢占, 进行lease recovery, 再进行lease recovery 的时候, 发现文件的最后一个块不是 COMPLETED , 也就是这个块的所有副本有一定数量的FINALIZED replica, (一定数量根据min-replica 来控制), 故需要进行block recovery, 就是把所有replica 统一成一样大小, 根据最近一次更新的replica ,具体怎么恢复?

![enter image description here](https://drive.google.com/uc?id=1z83xj9YOKodLUEvCoFhQCGiZriAUJG7Q)

![enter image description here](https://drive.google.com/uc?id=1IaTcvMjNX-Asq4HchjYHhPnshd82fFcA)
> Written with [StackEdit](https://stackedit.io/).
<!--stackedit_data:
eyJoaXN0b3J5IjpbLTE3NjgyOTA4NTEsLTM5Njc2NjIzMiwxMj
Y3NTU3OTk4LDEwMzg3NTIxLDczMDk5ODExNl19
-->