## high performance mysql 3rd notes
### 6. query performance optimization
#### 查询性能的度量
In MySQL, the simplest query cost metrics are:
 Response time, Number of rows examined, Number of rows returned
* response time 
包括服务时间和排队时间

In general, MySQL can apply a WHERE clause in three ways, from best to worst:
• Apply the conditions to the index lookup operation to eliminate nonmatching rows. This happens at the storage engine layer.
• Use a covering index (“Using index” in the Extra column) to avoid row accesses, and filter out nonmatching rows after retrieving each result from the index. This happens at the server layer, but it doesn’t require reading rows from the table.
• Retrieve rows from the table, then filter nonmatching rows (“Using where” in the Extra column). This happens at the server layer
#### 分解关联查询
在应用程序层面关联多个表的查询, 而不是在mysql 层面直接关联

#### 如何衡量查询时间
* information_schema
https://dev.mysql.com/doc/refman/8.0/en/performance-schema-query-profiling.html

* 开启慢查询日志

#### mysql 查询过程
1. 客户端发送查询语句到服务器
2. 服务器查询缓存, 缓存命中直接返回, 否则进入3
3. 服务器进行sql 解析和预处理, 再由优化器生成对应的执行计划
解析器将sql 解析成一颗语法树, 并检查语法错误
预处理器检查sql 的表和列是否存在等.
优化器将sql 转化为最佳的查询计划, 
4. 根据执行计划调用存储引擎的api 来完成查询
5. 查询完毕后, 就将结果逐条推送并批量的推送给客户端, 并缓存在客户端本地. 

* 服务器与客户端的通信协议
半双工的, 某一时刻只有一方能发送数据, 另一方在等待

* 优化器的策略
> 优化count, min, max

例如在B+ tree 索引中, min 对应索引的第一条记录

> 预估并转化为常数

> 使用 straight_join 关键字来比较原执行计划和优化后的执行计划

