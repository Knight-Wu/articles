#### 异常现象
启用HA 的hdfs 集群的两个NN 都是standby, hdfs ls 命令报错也是"operation is not support in standBy nn"

#### 解决思路
hdfs 的HA 是通过zookeeper和zkfc 来协同的, 故检查zkfc 的日志
![enter image description here](https://drive.google.com/uc?id=1UemY2eTs8V3bjp08e7XZJPFsmybyODUU)

通过查看异常栈的相关代码, zkfc 进行集群选举的时候, 在选举了nn 作为active nn之后, 需要检查 zk node: /hadoop-ha/NameNodeHA/ActiveBreadCrumb/ 是否存在old active nn的信息, 若存在, 则需要进行fencing, 但是可能由于之前重启等操作没有执行完, 这个节点的信息出现了错乱, 和hdfs-site.xml 里面的配置不一致, 找不到old active nn 的rpc address, 就选举出active nn. 

故通过zkcli 手动清空了 /hadoop-ha/NameNodeHA/ActiveBreadCrumb/ 这个节点, 清空前后的截图如下, 发现nn 的逻辑id 与hdfs-site.xml 里面的配置是一致了. 
![enter image description here](https://drive.google.com/uc?id=1VbpDrK0x9QlACs4IFoPH6ahdKg--DKQJ)


然后zkfc 再次进行选举, 日志如下图, 因为 old  active nn 信息为空了, 跳过了fencing, 成功注册了 active nn.

![enter image description here](https://drive.google.com/uc?id=1_2_IJK96wFAC4xyvlIFECFXAWKUaYpFN)

> Written with [StackEdit](https://stackedit.io/).
<!--stackedit_data:
eyJoaXN0b3J5IjpbLTg0NDE4MDc0OF19
-->