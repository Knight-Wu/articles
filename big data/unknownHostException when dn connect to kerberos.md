
![enter image description here](https://drive.google.com/uc?id=1qSuwj042SNRrOlz6P39Yif_Ysd9-1YOE)
![enter image description here](https://drive.google.com/uc?id=15ks42HSeesB5DtYTAlIAKjDgyw7fDBZB)

datanode 报告heartBeat 给nn的时候, 需要先认证 kerberos, 但是认证出现了异常, 导致nn 认为dn 挂掉, 把多台dn 都踢掉了, 整个hdfs 集群磁盘容量暴增, 差点引起雪崩. 

#### 问题
1. 为什么出现了认证
2. 认证为什么出现unknownHostEx
3. 如何监控, 因为dn 进程和端口都在, 只能更细粒度的监控 .

#### 为什么出现了认证
看异常栈是因为这个, 可能一瞬间流量比较大

#### 认证为什么出现unknownHostEx
一步步跟随异常栈看代码, 关键点 
> Written with [StackEdit](https://stackedit.io/).
<!--stackedit_data:
eyJoaXN0b3J5IjpbLTE5OTMyMzY4OCw3MzA5OTgxMTZdfQ==
-->