* mysql 如何执行关联查询
![enter image description here](https://drive.google.com/uc?id=1ENqjSNgGGsiCMbk1Q2FQ6A6SadjXwQvM)

#### 优化关联查询
确保关联的列上有索引, 当表A和表B 在列c上关联时, 只需要在第二个表, 表b 上的列c 建索引, 表A 就不需要了.
#### 优化limit 
在limit 性能低底下的时候加上索引
#### 优化union 查询
除非必要, 不要用union, 用union all(不排除重复的)
####  存储引擎
#### InnoDB 
是mysql 默认的事务型引擎, 支持事务, 支持行锁, 支持崩溃后自动恢复, 基于聚簇索引, 
https://juejin.im/post/5b1685bef265da6e5c3c1c34
#### innodb file structure
* space
![enter image description here](https://drive.google.com/uc?id=1vNYq2tfqF9cXoIu6RkVMrbERRjManKDT)
an .ibd file for each MySQL table, 代表了一个space, 由一个32 bit 的space id 确定. 由多个pages 组成, 最多是2的32次方的page,  For more efficient management, pages are grouped into blocks of 1 MiB (64 contiguous pages with the default page size of 16 KiB), and  called an **“extent”**

对于每256 个extents, space 需要有一个page 来store bookkeeping information. 
> An FSP_HDR page only has enough space internally to store bookkeeping information for 256 extents (or 16,384 pages, 256 MiB), so additional space must be reserved every 16,384 pages for bookkeeping information in the form of an XDESpage.

* page
![enter image description here](https://drive.google.com/uc?id=1LAmNPpwGYIrfjgg0X8RV9kVn_Q0jXcLA)

![enter image description here](https://drive.google.com/uc?id=1yIYifmSdffYSZn2Ojms8fZ2LvxeSkLv1)
Each page within a space is assigned a 32-bit integer page number, often called “offset”, which is actually just the page’s offset from the beginning of the space (not necessarily the file, for multi-file spaces). So, page 0 is located at file offset 0, page 1 at file offset 16384=16*1024 bytes, and so on. (The astute may remember that InnoDB has a limit of 64TiB of data; this is actually a limit per space, and is due primarily to the page number being a 32-bit integer combined with the default page size: 2的32次方 x 16 KiB = 64 TiB.


#### MyISAM
不支持行锁, 只支持表锁, 不支持事务, 不支持崩溃后快速回复, 不支持外键, 适合读的场景


### 索引

#### INNODB 索引


* B+ tree index structure in INNODB 

![enter image description here](https://drive.google.com/uc?id=1jOIFUv2qT3d__lWSkkqsfuff2_N7LDoK)

* leaf node size=page size=16KB , 可以在初始化mysql instance 的时候进行配置
* 所有数据都由叶子节点保管
* Because B+Trees store the indexed columns in order, they’re useful for searching for ranges of data.(叶子节点的记录持有一个指向下一条记录的指针, 保存着下条记录在这个page 的offset; 记录间物理上并不是顺序排列的)
* primary key value 是存在非叶子节点的, 如果过长, 非叶子节点的记录数就会减少, 索引的结构会更大. 


> how to search quickly between records in b+ tree node
https://blog.jcole.us/2013/01/14/efficiently-traversing-innodb-btrees-with-the-page-directory/

因为record 是以链表的形式, 逻辑连接, 遍历列表是很慢的,  use page directory(可以理解是用一个数组来保存一个范围内key, 二分查找这个数组, 找到满足要求的key 再去线性查找. 
![enter image description here](https://drive.google.com/uc?id=1DRE4r96br10emG6XHAZ_5wkQ5xf2e3yR)


1.  Start at the root page of the index.
2.  Binary search using the page directory (repeatedly splitting the directory in half based on whether the current record is greater than or less than the search key) until a record is found via the page directory with the highest key that does not exceed the search key.
3.  Linear search from that record until finding an individual record with the highest key that does not exceed the search key. If the current page is a leaf page, return the record. If the current page is a non-leaf page, load the child page this record points to, and return to step 2.


一个实际的数据例子阐述innodb 索引: 
![enter image description here](https://drive.google.com/uc?id=1CCbvzgDAKugLkRhRx-7d7kg1bLVsLWL2)

根据以上结构, 支持全值匹配, 最左匹配, 范围匹配, 而且因为index value 也是排序的, 所以也优化了 order by, 
* 缺点
1. 不支持列顺序混乱的匹配, 例如index: (A,B,C), 查询顺序是(A,C,B), 只支持到A 列, 因为不知道B 列的长度.
2. 不支持跳列, 必须指定前几个列的值. 
3. if your query is WHERE last_name="Smith" AND first_name LIKE 'J%' AND dob='1976-12-23' , the index access will use only the first two columns in the index, because the LIKE is a range condition

> 使用B+ tree 索引的原则

1. 最左前缀匹配原则, 碰到范围查询(>、<、between、like) 就终止匹配, 比如a = 1 and b = 2 and c > 3 and d = 4 如果建立(a,b,c,d)顺序的索引，d是用不到索引的，如果建立(a,b,d,c)的索引则都可以用到，a,b,d 作为查询条件, and 连接的列顺序可以交换。但是跳列和非and 情况下列顺序混乱都是用不了索引的. 
https://dev.mysql.com/doc/refman/8.0/en/multiple-column-indexes.html
 
2. (a,b,c) 的联合索引，可以用col in (val) 来使后面的索引列仍然有效, 例如
where a="A" and b in ('b','B') and c = 'C' , (a,b,c) 的索引仍然有效. 

3. 当不需要排序和分组的时候, 将选择性高的列放到多列索引的前边

4. 索引列的值不能作为函数入参进行计算, 比如from_unixtime(create_time) = ’2014-05-29’就不能使用到索引，原因很简单，b+树中存的都是数据表中的字段值，但进行检索时，需要把所有元素都应用函数才能比较，显然成本太大。所以语句应该写成create_time = unix_timestamp(’2014-05-29’)
5. 前缀索引
 只采取索引列的前缀作为索引, 减少空间, 但是这个前缀要能一定程度上的获取你想要的结果. 可以具体参考"高性能mysql 5.3.2"
6. 多个单列索引, 可能会引起索引的合并, 

* clustered index
聚簇索引并不是一个索引类型, 而是一种数据存储方式, 而InnoDB 的聚簇索引实际上就是B+ tree 的叶子节点将key 和data row 存放在一起. 

>  When you define a  `PRIMARY KEY`  on your table,  `InnoDB`  uses it as the clustered index. Define a primary key for each table that you create. If there is no logical unique and non-null column or set of columns, add a new  [auto-increment](https://dev.mysql.com/doc/refman/8.0/en/glossary.html#glos_auto_increment "auto-increment")  column, whose values are filled in automatically.(如果有主键的话, 就把主键作为 clustered index )
    
>  If you do not define a  `PRIMARY KEY`  for your table, MySQL locates the first  `UNIQUE` and `NOT NULL`  col  as the clustered index. (如果没有主键, 选择一个unique 的index 作为聚簇索引,  要求该索引的所有key col 都要求非 null) and failing that, a 48-bit hidden “Row ID” field is automatically added to the table structure and used as the primary key. _Always_ add a primary key yourself. The hidden one is useless to you but still costs 6 bytes per row.

* Secondary Indexes
> All indexes other than the clustered index are known as [secondary indexes](https://dev.mysql.com/doc/refman/8.0/en/glossary.html#glos_secondary_index "secondary index").(非聚簇索引都是二级索引)
> In `InnoDB`, each record in a secondary index contains the primary key columns for the row, as well as the columns specified for the secondary index. (每一条在secondary index 的记录都包含主键, 所以主键尽可能短)

> 聚簇索引和非聚簇索引的区别

1. 聚簇索引数据访问更快, row 直接保存在索引的叶子节点; 非聚簇索引查找行, 需要两次查找, 因为非聚簇索引的叶子节点保存的是主键列, 再通过主键列去聚簇索引中找到对应的行. 
2. 聚簇索引的更新代价更大, 
3. 因为聚簇索引的是按顺序排列的, 如果是主键作为了聚簇索引, 则按主键进行排序, 主键最好设置为AUTO_INCREMENT, 插入的效率会很高, 否则如果是使用uuid 作为主键, 可能引起page split, 因为后面的插入的记录可能在之前形成的索引的page 中间, 引起page split; 但是设置为自增, 插入的热点集中在主键的上界, 引起gap lock的竞争.

* 覆盖索引
这是B+ tree 索引的在某些情况下的一个特性, 直接从索引就返回需要查询的数据了, 并不需要再次查询具体的表数据, 在这种情况下, 可以称为covering index, 而且在B+ tree 聚簇索引的情况下, 特别适用, 直接根据secondary index 返回索引列和主键. 并不需要二次查询主键. 如下图
![enter image description here](https://drive.google.com/uc?id=1z1I_cejEGOnx70mEMPEB6724PqfTKy8_)

* 好处
1. 能减少数据量, 减少随机IO


* 使用索引来排序
最好设计索引的时候覆盖查询和排序两种任务, 只有当索引列的顺序和order by 的列顺序一致时, 且所有列的排序方向也跟索引是一致时(索引是正序, order by 也是正序), 具体参考"高性能 mysql 5.3.7"
 
* 多列索引和多个单列索引
最好能建成多列索引, 能匹配的查询更多, 多个单列索引在某些情况下, 会经过index merge 优化, 但是优化的成本会比较高, 所以最好能建成多列索引

* 为什么选择性低的列不适合做索引
因为除非是聚簇索引, 直接能返回数据,  不然查索引要多消耗 IO , 再去扫描表, 有可能还不如直接去扫描表. 




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

#### mysql explain
主要是优化查询的行数

* join type
https://dev.mysql.com/doc/refman/8.0/en/explain-output.html
 EXPLAIN Join Types 对查询的友好程度由高到低
 const 可以理解为利用primary key, 或unique index 查询单表一行数据
 eq_ref 可以理解为多表一行
 ref 可以理解为多表join 多行,uses only a leftmost prefix of the key or if the key is not a `PRIMARY KEY` or `UNIQUE` index
* extra
using_index 和using_where 区别: https://stackoverflow.com/questions/25672552/whats-the-difference-between-using-index-and-using-where-using-index-in-the


####  事务隔离级别（定义了一个事务可能受其他并发事务影响的程度）
> 数据并发问题

* 脏读(dirty read), A事务读到了B事务尚未提交的数据, 可理解为读到了脏数据, 若此时B事务回滚, 则会产生数据不一致的情况.

```
Transaction 1	                                          Transaction 2
/* Query 1 */
SELECT age FROM users WHERE id = 1;
/* will read 20 */
                                                        /* Query 2 */
                                                        UPDATE users SET age = 21 WHERE id = 1;
                                                        /* No commit here */
/* Query 1 */
SELECT age FROM users WHERE id = 1;
/* will read 21 */
                                                        ROLLBACK; /* lock-based DIRTY READ */

```

* 不可重复读(unrepeatable read)
    发生在一个事务的进行两次读取时, 可能会读取到不同的数据, 此时select 是不需要锁的, 或者锁还没来得及加上, 就发生了读取
    
 ```

Transaction 1	                                    Transaction 2
/* Query 1 */
SELECT * FROM users WHERE id = 1;
                                                    /* Query 2 */
                                                    UPDATE users SET age = 21 WHERE id = 1;
                                                    COMMIT; 
                                                    /* in multiversion concurrency
                                                    control, or lock-based READ COMMITTED */
/* Query 1 */
SELECT * FROM users WHERE id = 1;
COMMIT; 
/* lock-based REPEATABLE READ */

```

   根据不同的隔离级别, 会有不同结果
> At the SERIALIZABLE and REPEATABLE READ isolation levels, the DBMS must return the old value for the second SELECT. At READ COMMITTED and READ UNCOMMITTED, the DBMS may return the updated value; this is a non-repeatable read.
    
* 有两种策略来处理此种情况
    1. 延迟事务2的执行直到事务1提交或者回滚, 等于说维护一个时序: serial schedule T1, T2. 
    2. 为了获取更好的性能,让事务2被先提交, 事务1的执行必须满足时序 T1, T2, 若不满足事务1被回滚.

在不同模型下, 结果会不一样
> Using a lock-based concurrency control method, at the REPEATABLE READ isolation mode, the row with ID = 1 would be locked, thus blocking Query 2 until the first transaction was committed or rolled back. In READ COMMITTED mode, the second time Query 1 was executed, the age would have changed.

> Under multiversion concurrency control, at the SERIALIZABLE isolation level, both SELECT queries see a snapshot of the database taken at the start of Transaction 1. Therefore, they return the same data. However, if Transaction 1 then attempted to UPDATE that row as well, a serialization failure would occur and Transaction 1 would be forced to roll back.

> At the READ COMMITTED isolation level, each query sees a snapshot of the database taken at the start of each query. Therefore, they each see different data for the updated row. No serialization failure is possible in this mode (because no promise of serializability is made), and Transaction 1 will not have to be retried.(这段怎么跟之前说的 read commited 不一样 ? )

* 幻象读(Phantom reads),是不可重复读的特例, 相当于是范围读.

```
    Transaction 1	                                        Transaction 2
/* Query 1 */
SELECT * FROM users
WHERE age BETWEEN 10 AND 30;
                                                /* Query 2 */
                                                INSERT INTO users(id,name,age) VALUES ( 3, 'Bob', 27 );
                                                COMMIT;
/* Query 1 */
SELECT * FROM users
WHERE age BETWEEN 10 AND 30;
COMMIT;


```
> Note that Transaction 1 executed the same query twice. If the highest level of isolation were maintained, the same set of rows should be returned both times, and indeed that is what is mandated to occur in a database operating at the SQL SERIALIZABLE isolation level. However, at the lesser isolation levels, a different set of rows may be returned the second time.

> In the SERIALIZABLE isolation mode, Query 1 would result in all records with age in the range 10 to 30 being locked, thus Query 2 would block until the first transaction was committed. In REPEATABLE READ mode, the range would not be locked, allowing the record to be inserted and the second execution of Query 1 to include the new row in its results.
 
* 丢失更新(lost update)
指在一个事务读取一个数据时，另外一个事务也访问了该数据，那么在第一个事务中修改了这个数据后，第二个事务也修改了这个数据。这样第一个事务内的修改结果就被丢失，因此称为丢失修改。
    
* 以上均参见[wiki isolation](https://en.wikipedia.org/wiki/Isolation_(database_systems)#Dirty_reads)
mysql 默认隔离级别是重复读
![enter image description here](https://drive.google.com/uc?id=1NdpnXgkU7Q3TW0G73P0WPR_ejiUgf-Qp)



#### mysql 悲观锁和乐观锁
https://blog.csdn.net/puhaiyang/article/details/72284702
悲观锁就是先加锁再查询, 分为共享锁和互斥锁
乐观锁是用版本号, 写操作的时候要判断此时的版本号是否和之前最新的版本号一致.
#### 资料
https://blog.jcole.us/innodb/
relational database index design and the optimizers

#### 问题
* 全文索引
> Written with [StackEdit](https://stackedit.io/).
<!--stackedit_data:
eyJoaXN0b3J5IjpbODkyNTU4ODQ5XX0=
-->