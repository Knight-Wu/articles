很奇怪cdh 测试环境出现的这个问题, 死活查不到, spark-2.2 , hive-1.1, 使用spark-shell, spark.sql("show databases"), spark.sql("show tables") 均查不到

### 测试方法
* 看spark 的配置 是否依赖了hive 服务, 并且把spark shell的日志级别调到info, 跟踪, 看日志里, sparkSession 已经默认启用了hive support, 不需要显示指定 enableHiveSupport

![enter image description here](https://drive.google.com/uc?id=1EMvPfK4EHC1TQDEScJWgOzV6vRM__vkq)

* 查询spark 是否连接hive 正常
```
scala> spark.catalog.listDatabases.show(false) 
如果结果是类似这样的， 
name |description |locationUri
default|default database|file:/root/spark-warehouse|
说明没有找到hive warehouse, 找不到hive的数据
```

* 根据文档 [https://spark.apache.org/docs/2.2.0/sql-programming-guide.html#hive-tables](https://spark.apache.org/docs/2.2.0/sql-programming-guide.html#hive-tables)
> Note that the `hive.metastore.warehouse.dir` property in `hive-site.xml` is deprecated since Spark 2.0.0. Instead, use `spark.sql.warehouse.dir` to specify the default location of database in warehouse

显示指定了 spark.sql.warehouse.dir = "hive warehouse" 还是没用,spark.catalog.listDatabases.show(false) 这个命令输出仍然是单独的spark warehouse

* 显示指定hadoop confi dir

```
export HADOOP_CONF_DIR=/etc/spark2/conf/yarn-conf; sh spark2-shell 

```
然后终于查到了hive 表, 具体原因仍待查, 后续采用如下方式排查: 
```
set -x; source /etc/spark2/conf/spark-env.sh ; set +x 
// set -x 

```



> Written with [StackEdit](https://stackedit.io/).
<!--stackedit_data:
eyJoaXN0b3J5IjpbLTE5MDE0MDAxNzIsLTE4Mjg2Mjc1OTFdfQ
==
-->