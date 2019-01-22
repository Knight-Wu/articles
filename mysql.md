####  存储引擎
* InnoDB 
是mysql 默认的事务型引擎, 支持事务, 支持行锁, 支持崩溃后自动恢复, 基于聚簇索引, 
https://juejin.im/post/5b1685bef265da6e5c3c1c34

* MyISAM
不支持行锁, 只支持表锁, 不支持事务, 不支持崩溃后快速回复, 不支持外键, 适合读的场景


### 索引
#### INNODB 索引
* B+ tree index structure in INNODB 

![enter image description here](https://drive.google.com/uc?id=1jOIFUv2qT3d__lWSkkqsfuff2_N7LDoK)

* node size=disk page size=16KB , 可以在初始化mysql instance 的时候进行配置
* 所有数据都由叶子节点保管
* Because B-Trees store the indexed columns in order, they’re useful for searching for ranges of data.

一个实际的数据例子阐述innodb 索引: 
![enter image description here](https://drive.google.com/uc?id=1CCbvzgDAKugLkRhRx-7d7kg1bLVsLWL2)

根据以上结构, 支持全值匹配, 最左匹配, 范围匹配, 而且因为index value 也是排序的, 所以也优化了 order by, 
* 缺点
1. 不支持列顺序混乱的匹配, 例如index: (A,B,C), 查询顺序是(A,C,B), 只支持到A 列, 因为不知道B 列的长度.
2. 不支持跳列, 必须指定前几个列的值. 
3. if your query is WHERE last_name="Smith" AND first_name LIKE 'J%' AND dob='1976-12-23' , the index access will use only the first two columns in the index, because the LIKE is a range condition

* 使用B+ tree 索引的原则

1. 最左前缀匹配原则, 碰到范围查询(>、<、between、like) 就终止匹配, 比如a = 1 and b = 2 and c > 3 and d = 4 如果建立(a,b,c,d)顺序的索引，d是用不到索引的，如果建立(a,b,d,c)的索引则都可以用到，a,b,d的顺序可以任意调整。 2.=和in可以乱序，比如a = 1 and b = 2 and c = 3 建立(a,b,c)索引可以任意顺序，mysql的查询优化器会帮你优化成索引可以识别的形式。

2. 当不需要排序和分组的时候, 将选择性高的列放到多列索引的前边

3. 索引列的值不能作为函数入参进行计算, 比如from_unixtime(create_time) = ’2014-05-29’就不能使用到索引，原因很简单，b+树中存的都是数据表中的字段值，但进行检索时，需要把所有元素都应用函数才能比较，显然成本太大。所以语句应该写成create_time = unix_timestamp(’2014-05-29’)
4. 前缀索引
 只采取索引列的前缀作为索引, 减少空间, 但是这个前缀要能一定程度上的获取你想要的结果. 可以具体参考"高性能mysql 5.3.2"
5. 多列索引, 可能会引起索引的合并, 

* clustered index
聚簇索引并不是一个索引类型, 而是一种数据存储方式, 而InnoDB 的聚簇索引实际上就是B+ tree 的叶子节点将key 和data row 存放在一起. 

>  When you define a  `PRIMARY KEY`  on your table,  `InnoDB`  uses it as the clustered index. Define a primary key for each table that you create. If there is no logical unique and non-null column or set of columns, add a new  [auto-increment](https://dev.mysql.com/doc/refman/8.0/en/glossary.html#glos_auto_increment "auto-increment")  column, whose values are filled in automatically.(如果有主键的话, 就把主键作为 clustered index )
    
>  If you do not define a  `PRIMARY KEY`  for your table, MySQL locates the first  `UNIQUE`  index where all the key columns are  `NOT NULL`  and  `InnoDB`  uses it as the clustered index. (如果没有主键, 选择一个unique 的非null 的列作为聚簇索引 )

* Secondary Indexes
> All indexes other than the clustered index are known as [secondary indexes](https://dev.mysql.com/doc/refman/8.0/en/glossary.html#glos_secondary_index "secondary index").(非聚簇索引都是二级索引)
> In `InnoDB`, each record in a secondary index contains the primary key columns for the row, as well as the columns specified for the secondary index. (每一条在secondary index 的记录都包含主键, 所以主键尽可能短)

> 聚簇索引和非聚簇索引的区别

聚簇索引数据访问更快, row 直接保存在索引的叶子节点; 非聚簇索引查找行, 需要两次查找, 因为非聚簇索引的叶子节点保存的是主键列, 再通过zhujian

#### hash index
> only the Memory storage engine supports explicit hash indexes. They are
the default index type for Memory tables, though Memory tables can have B-Tree indexes, too.
 ![enter image description here](https://drive.google.com/uc?id=1MWG_sNbdCIJ5SodnSlhs5k1lNgHU0i9J)

* 缺点
1. 不支持row 排序, 因为只是hash value是排序的
2. 不支持部分列匹配, 因为hash value是根据所有列的值计算的, 如果index : (A,B), 只指定where A=someVal , 是不起作用的
3. 不支持范围查询
4. 当hash entry 冲突很多的时候, 冲突的值会退化成列表

#### 慢查询优化
用explain 语句查看执行计划, 目标是降低 rows
https://dev.mysql.com/doc/refman/5.5/en/explain-output.html

1. 先设置sql_no_cache , 看查询是否真的很慢
2. 把where 条件应用到表中, 从返回记录数最小的表开始查起
3. order by limit 形式的sql语句让排序的表优先查


#### 问题
* innodb 的文件结构
* mysql 几种索引
* mysql 语句执行的大致流程
* mysql 的B+ 树索引如何跟磁盘页对齐, 如何配置
* 全文索引



> Written with [StackEdit](https://stackedit.io/).
<!--stackedit_data:
eyJoaXN0b3J5IjpbLTE4MTIzNDEzOTcsLTYxOTc4MjY2Nyw2Mj
U3NDY3NDAsNTg0MDcwNDM3LC03Njg4MzQ0MjQsLTk2MjMwODcs
MTIzNTAxODAxNSwxNTgyODc3ODA0LDMyNTY4NjAzMyw1NDg3OT
YyNTQsMjA4NzA2NzcyNiwtMjEwNDQ0NjYxMSwxOTEzMjM2Njcx
LDE5ODE1MTgzMDksNTc4NTkwOTEwLC02MjgwMTg1MTIsLTE0NT
Q2MDU2NTgsMTU5MjQ1NjE4MCw3MjQ4MTkzODcsOTA5OTIwNjcw
XX0=
-->