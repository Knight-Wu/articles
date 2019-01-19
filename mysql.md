####  存储引擎
* InnoDB 
是mysql 默认的事务型引擎, 支持事务, 支持行锁, 支持崩溃后自动恢复, 基于聚簇索引, 
https://juejin.im/post/5b1685bef265da6e5c3c1c34

* MyISAM
不支持行锁, 只支持表锁, 不支持事务, 不支持崩溃后快速回复, 不支持外键, 适合读的场景


#### 索引

最左前缀匹配原则, 碰到范围查询(>、<、between、like) 就终止匹配, 比如a = 1 and b = 2 and c > 3 and d = 4 如果建立(a,b,c,d)顺序的索引，d是用不到索引的，如果建立(a,b,d,c)的索引则都可以用到，a,b,d的顺序可以任意调整。 2.=和in可以乱序，比如a = 1 and b = 2 and c = 3 建立(a,b,c)索引可以任意顺序，mysql的查询优化器会帮你优化成索引可以识别的形式。

尽量选择区分度高的列作为索引, 

索引列的值不能作为函数入参进行计算, 比如from_unixtime(create_time) = ’2014-05-29’就不能使用到索引，原因很简单，b+树中存的都是数据表中的字段值，但进行检索时，需要把所有元素都应用函数才能比较，显然成本太大。所以语句应该写成create_time = unix_timestamp(’2014-05-29’)

#### 慢查询优化
1. 先设置sql_no_cache , 看查询是否真的很慢
2. 

> Written with [StackEdit](https://stackedit.io/).
<!--stackedit_data:
eyJoaXN0b3J5IjpbMTU5MjQ1NjE4MCw3MjQ4MTkzODcsOTA5OT
IwNjcwLC0xMzU4MjI0NTI4LDE0ODUxMTQxOTcsNzMwOTk4MTE2
XX0=
-->