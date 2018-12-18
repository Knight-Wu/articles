> 记录一下这个排查了挺久的生产hive集群跑批问题, 使用场景是hive on spark, hive版本是2.2, 使用的原生社区的版本, spark版本是1.6, 使用的是cdh的spark.
> 大致场景是hive on spark每天晚上跑批的时候, 集群高峰时段偶尔会有任务卡主, 然后超时被我们的调度系统杀掉, 影响跑批的进度
* 现象
1. yarn logs收集到driver端的日志, 报错是 commonTree classNotFound exception 
![1](https://user-images.githubusercontent.com/20329409/45937069-a3bd9600-bfef-11e8-9ea0-fc2b8f17d1ee.png)
![2](https://user-images.githubusercontent.com/20329409/45937070-a4562c80-bfef-11e8-8a51-ad5d451148a8.png)

然后因为有部分task没有完成, 导致下一个stage没能开始, 这个job就一直卡住了.

2. 同一时间在executor上面报了如下异常, executor日志如下, 这个日志跟上面driver端的时间不一样, 因为这后面写这篇文章的时候, 截图比较乱了, 没有找到完整一个job的所有异常截图.
![图3](https://user-images.githubusercontent.com/20329409/45937116-ea12f500-bfef-11e8-9e82-11c46502b1d9.png)

> 因为hdfs写文件的时候, 当client把最后一个block 提交到dn之后, 最后通过 DFSInputStream.close() 去关闭连接, 会轮训请求nn 进行一系列的检查, 包括文件副本最小数必须大于1(默认配置), 否则抛出异常给client.
> 同时namenode日志如下: 
![4](https://user-images.githubusercontent.com/20329409/45937181-863cfc00-bff0-11e8-8696-74f72c3fb013.png)


3. datanode的日志找不到了,  同时出现了 "IOException: channel is broken ". 代码如图

![enter image description here](https://drive.google.com/uc?id=1oBOiQAmfVNUQpNunfBTbUgKR-Pqpl1C-)

> 总结来看, 由于dn写失败, 导致nn检查小于最小副本数, executor接受到无法关闭文件的异常, driver 反序列化executor 端异常时出现classNotFoundEx.
 
* 解决过程
4. 关于第一个现象, 一开始只发现了第一个现象, 没有发现executor的异常, 所以自然是觉得jar包问题, 尝试了如下几种添加jar包的方式: 
* hive添加配置: Add Jar(一开始不懂, 应该在spark端添加jar)
* spark依赖第三方jar, 将jar包放到driver和executor的container中, 前两个参数告诉container jar包在哪 , 第三个参数会把jar包加载到container中.
```
hive -hiveconf spark.driver.extraClassPath=./antlr-runtime-3.4.jar  -hiveconf spark.executor.extraClassPath=./antlr-runtime-3.4.jar -hiveconf spark.yarn.dist.files=/opt/cloudera/parcels/CDH-5.12.1-1.cdh5.12.1.p0.3/jars/antlr-runtime-3.4.jar
```
并且打印executor和driver加载的class, 仍然没发现加载了commonTree这个class, 且仍然报错
```
spark.executor.extraJavaOptions=-verbose:class
spark.driver.extraJavaOptions=-verbose:class
```
期间还设置了eventLog, 以上方法都没有解决这个问题, 后面发现了现象2.

5. 针对executor写datanode报错, IO层面的异常就比较难排查了, 暂且想到了几个优化集群的方案.
* 禁止THP(Transparent-Huge-Page-THP-Compaction-Causes-Poor-Performance)
* 小文件问题, 见我收藏的资料, http://note.youdao.com/noteshare?id=0f01349f4abfc0834d3f6cebc4fbf33f&sub=wcp1536734480094475

6. 根据图3的异常, 看了下源码, completeFile的时候会检查block的最小副本数是否达到, 客户端会轮询等待nn, 根据后续block 结束completeFile的时间(大概有二十几秒), 增加了retry次数之后, 后续的达不到最小副本数的异常有所减小, 
![image](https://user-images.githubusercontent.com/20329409/45943305-51dd3600-c018-11e8-85df-44ffa688c109.png)

但是仍然出现异常, 想了下最根本的异常是dn写文件失败, 可以尝试一下dn 故障转移, 让pipeline的某个 dn写失败时,重试其他dn, 通过多方资料找到以下配置: 
![image](https://user-images.githubusercontent.com/20329409/45943455-ee073d00-c018-11e8-88a5-f251c1d42453.png)
并结合[http://blog.cloudera.com/blog/2015/03/understanding-hdfs-recovery-processes-part-2/](http://blog.cloudera.com/blog/2015/03/understanding-hdfs-recovery-processes-part-2/)这篇文章和源码(DFSOutputStream.DataStreamer) 进行理解.

> dfs.client.block.write.replace-datanode-on-failure.policy

这个配置的解释一开始没看懂n这个参数( let n be the number of existing datanodes), 反复理解, 应该是集群中现有的dn的数量,如果是default则按照公式进行计算,  如果是always, 每次都新加一个dn到pipeline中.

>  dfs.client.block.write.replace-datanode-on-failure.best-effort

假设这个参数是false, 如果作为replacement的dn也写失败的话就会直接抛出异常, 终止重试; 如果设为true, 则假设replacement的dn也写失败, 仍然会找新的dn去重试.


> 所以我们想要的是反复重试新的dn, 直到客户端发起completeFile请求时, 轮询nn超时, 故把dfs.client.block.write.replace-datanode-on-failure.policy设置为always, dfs.client.block.write.replace-datanode-on-failure.best-effort设为true, namenode的block state change的日志级别调成debug(方便观察), 再观察后续出现写异常的时候是否有重试其他dn.

**一直没有来反馈** ， 经过后续的观察，再也没有出现commonTree的报错了， 证明解决问题的思路是正确的！

* 根本原因

应该是dn 写入失败, executor 报错而终止job, 至于为什么dn 会写入失败, 这个情况只在凌晨hive 跑批, 集群压力大的时候出现, 其他时间均为出现, 后来查询到可能是CentOS 的一个bug, 其他团队在升级了操作系统版本之后解决, 这个还有待考察. 


* 问题
  * 因为具体的blocken pipe的错误日志找不到了, 为什么没有触发spark 的集群容错有点困惑, 一个task 写异常至少是会根据 spark.task.maxFailures 这个配置重试的啊, 不知道这个异常抛到哪个层面了


 



 

 




> Written with [StackEdit](https://stackedit.io/).
<!--stackedit_data:
eyJoaXN0b3J5IjpbODAyOTIyMTE0LDc5OTM0NTYxMiwxNzY4OD
IxMDQ2LC0xMTg3NjI1MjA0LDE5MTc4NzczNTQsLTI4NTY0NDk0
OSwxMTU0MzAyMzYxLC0xODkxNzMyNzY5XX0=
-->