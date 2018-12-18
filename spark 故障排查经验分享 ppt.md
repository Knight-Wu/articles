
## spark 故障排查
从一次跑批组的异常说起, driver 端出现classNotFoundEx, 导致部分task 无法完成, stage 卡住, job中止
![2](https://user-images.githubusercontent.com/20329409/45937070-a4562c80-bfef-11e8-8a51-ad5d451148a8.png)

* 可能是jar包冲突
https://cwiki.apache.org/confluence/display/Hive/Hive+on+Spark%3A+Getting+Started
spark-1.6和hive-2.2是兼容的

* 可能是类缺失

```
hive -hiveconf spark.driver.extraClassPath=./antlr-runtime-3.4.jar  -hiveconf spark.executor.extraClassPath=./antlr-runtime-3.4.jar -hiveconf spark.yarn.dist.files=/opt/cloudera/parcels/CDH-5.12.1-1.cdh5.12.1.p0.3/jars/antlr-runtime-3.4.jar
// 将jar包放到driver和executor的container中, 前两个参数告诉container jar包在哪 , 第三个参数会把jar包加载到container中.
```

```
spark.executor.extraJavaOptions=-verbose:class
spark.driver.extraJavaOptions=-verbose:class
// 打印executor和driver加载的class
```
---

尝试了一番无果, 可能解决思路不对, 重新汇总所有信息, 从查看所有spark 日志开始

* spark 日志分类
1. driver : yarn 界面和命令
2. executor : 由yarn nodemanager管理, 
yarn.nodemanager.remote-app-log-dir  			  /tmp/logs
yarn.nodemanager.remote-app-log-dir-suffix   logs
可以通过 hadoop fs -get /tmp/logs/${usr}/logs/${applicationId}
或yarn 界面查看, 点进去可以看到这个application 的container 的运行的应用的log, stderr, stdout. 
![enter image description here](https://drive.google.com/uc?id=1Fj8qN6D4HqfM0YO2YiP2QB496xOiakS6)

3. eventLog
通过spark history server 查看, 提供可视化界面

> 问题来了, 如何知道异常的task 是属于哪个executor ? 

eventLog missing, grep ex-timestamp -C 1000  *executor.log  |less

* executor 异常
![图3](https://user-images.githubusercontent.com/20329409/45937116-ea12f500-bfef-11e8-9e82-11c46502b1d9.png)



* hdfs 写入流程
![](https://drive.google.com/uc?id=1LjDrWGX6zhQzEJOzNWG615eKFyK2XHDF)
异常场景: executor 写dn, 客户端检查这个块的时候发现一个副本也没有, 出现在上文第二步. 同时dn 出现"IOException: channel is broken " .

* IO 层面的异常
![enter image description here](https://drive.google.com/uc?id=1yJN8y_HmloD7eTdDSwcV4w67gk61hlW4)

* hdfs 集群容错

最根本的原因可能是dn 写失败, 思路是: 某个 dn写失败时,重试其他dn

![image](https://user-images.githubusercontent.com/20329409/45943455-ee073d00-c018-11e8-88a5-f251c1d42453.png)
[http://blog.cloudera.com/blog/2015/03/understanding-hdfs-recovery-processes-part-2/](http://blog.cloudera.com/blog/2015/03/understanding-hdfs-recovery-processes-part-2/)
https://hadoop.apache.org/docs/r2.4.1/hadoop-project-dist/hadoop-hdfs/hdfs-default.xml
DFSOutputStream.DataStreamer

> dfs.client.block.write.replace-datanode-on-failure.policy

如果是default则按照公式进行计算, 
r 是副本数 3, n是集群中现有的dn的数量, 目前是 12
r>=3 &&( floor(r/2) >= 12 || r>=n && (block is flushed or appended))
如果是always, 每次都新加一个dn到pipeline中.

>  dfs.client.block.write.replace-datanode-on-failure.best-effort

假设这个参数是false, datanode replacement 的过程失败的话就会直接抛出异常, 终止重试; 如果设为true, 则假设replacement的dn也写失败, 仍然会找新的dn去重试.

* 解决办法
dfs.client.block.write.replace-datanode-on-failure.policy = always, 
dfs.client.block.write.replace-datanode-on-failure.best-effort = true


* spark on yarn 的其他容错策略

* spark 本地环境debug

* spark 的执行任务的整个流程



> Written with [StackEdit](https://stackedit.io/).
<!--stackedit_data:
eyJoaXN0b3J5IjpbMzQ3NDEwOTAzLDEwNjMzNjQxMTEsMTQ5ND
QwMDEzNiwtMTQ2NDMzMzMwNSwzMjMyNjQwNTRdfQ==
-->