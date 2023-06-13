
## sqoop执行mysql命令,貌似只限于增删查改
* eval statement
```
sh  /app/hadoop/sqoop/bin/sqoop   eval  --connect "jdbc:mysql://sample.com:3306/crm_cust?useUnicode=true&characterEncoding=utf-8"   --username hive --password Hive#Qsmyhs4g -e "truncate crm_cust.exp_ntw_act"
```

# export to RDBMS
## example 
```
sh  /app/hadoop/sqoop/bin/sqoop export -D mapred.job.queue.name=group1 -D sqoop.export.records.per.statement=100 -D sqoop.export.statements.per.transaction=60 --connect "jdbc:mysql://mysql.com:3306/db?useUnicode=true&characterEncoding=utf-8" --username hive --password pwd --table exp_usr_inf --update-key cust_id,usr_id --update-mode allowinsert --fields-terminated-by '\001' --export-dir "hdfs:table" --input-null-string '\\N' --input-null-non-string '\\N' -m 30

```
## configuration
* -D sqoop.export.records.per.statement=100, -D sqoop.export.statements.per.transaction=100
  * 可控制mysql 一个transaction中操作的记录,极其影响速度,默认两个均为100
  * -D property=value 这类配置要写在连接配置的前面,写在最前面准没错了
  * 还可以把此类配置写到一个配置文件中, 具体语法忘了

* -m=mapper num 
可控制mappers的数量,次要影响速度,默认是四,具体数量需要考虑hive 中表分了几个文件,每个文件默认128MB,故也不是越大m越好,盲目增加m提升速度很少,主要取决于数据库的速度

* --direct
需要mysql 加装mysqlimport client,可尝试是否提升性能

* --batch

## FAQ
### 1. batch export to RDBMS(mysql etc.), BatchUpdateExeption:deadlock found when trying to get lock,try restarting transcation.

用show engine innodb status,的确是有行锁, 后面发现是有两条主键相同的记录, 在两个事务中产生了死锁.
<!--stackedit_data:
eyJoaXN0b3J5IjpbLTIwODY2MzQ1NzBdfQ==
-->
