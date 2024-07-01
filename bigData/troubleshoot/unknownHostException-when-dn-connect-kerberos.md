
![enter image description here](https://drive.google.com/uc?id=1qSuwj042SNRrOlz6P39Yif_Ysd9-1YOE)
![enter image description here](https://drive.google.com/uc?id=15ks42HSeesB5DtYTAlIAKjDgyw7fDBZB)

datanode 报告heartBeat 给nn的时候, 需要先认证 kerberos, 但是认证出现了异常, 导致nn 认为dn 挂掉, 把多台dn 都踢掉了, 整个hdfs 集群磁盘容量暴增, 差点引起雪崩. 

#### 问题
1. 为什么出现了认证
2. 认证为什么出现 unknownHostException
3. 如何监控, 因为dn 进程和端口都在, 只能更细粒度的监控 .

#### 为什么出现了认证
看异常栈是因为这个, 可能一瞬间流量比较大
![enter image description here](https://drive.google.com/uc?id=1tKDfUS9S3IdNdT1ErJ2cyq5T2TpYLOFY)
#### 认证为什么出现unknownHostEx
一步步跟随异常栈看代码, 关键点:
 1. 先从缓存解析host, 为了防止dns 中间人攻击, 默认successCache 是forever 的, 有个疑问, 那修改了host 如何取到新的呢? 
![enter image description here](https://drive.google.com/uc?id=1GWodCqUsiDTDekvz4VOsJScPwtZRZyUr)

2. lookup table, 用于多线程环境下, 遇到比较耗时的可以共享结果的操作, 一个线程去查了, 其他线程就block 等他返回好了, 不要再去查. 
![enter image description here](https://drive.google.com/uc?id=14gGYalVWkkNjA_D85zXrkOF9Z2Jv1DD-)
3. 


#### 资料
* kerberos  超时时间配置
[https://blog.csdn.net/yinansheng1/article/details/79397309](https://blog.csdn.net/yinansheng1/article/details/79397309)


> Written with [StackEdit](https://stackedit.io/).
<!--stackedit_data:
eyJoaXN0b3J5IjpbNjQxNzIyNTE4LDczMDk5ODExNl19
-->