#### 目的 
想对hdfs 小文件数量作监控, 以明确某个目录下的小文件数量过多, 这个目录是哪个业务或者组件使用的, 可以进一步优化

#### 数据来源
hbase, hive

* hbase


hbase 目录结构: /hbase/data/default/tableName/regionName/columnFamilyName/hfile

* hive 

hive目录结构: /hive/warehouse/databaseName.db/tableName/





> Written with [StackEdit](https://stackedit.io/).
<!--stackedit_data:
eyJoaXN0b3J5IjpbLTEzNzAxMzc3OCwzMTMzMzAwMjQsMTE1ND
g3OTgxNSwyMDE2MDg2ODNdfQ==
-->