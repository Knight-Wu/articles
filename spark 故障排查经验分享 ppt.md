
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
driver : yarn 界面和命令
executor : 由yarn nodemanager管理, 
yarn.nodemanager.remote-app-log-dir  			  /tmp/logs
yarn.nodemanager.remote-app-log-dir-suffix   logs
可以通过 hadoop fs -get /tmp/logs/${usr}/logs/${applicationId}
或yarn 界面查看, 点进去可以看到这个application 的container 的运行的应用的log, stderr, stdout. 
![enter image description here](https://drive.google.com/uc?id=1Fj8qN6D4HqfM0YO2YiP2QB496xOiakS6)



> Written with [StackEdit](https://stackedit.io/).
<!--stackedit_data:
eyJoaXN0b3J5IjpbLTY4NTY2NjYzMSwzMjMyNjQwNTRdfQ==
-->