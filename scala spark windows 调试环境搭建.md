### 前言
老早就想在windows 搭建一个本地环境, 一拖再拖, 这次终于花了两三天零碎的时间搞定了. 本文属于搭建好之后的总结, 一些零碎和基础的点就没写, 只画重点和一些常见的问题. 

---

### 一. 环境准备
1. ide 安装scala环境, 需要从scala官网下载scala 压缩包



#### 遇到的问题
* 在bin/hadoop 路径下, cmd输入hadoop classpath出错, 详情见[# [Hadoop on Windows - “Error JAVA_HOME is incorrectly set.”](https://stackoverflow.com/questions/31621032/hadoop-on-windows-error-java-home-is-incorrectly-set)](Hadoop%20on%20Windows%20-%20%E2%80%9CError%20JAVA_HOME%20is%20incorrectly%20set.%E2%80%9D), 原因是JAVA_HOME 的路径包括有空格, 去掉空格即可.

* spark依赖hadoop版本问题, 详情见[# [NoClassDefFoundError com.apache.hadoop.fs.FSDataInputStream when execute spark-shell](https://stackoverflow.com/questions/30906412/noclassdeffounderror-com-apache-hadoop-fs-fsdatainputstream-when-execute-spark-s), 要添加hadoop classpath的输出, 和HADOOP_HOME(因为启动需要hadoop/bin/winutils.exe)



> Written with [StackEdit](https://stackedit.io/).
<!--stackedit_data:
eyJoaXN0b3J5IjpbLTE4NTY0OTE3NjgsNDE4MDE1OTcsLTc4OD
EzODM5M119
-->