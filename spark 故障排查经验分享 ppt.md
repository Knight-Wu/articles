
## spark 故障排查
从一次跑批组的异常说起, driver 端出现classNotFoundEx, 导致部分task 无法完成, stage 卡住, job中止
![2](https://user-images.githubusercontent.com/20329409/45937070-a4562c80-bfef-11e8-8a51-ad5d451148a8.png)

* 可能是jar包冲突
https://cwiki.apache.org/confluence/display/Hive/Hive+on+Spark%3A+Getting+Started
spark-1.6和hive-2.2是兼容的

* 可能是类缺失
```
hive -hiveconf spark.driver.extraClassPath=./antlr-runtime-3.4.jar  -hiveconf spark.executor.extraClassPath=./antlr-runtime-3.4.jar -hiveconf spark.yarn.dist.files=/opt/cloudera/parcels/CDH-5.12.1-1.cdh5.12.1.p0.3/jars/antlr-runtime-3.4.jar
```
并且打印executor和driver加载的class, 仍然没发现加载了commonTree这个class, 且仍然报错
```
spark.executor.extraJavaOptions=-verbose:class
spark.driver.extraJavaOptions=-verbose:class
```

> Written with [StackEdit](https://stackedit.io/).
<!--stackedit_data:
eyJoaXN0b3J5IjpbLTgzMDk0MTk5NV19
-->