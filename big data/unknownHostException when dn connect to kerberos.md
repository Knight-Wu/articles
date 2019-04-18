
![enter image description here](https://drive.google.com/uc?id=1qSuwj042SNRrOlz6P39Yif_Ysd9-1YOE)
![enter image description here](https://drive.google.com/uc?id=15ks42HSeesB5DtYTAlIAKjDgyw7fDBZB)

datanode 报告heartBeat 给nn的时候, 需要先认证 kerberos, 但是认证出现了异常, 导致nn 认为dn 挂掉, 把多台dn 都踢掉了, 整个hdfs 集群磁盘容量暴增, 差点引起雪崩. 

#### 问题
1. 为什么出现了认证
2. 认证
> Written with [StackEdit](https://stackedit.io/).
<!--stackedit_data:
eyJoaXN0b3J5IjpbLTExNDY5MDg2ODQsNzMwOTk4MTE2XX0=
-->