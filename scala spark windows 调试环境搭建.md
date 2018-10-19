### 前言
老早就想在windows 搭建一个本地环境, 一拖再拖, 这次终于花了两三天零碎的时间搞定了. 本文属于搭建好之后的总结, 纯手打, 一些零碎和基础的点就没写, 只画重点和一些常见的问题. 

---

### 一. 环境准备
1. 安装scala环境, 需要从scala官网下载scala 压缩包, idea安装scala插件
2. 下载hadoop windows编译好的包和spark 包, hadoop 下载: [hadoop-2.6.0 windows](http://www.barik.net/archive/2015/01/19/172716/), 这个链接是从 [[NoClassDefFoundError com.apache.hadoop.fs.FSDataInputStream when execute spark-shell](https://stackoverflow.com/questions/30906412/noclassdeffounderror-com-apache-hadoop-fs-fsdatainputstream-when-execute-spark-s)这个回答里面找的; spark下载: [spark-download](https://spark.apache.org/downloads.html), 这个是官网地址, choose a package type: pre-build with user-provided apache hadoop , 因为我们之前已经从其他地方下载了hadoop, 也可以选择使用hadoop 编译的版本, 这个没有试过, 
3.  从github 选择对应的分支, 下载spark的源码, 然后导入idea
4. 添加集群配置到hadoop 和spark的配置路径
	hadoop的配置路径: ./etc/hadoop , spark的配置路径: ./conf, 因为公司用的cloudera 集群, 我直接从cloudera-manager 下载了yarn和hadoop的客户端配置

5. 测试hadoop 环境

	用命令 hadoop fs -ls /
6. 测试spark 环境

	之前我天真的以为 spark 不强依赖hadoop, 直接用spark-shell 验证环境, 就碰到了这个错误: NoClassDefFoundError com.apache.hadoop.fs.FSDataInputStream when execute spark-shell 事后总结应该先验证hadoop, 可以参考这个链接的第一个回答: [NoClassDefFoundError com.apache.hadoop.fs.FSDataInputStream](%5BNoClassDefFoundError%20com.apache.hadoop.fs.FSDataInputStream%20when%20execute%20spark-shell%5D%28https://stackoverflow.com/questions/30906412/noclassdeffounderror-com-apache-hadoop-fs-fsdatainputstream-when-execute-spark-s%29),  总结一下: 目前spark需要依赖Hadoop client libraries for HDFS and YARN,  但是可以灵活依赖hadoop版本, 不需要像1.4之前和hadoop一起build, **目前把 bin/hadoop classpath的输出以及hadoopHome 配置到conf/spark-env.sh就可以了, windows是 conf/spark-env.cmd(没有就创建一个)**,如下图, 配置的加载逻辑可以看 spark-class脚本.
>	bin/hadoop classpath input

![hadoop classpath](https://drive.google.com/uc?id=1Mjm78ZLoSMzIpoegwPIaaIdxb0JO0Pvx)

> spark-env

![](https://drive.google.com/uc?id=1CZ8yTmYyNkv6MkvHHM4ZR5JX0Tx-A7Ha)

这一步还遇到一个javaHome配置的问题: 在bin/hadoop 路径下, cmd输入hadoop classpath出错, 详情见[# [Hadoop on Windows - “Error JAVA_HOME is incorrectly set.”](https://stackoverflow.com/questions/31621032/hadoop-on-windows-error-java-home-is-incorrectly-set)](Hadoop%20on%20Windows%20-%20%E2%80%9CError%20JAVA_HOME%20is%20incorrectly%20set.%E2%80%9D), 原因是JAVA_HOME 的路径包括有空格, 去掉空格即可.

在$SPARK_HOME/bin目录下, 执行

```
run-example --verbose --master local[*] SparkPi // 先测试local模式, verbose(打印详细配置)

spark-submit --help // chak
```

> Written with [StackEdit](https://stackedit.io/).
<!--stackedit_data:
eyJoaXN0b3J5IjpbLTEzOTMwNzIxMTQsLTQzMDc2NDQ1NSwtMT
cxMDM1NTcxOCwtNzM2MDgwMzgwLDQxODAxNTk3LC03ODgxMzgz
OTNdfQ==
-->