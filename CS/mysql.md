# mysql innodb 可以放多少数据, 比如三层innodb 可以放多少数据, 需要查询几次page
假设一条数据 1KB, 主键是bigint 8byte
一个page 默认16KB, 有1KB 元数据, 其他15KB 用来放数据, 每个叶子结点可以放 15KB /1KB = 15 条数据, 
非叶子结点需要放主键和page number, 8 byte + 4 byte = 12 byte, 那么一个非叶子结点可以存多少条引用数据呢, 15KB / 12 byte = 1280, 

那么两层innodb可以存放的行数: 1280 * 15 = 19200 (两万条), 需要查两次page
三层: 1280 * 1280 * 15 = 2450 (万条), 需要查三次page. 
那么此时需要几次磁盘IO 取决于内存(InnoDB buffer size) 可以放下多少索引, 以及有多少个表, 和多少个索引列, 
两层innodb 的大小: 1280 * 16 KB + 16 KB 约等于 16MB, 
三层innodb 的大小: 1280 * 1280 * 16 KB + 1280 * 16 KB + 16 约等于 25GB,
那么通常来说三层innodb 不能完全放到内存里, 就需要一次磁盘IO. 

下图是三层两阶的B+ 树图示: 
![image](https://github.com/Knight-Wu/articles/assets/20329409/20cb6f6c-fbbb-4fdf-9c24-2a5b082fb368)
查找过程是根据id 在page 中进行二分查找, 然后找到page number 相当于地址, 再找下层page , 直到找到叶子结点, 如果是非聚簇索引就先用索引列找到主键, 再根据主键去找行数据. 

下图是page 的结构图示: 
![image](https://github.com/Knight-Wu/articles/assets/20329409/5df9f864-a7b2-47a7-b89d-5501351ce0fa)

* 如何找到根页呢

其实每张表的根页位置在表文件中是固定的，即page number=3的页, 等于告诉了你根页在表数据文件的offset. 

# 构造死锁sql
其实就是两个事务, 每个事务两条sql, 第二条sql 等待其他事务的第一条sql 释放, 互相等待就发生了死锁. 
```
-- 事务1
begin;
-- SQL1更新id为1的
update user set age = 1 where id = 1;
-- SQL2更新id为2的
update user set age = 2 where id = 2;
commit;
复制代码
-- 事务2
begin;
-- SQL1更新id为2的
update user set age = 3 where id = 2;
-- SQL2更新id为1的
update user set age = 4 where id = 1;
commit;
```
过程是先执行事务1 sql 1, 事务2 的sql 1, 再执行事务1 的sql 2 此时发生了等待, 因为是修改, 独占锁住了同一行, 然后再执行事务2 的sql 2 就会死锁. 
事务1的commit 成功，再查看数据，id为1和2的age字段分别被修改为了1和2，即事务1执行成功。
事务2即使再执行commit数据也不会发生变化，因为事务2报错终止操作被回滚了。

* java 程序模拟数据库死锁
```
通过Java操作数据库，模拟在实际应用中的数据库死锁。
首先是第一个业务方法，其实和上面用SQL模拟死锁的思路是一样的，这里的业务也很简单，先更新id为1的，再更新id为2的
@Transactional(rollbackFor = Exception.class)
public void updateById() {
    User record1 = new User();
    record1.setId(1);
    record1.setAge(1);
    userMapper.updateByPrimaryKey(record1);
    System.out.println("事务1 执行第一条SQL完毕");

    User record2 = new User();
    record2.setId(2);
    record2.setAge(2);
    userMapper.updateByPrimaryKey(record1);
    System.out.println("事务1 执行第二条SQL完毕");
}
复制代码
然后第二个业务方法，同样，模拟上面的SQL死锁，先更新id为2的，然后为了使这个先后顺序更加明显，效果更突出，我们让第二个业务方法休眠30毫秒，再更新id为1的
@Transactional(rollbackFor = Exception.class)
public void updateById1() {
    User record1 = new User();
    record1.setId(2);
    record1.setAge(3);
    userMapper.updateByPrimaryKeySelective(record1);
    System.out.println("事务2 执行第一条SQL完毕");
    //休眠，保证先后执行顺序
    try {
        Thread.sleep(30);
    } catch (InterruptedException e) {
        e.printStackTrace();
    }
    User record2 = new User();
    record2.setId(1);
    record2.setAge(4);
    userMapper.updateByPrimaryKeySelective(record2);
    System.out.println("事务2 执行第二条SQL完毕");
}
复制代码
然后我们进行单元测试，开两个线程，模拟多个用户请求，触发不同的业务操作数据库
@Test
public void testDeadLock() {
    new Thread(() -> {
      userService.updateById(); 
      System.out.println("事务1 执行完毕");
    }).start();

    new Thread(() -> {
      userService.updateById1(); 
      System.out.println("事务2 执行完毕");
    }).start();
    Thread.sleep(2000);//休眠，等待两个线程，确保都能执行
}

```
## high performance mysql 3rd notes
# 6. query performance optimization
## 查询性能的度量
In MySQL, the simplest query cost metrics are:
 Response time, Number of rows examined, Number of rows returned
* response time 
包括服务时间和排队时间

In general, MySQL can apply a WHERE clause in three ways, from best to worst:
• Apply the conditions to the index lookup operation to eliminate nonmatching rows. This happens at the storage engine layer.
• Use a covering index (“Using index” in the Extra column) to avoid row accesses, and filter out nonmatching rows after retrieving each result from the index. This happens at the server layer, but it doesn’t require reading rows from the table.
• Retrieve rows from the table, then filter nonmatching rows (“Using where” in the Extra column). This happens at the server layer
# 分解关联查询
在应用程序层面关联多个表的查询, 而不是在mysql 层面直接关联

# 如何衡量查询时间
* information_schema
https://dev.mysql.com/doc/refman/8.0/en/performance-schema-query-profiling.html

* 开启慢查询日志

# mysql 查询过程
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

## 优化关联查询
确保关联的列上有索引, 当表A和表B 在列c上关联时, 只需要在第二个表, 表b 上的列c 建索引, 表A 就不需要了.
## 优化limit
在limit 性能低底下的时候加上索引
## 优化union 查询
除非必要, 不要用union, 用union all(不排除重复的)
#  存储引擎
## InnoDB 
是mysql 默认的事务型引擎, 支持事务, 支持行锁, 支持崩溃后自动恢复, 基于聚簇索引, 
https://juejin.im/post/5b1685bef265da6e5c3c1c34
### innodb file structure
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


# 索引
## B+ tree vs skip list
总结来说B+ tree 是对磁盘文件的索引，树高越低，IO次数越低，相比跳表树高低很多，查询会快，但是写入需要page 的合并和更新以及分裂，写一条数据可能需要重写整个page，存在写放大。随着写入进行会进行page 的分裂和合并，
产生随机IO。

B+树是多叉树结构，每个结点都是一个16k的数据页，能存放较多索引信息，所以扇出很高。三层左右就可以存储2kw左右的数据, 也就是说查询一次数据，如果这些数据页都在磁盘里，那么最多需要查询三次磁盘IO。

跳表是链表结构，一条数据一个结点，如果最底层要存放2kw数据，且每次查询都要能达到二分查找的效果，2kw大概在2的24次方左右，所以，跳表大概高度在24层左右。最坏情况下，这24层数据会分散在不同的数据页里，也即是查一次数据会经历24次磁盘IO。

因此存放同样量级的数据，B+树的高度比跳表的要少，如果放在mysql数据库上来说，就是磁盘IO次数更少，因此B+树查询更快。

而针对写操作，B+树需要拆分合并索引数据页，跳表则独立插入，并根据随机函数确定层数，没有旋转和维持平衡的开销，因此跳表的写入性能会比B+树要好。
# INNODB 索引


## B+ tree index structure in INNODB 

![enter image description here](https://drive.google.com/uc?id=1jOIFUv2qT3d__lWSkkqsfuff2_N7LDoK)

* leaf node size=page size=16KB , 可以在初始化mysql instance 的时候进行配置
* 所有数据都由叶子节点保管
* Because B+Trees store the indexed columns in order, they’re useful for searching for ranges of data.(叶子节点的记录持有一个指向下一条记录的指针, 保存着下条记录在这个page 的offset; 记录间物理上并不是顺序排列的)
* primary key value 是存在非叶子节点的, 如果过长, 非叶子节点的记录数就会减少, 索引的结构会更大. 

* B+ tree and B tree difference
B+ 树把所有的 val 都存储在叶子节点, B 树把 val 跟 key 一起保存, 叶子节点和中间节点都有, 所有 B + 树的中间节点更小, 假设每次从磁盘拿取的数据大小相同, 那么 b+ 树数据的 key 更多, 范围更大, 更容易命中, 那么对应的 IO 次数就会更少; 而且因为 B+ 树叶子节点像一个链表, 而且所有 val 都在叶子节点, 非常方便做范围查找, 而不像 B 树做范围查找相当于一个深度优先搜索, IO 次数更多, 肯定测试对比过 B+ 树的平均查询性能更好, 而且数据都在叶子节点利于压缩, 可以用字符串前缀压缩和 整型的 delta 压缩等, 

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

* 多列索引

可以认为是多个列的值拼成一个值作为这个索引的值, 所以必须要精确匹配并查找到索引的值才行, 所以就有了类似最左匹配原则等. 
https://dev.mysql.com/doc/refman/8.0/en/multiple-column-indexes.html

例如
```
CREATE TABLE test (
    id         INT NOT NULL,
    last_name  CHAR(30) NOT NULL,
    first_name CHAR(30) NOT NULL,
    PRIMARY KEY (id),
    INDEX name (last_name,first_name)
);

SELECT * FROM test WHERE last_name='Jones';

SELECT * FROM test
  WHERE last_name='Jones' AND first_name='John';

SELECT * FROM test
  WHERE last_name='Jones'
  AND (first_name='John' OR first_name='Jon');

SELECT * FROM test
  WHERE last_name='Jones'
  AND first_name >='M' AND first_name < 'N';

前四个查询可以匹配索引, 后面查询不能匹配. 就是因为要能完全按照索引列的顺序拼出索引的值.

SELECT * FROM test WHERE first_name='John';

SELECT * FROM test
  WHERE last_name='Jones' OR first_name='John';
```

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
意思是, 直接从索引就返回需要查询的数据了, 并不需要再次查询具体的表数据, 直接根据secondary index 返回索引列和主键. 并不需要二次查询主键. 如下图
![enter image description here](https://drive.google.com/uc?id=1z1I_cejEGOnx70mEMPEB6724PqfTKy8_)

* 好处
1. 能减少数据量, 减少随机IO


* 使用索引来排序
最好设计索引的时候覆盖查询和排序两种任务, 只有当索引列的顺序和order by 的列顺序一致时, 且所有列的排序方向也跟索引是一致时(索引是正序, order by 也是正序), 具体参考"高性能 mysql 5.3.7"
 
* 多列索引和多个单列索引
最好能建成多列索引, 能匹配的查询更多, 多个单列索引在某些情况下, 会经过index merge 优化, 但是优化的成本会比较高, 所以最好能建成多列索引

* 为什么选择性低的列不适合做索引
因为除非是聚簇索引, 直接能返回数据,  不然查索引要多消耗 IO , 再去扫描表, 有可能还不如直接去扫描表. 




## hash index
> only the Memory storage engine supports explicit hash indexes. They are
the default index type for Memory tables, though Memory tables can have B-Tree indexes, too.
 ![enter image description here](https://drive.google.com/uc?id=1MWG_sNbdCIJ5SodnSlhs5k1lNgHU0i9J)

* 缺点
1. 不支持row 排序, 因为只是hash value是排序的
2. 不支持部分列匹配, 因为hash value是根据所有列的值计算的, 如果index : (A,B), 只指定where A=someVal , 是不起作用的
3. 不支持范围查询
4. 当hash entry 冲突很多的时候, 冲突的值会退化成列表

# 慢查询优化
用explain 语句查看执行计划, 目标是降低 rows
https://dev.mysql.com/doc/refman/5.5/en/explain-output.html

1. 先设置sql_no_cache , 看查询是否真的很慢
2. 把where 条件应用到表中, 从返回记录数最小的表开始查起
3. order by limit 形式的sql语句让排序的表优先查

# mysql explain
主要是优化查询的行数

* join type
https://dev.mysql.com/doc/refman/8.0/en/explain-output.html
 EXPLAIN Join Types 对查询的友好程度由高到低
 const 可以理解为利用primary key, 或unique index 查询单表一行数据
 eq_ref 可以理解为多表一行
 ref 可以理解为多表join 多行,uses only a leftmost prefix of the key or if the key is not a `PRIMARY KEY` or `UNIQUE` index
* extra
using_index 和using_where 区别: https://stackoverflow.com/questions/25672552/whats-the-difference-between-using-index-and-using-where-using-index-in-the

# ACID
## atomicity
原子性, 事务执行的要么成功要么失败, 没有第三个状态

## consistency
一致性, 什么样的结果是一致的, 需要由应用程序去定义, 除非数据库是一个强读写一致的系统, 否则数据库的在一致性方面的表现是由应用程序去决定的. 

## Isolation
隔离性, 事务有不同的隔离级别, 理想情况不能互相影响, 事务感知不到其他事务的存在, 相当于串行执行, 但是需要一些手段去保障, 所以需要牺牲一点性能

## durability
持久性, 一旦事务提交成功可以认为已经持久化到了数据库, 但是能容忍什么故障或者多少故障, 由副本数以及副本间的同步协议等特性去决定的

# 事务隔离级别（定义了一个事务可能受其他并发事务影响的程度）
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
    发生在一个事务的进行两次读取时, 可能会读取到不同的数据, 就是说在 A 事务在 t1 开始后, 进行了两次读取, 其中第二次读取, 读到了 B 事务在 t2 修改的数据, t2>t1. 
    
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

**这个对隔离级别解释更详细 : 解释了 MVCC 和 undo log** https://blog.csdn.net/qq_35190492/article/details/109044141

# mysql 悲观锁和乐观锁
https://blog.csdn.net/puhaiyang/article/details/72284702
悲观锁就是先加锁再查询, 分为共享锁和互斥锁
乐观锁是用版本号, 写操作的时候要判断此时的版本号是否和之前最新的版本号一致.
# 资料
https://blog.jcole.us/innodb/
relational database index design and the optimizers

# int(3) vs int(4)
跟storage size 并没有关系, 仍然需要4 bytes, 
但是若指定了ZERO_FILL, 则不足三位会在前面补0, 

   if the stored value has less digits than  _x_,  `ZEROFILL`  will prepend zeros.
    
    > **INT(5) ZEROFILL**  with the stored value of 32 will show  **00032**  
    > **INT(5)**  with the stored value of 32 will show  **32**  
    > **INT**  with the stored value of 32 will show  **32**
    
  if the stored value has more digits than  _x_, it will be shown as it is.
    
    > **INT(3) ZEROFILL**  with the stored value of 250000 will show  **250000**  
    > **INT(3)**  with the stored value of 250000 will show  **250000**  
    > **INT**  with the stored value of 250000 will show  **250000**

* varchar(3) vs varchar(4)
并不影响storage size, storage size 只跟存储的具体string 有关, 若是单字节编码, 需要用一个字节或两个字节来存储长度, max len <= 65535 bytes, , 但是 varchar(3) 和 varchar(4) 查询中使用的内存并不一样, 后者让查询引擎使用更多的内存, 所以需要合理计算 varchar(x), string 长度大于 x 会截断
[https://stackoverflow.com/questions/1151667/what-are-the-optimum-varchar-sizes-for-mysql](https://stackoverflow.com/questions/1151667/what-are-the-optimum-varchar-sizes-for-mysql)

* varchar vs char
[https://dba.stackexchange.com/questions/2640/what-is-the-performance-impact-of-using-char-vs-varchar-on-a-fixed-size-field/2643#2643](https://dba.stackexchange.com/questions/2640/what-is-the-performance-impact-of-using-char-vs-varchar-on-a-fixed-size-field/2643#2643)

1. char 是定长, 在其他情况相同的时候, char 用于做索引会快 20% 
> The book MySQL Database Design and Tuning performed something marvelous on a MyISAM table to prove this

2. varchar 是变长, 如果实际存储的不是定长string, 那么肯定会更省空间. 
3. ALTER TABLE tblname ROW_FORMAT=FIXED;
让varchar 表现得跟char 一样, index 速度会加快, 但是存储的size 会增加不少

* varchar vs text

[https://stackoverflow.com/questions/25300821/difference-between-varchar-and-text-in-mysql](https://stackoverflow.com/questions/25300821/difference-between-varchar-and-text-in-mysql)
两个都是变长的存储方式, 但是因为mysql max row size 是 65535, 所以当需要存储很大的string 的时候最好用text, 因为他存储的是string 的ref, 只需要9-12 bytes, 
Reasons to use  `TEXT`:

-   If you want to store a paragraph or more of text
-   If you don't need to index the column
-   If you have reached the row size limit for your table

Reasons to use  `VARCHAR`:

-   If you want to store a few words or a sentence
-   If you want to index the (entire) column
-   If you want to use the column with foreign-key constraints



# mysql COLLATE
对于mysql中那些字符类型的列，如`VARCHAR`，`CHAR`，`TEXT`类型的列，都需要有一个`COLLATE`类型来告知mysql如何对该列进行排序和比较。简而言之，**COLLATE会影响到ORDER BY语句的顺序，会影响到WHERE条件中大于小于号筛选出来的结果，会影响**`**DISTINCT**`**、**`**GROUP BY**`**、**`**HAVING**`**语句的查询结果**。另外，mysql建索引的时候，如果索引列是字符类型，也**会影响索引创建**，

这是mysql的一个遗留问题，mysql中的`utf8`最多只能支持3bytes长度的字符编码，对于一些需要占据4bytes的文字，mysql的`utf8`就不支持了，要使用`utf8mb4`才行。

很多`COLLATE`都带有`_ci`字样，这是Case Insensitive的缩写，即大小写无关，也就是说"A"和"a"在排序和比较的时候是一视同仁的。对于那些`_cs`后缀的`COLLATE`，则是Case Sensitive，即大小写敏感的。

> Written with [StackEdit](https://stackedit.io/).
<!--stackedit_data:
eyJoaXN0b3J5IjpbLTE5NjYyNDcwNDVdfQ==
-->

# 问题
* 全文索引
> Written with [StackEdit](https://stackedit.io/).
<!--stackedit_data:
eyJoaXN0b3J5IjpbMTk2MjE3Njc1OSwxOTc2ODUyNTQ5LC0xOD
g2NTE0MzksLTk2ODk4Mjc4LDg5MjU1ODg0OV19
-->
