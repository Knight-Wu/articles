#### 异常现象
启用HA 的hdfs 集群的两个NN 都是standby, ls 命令报错也是"operation is not support in standBy nn"

#### 解决思路
hdfs 的HA 是通过zookeeper和zkfc 来协同的, 故检查zkfc 的日志
![enter image description here](https://drive.google.com/uc?id=1UemY2eTs8V3bjp08e7XZJPFsmybyODUU)

通过查看异常栈的相关代码, 发现是

> Written with [StackEdit](https://stackedit.io/).
<!--stackedit_data:
eyJoaXN0b3J5IjpbLTIxNDY0NzI3OTFdfQ==
-->