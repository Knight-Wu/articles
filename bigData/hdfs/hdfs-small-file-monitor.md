#### 目的 
1. 分析: 分析现有hdfs 的文件系统, 得出哪些目录下的小文件数量较多
2. 监控: 周期性扫描文件目录, 若某个目录的小文件数量增多, 则上报
资料: [https://community.hortonworks.com/content/supportkb/151118/small-files-detection.html](https://community.hortonworks.com/content/supportkb/151118/small-files-detection.html)
#### 小文件的标准

文件大小远小于块大小的, 并需要返回小文件路径的前五级目录


#### 数据来源
hbase, hive

* hbase


hbase 目录结构: /hbase/data/default/tableName/regionName/columnFamilyName/hfile

* hive 

hive目录结构: /hive/warehouse/databaseName.db/tableName/





> Written with [StackEdit](https://stackedit.io/).
<!--stackedit_data:
eyJoaXN0b3J5IjpbMTA1ODczNTg5MV19
-->
