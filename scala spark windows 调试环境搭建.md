#### 遇到的问题
* 在bin/hadoop 路径下, cmd输入hadoop classpath出错, 详情见[# [Hadoop on Windows - “Error JAVA_HOME is incorrectly set.”](https://stackoverflow.com/questions/31621032/hadoop-on-windows-error-java-home-is-incorrectly-set)](Hadoop%20on%20Windows%20-%20%E2%80%9CError%20JAVA_HOME%20is%20incorrectly%20set.%E2%80%9D), 原因是JAVA_HOME 的路径包括有空格, 去掉空格即可.

* spark依赖hadoop版本问题, 详情见[# [NoClassDefFoundError com.apache.hadoop.fs.FSDataInputStream when execute spark-shell](https://stackoverflow.com/questions/30906412/noclassdeffounderror-com-apache-hadoop-fs-fsdatainputstream-when-execute-spark-s), 要添加hadoop classpath的输出, 和HADOOP_HOME(因为启动需要hadoop/bin/winutils.ext)



> Written with [StackEdit](https://stackedit.io/).
<!--stackedit_data:
eyJoaXN0b3J5IjpbNjk3OTUwNjA0LC03ODgxMzgzOTNdfQ==
-->