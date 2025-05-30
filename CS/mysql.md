# 数据库选型
其实针对后端常用业务, 就四种, RDBMS 关系型数据库 mysql , NOSQL 分为缓存 redis 和 宽列存储 hbase, Cassandra, 搜索引擎 es.</br> 文档型 MongoDB 和时序数据库, 图数据库都有专门特定的场景. 
* 数据库选型一般来说从读取的角度来考虑, 因为数据写入都是为了查的, 写入你怎么都可以写入, 问题是查能不能满足需求, qps , latency, 是否支持 sql, 事务, 数据一致性
* 再考虑写入的性能, 成本等.  
## 关系型数据库
有事务, ACID, 支持 sql

## NOSQL 数据库
kv, 高吞吐量, 最终一致性, 对 sql 支持不友好或者不是原生支持, 可能不支持事务

### 内存型
用来做缓存, 最终才能和数据库的数据保持一致性 redis, 数据通常不做持久化, 持久化给其他的数据来做, 可以从其他数据库构建缓存数据
 
### 宽列数据库
支持高吞吐量以及高 qps 的读写. 最终一致性, 对 sql 支持不友好, sql 比较慢, 可能不支持事务, 适合列式存储和 kv 查询, 存大量数据, 持久化支持好, hbase, Cassandra

## 搜索引擎
elasticsearch, 用于复杂的搜索场景, 虽然也可以通过 docId 来查某一列的数据, 但是相比 kv 的查询, 灵活性就差了很多, 一般用来做全文搜索, 侧重读体验, 所以索引比较重, 写入性能不高.

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

## 索引失效的情况
![db84afae3224b1292c05b9ea056ad71](https://github.com/user-attachments/assets/01052524-ff62-4b6d-9ae6-ec14e5ef3427)


## B+ tree index structure in INNODB 
* 手画B+ 树

```

                 [5, 10]
             /    |     \
        [1,3]   [6,8,9]   [11,13]
       /  | \    / | | \    /  |  \
叶子节点: 1 2 3 →6 7 8 9→11 12 13→15
（→表示叶子节点间的链表指针）

```

## 查找过程
1. 从根节点开始
输入：目标键值 k（例如 k=8）。

操作：访问根节点 [5, 10]，比较 k 与根节点的键值，确定下一步的子节点。

2. 内部节点路由
比较规则：

若 k < 第一个键值 → 进入最左侧子节点。

若 键值_i ≤ k < 键值_{i+1} → 进入第 i+1 个子节点。

若 k ≥ 最后一个键值 → 进入最右侧子节点。

示例：

k=8 满足 5 ≤ 8 < 10 → 进入中间子节点 [6,8,9]。

3. 递归向下查找
操作：重复步骤 2，直到到达叶子节点。

当前节点 [6,8,9] 是内部节点，继续比较：

k=8 满足 8 ≤ 8 < 9 → 进入第二个子节点（指向键值 8 对应的叶子节点）。

4. 叶子节点搜索
到达叶子节点：[6,7,8,9]（实际存储可能为 6→7→8→9）。

操作：在叶子节点中顺序或二分查找 k=8：


* 下图是三层两阶的B+ 树图示: 
![image](https://github.com/Knight-Wu/articles/assets/20329409/20cb6f6c-fbbb-4fdf-9c24-2a5b082fb368)
</br>
查找过程是根据id 在page 中进行二分查找, 然后找到page number 相当于地址, 再找下层page , 直到找到叶子结点, 如果是非聚簇索引就先用索引列找到主键, 再根据主键去找行数据. 
</br>
* 下图是page 的结构图示: 

![](https://github.com/Knight-Wu/articles/assets/20329409/5df9f864-a7b2-47a7-b89d-5501351ce0fa)
</br>

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

## innodb 索引缺点
1. 不支持列顺序混乱的匹配, 例如index: (A,B,C), 查询顺序是(A,C,B), 只支持到A 列, 因为不知道B 列的长度.
2. 不支持跳列, 必须指定前几个列的值. 
3. if your query is WHERE last_name="Smith" AND first_name LIKE 'J%' AND dob='1976-12-23' , the index access will use only the first two columns in the index, because the LIKE is a range condition


## 多列索引

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

## clustered index(聚簇索引)
聚簇索引并不是一个索引类型, 而是一种数据存储方式, 而InnoDB 的聚簇索引实际上就是B+ tree 的叶子节点将key 和data row 存放在一起. 

>  When you define a  `PRIMARY KEY`  on your table,  `InnoDB`  uses it as the clustered index. Define a primary key for each table that you create. If there is no logical unique and non-null column or set of columns, add a new  [auto-increment](https://dev.mysql.com/doc/refman/8.0/en/glossary.html#glos_auto_increment "auto-increment")  column, whose values are filled in automatically.(如果有主键的话, 就把主键作为 clustered index )
    
>  If you do not define a  `PRIMARY KEY`  for your table, MySQL locates the first  `UNIQUE` and `NOT NULL`  col  as the clustered index. (如果没有主键, 选择一个unique 的index 作为聚簇索引,  要求该索引的所有key col 都要求非 null) and failing that, a 48-bit hidden “Row ID” field is automatically added to the table structure and used as the primary key. _Always_ add a primary key yourself. The hidden one is useless to you but still costs 6 bytes per row.

## Secondary Indexes(非聚簇索引)
> All indexes other than the clustered index are known as [secondary indexes](https://dev.mysql.com/doc/refman/8.0/en/glossary.html#glos_secondary_index "secondary index").(非聚簇索引都是二级索引)
> In `InnoDB`, each record in a secondary index contains the primary key columns for the row, as well as the columns specified for the secondary index. (每一条在secondary index 的记录都包含主键, 所以主键尽可能短)

> 聚簇索引和非聚簇索引的区别

1. 聚簇索引数据访问更快, row 直接保存在索引的叶子节点; 非聚簇索引查找行, 需要两次查找, 因为非聚簇索引的叶子节点保存的是主键列, 再通过主键列去聚簇索引中找到对应的行. 
2. 聚簇索引的更新代价更大, 
3. 因为聚簇索引的是按顺序排列的, 如果是主键作为了聚簇索引, 则按主键进行排序, 主键最好设置为AUTO_INCREMENT, 插入的效率会很高, 否则如果是使用uuid 作为主键, 可能引起page split, 因为后面的插入的记录可能在之前形成的索引的page 中间, 引起页分裂 page split; 但是设置为自增, 插入的热点集中在主键的上界, 可能会引起热点.

## 覆盖索引
意思是, 直接从索引就返回需要查询的数据了, 并不需要再次查询具体的表数据, 例如当只查索引列和主键的时候, 直接根据secondary index 返回索引列和主键. 并不需要二次查询主键的索引, 来定位整行数据. 如下图
![enter image description here](https://drive.google.com/uc?id=1z1I_cejEGOnx70mEMPEB6724PqfTKy8_)

* 覆盖索引好处
1. 能减少数据量, 减少随机IO


## 使用索引来排序
最好设计索引的时候覆盖查询和排序两种任务, 只有当索引列的顺序和order by 的列顺序一致时, 且所有列的排序方向也跟索引是一致时(索引是正序, order by 也是正序), 具体参考"高性能 mysql 5.3.7"
 
## 多列索引和多个单列索引
### 多列索引（联合索引）
适用场景
查询条件包含多个列（AND 条件）

例如：WHERE col1 = A AND col2 = B。

多列索引 (col1, col2) 可高效定位数据，避免多次回表扫描。

* 覆盖索引（Covering Index）

若索引包含查询所需的所有字段, 如非聚簇索引, 只查询索引列和主键, 可直接从索引返回数据，无需访问表数据页。

排序或分组操作

如 ORDER BY col1, col2 或 GROUP BY col1, col2，联合索引能避免额外排序步骤。

* 优势
减少 I/O 次数：单次索引扫描即可满足复合条件。

高效排序/分组：避免临时表或文件排序。

覆盖查询优化：直接通过索引返回数据。

* 缺点
维护成本高：插入、更新时需维护更多列的组合。

灵活性低：若查询条件不固定（如不同列组合），需创建多个联合索引。

#### 最左前缀匹配
若查询仅使用索引的最左前缀（如 WHERE col1 = A），仍可利用索引。但若跳过前缀（如 WHERE col2 = B），索引可能失效。

* 为什么有这个最左前缀匹配规则

  为什么不符合最左匹配不能查找, 因为只有第一列是完全有序的, 后面的列是在前面列的基础上部分有序, 必须要找到第一列, 然后接着找才是有序的, 否则直接找第二列是无序的, 就需要全表扫描, 相当于索引失效
  
##### 举例

```
CREATE TABLE `t1`  (
  `a` int(11) NOT NULL AUTO_INCREMENT,
  `b` int(11) NULL DEFAULT NULL,
  `c` int(11) NULL DEFAULT NULL,
  `d` int(11) NULL DEFAULT NULL,
  `e` varchar(20) CHARACTER SET utf8 COLLATE utf8_general_ci NULL DEFAULT NULL,
  PRIMARY KEY (`a`) USING BTREE,
  INDEX `index_bcd`(`b`, `c`, `d`) USING BTREE
) ENGINE = InnoDB AUTO_INCREMENT = 8 CHARACTER SET = utf8 COLLATE = utf8_general_ci ROW_FORMAT = Dynamic;

```

![image](https://github.com/user-attachments/assets/76121769-23ba-487e-90b3-9f5973b03bcc)
第一行是b, 第二行是c, 第三行是d, 叶子节点最后一行是主键.

select * from T1 where b = 12 and c = 14 and d = 3; </br>
也就是T1表中a列为4的这条记录。存储引擎首先从根节点（一般常驻内存）开始查找，第一个索引的第一个索引列为1,12大于1，第二个索引的第一个索引列为56,12小于56，于是从这俩索引的中间读到下一个节点的磁盘文件地址，从磁盘上Load这个节点，通常伴随一次磁盘IO，然后在内存里去查找。当Load叶子节点的第二个节点时又是一次磁盘IO，比较第一个元素，b=12,c=14,d=3完全符合，于是找到该索引下的data元素即ID值，再从主键索引树上找到最终数据。

![image](https://github.com/user-attachments/assets/36d0f44b-45ca-4ccd-a6b4-48c0deb5cac9)

首先我们创建的index_bcd(b,c,d)索引，相当于创建了(b)、（b、c）（b、c、d）三个索引.
</br>
我们看，联合索引是首先使用多列索引的第一列构建的索引树，用上面idx_t1_bcd(b,c,d)的例子就是优先使用b列构建，当b列值相等时再以c列排序，若c列的值也相等则以d列排序。我们可以取出索引树的叶子节点看一下。
![image](https://github.com/user-attachments/assets/6d5d15c3-f946-4828-aba3-b2f3bdbefd52)
索引的第一列也就是b列可以说是从左到右单调递增的，但我们看c列和d列并没有这个特性，它们只能在b列值相等的情况下这个小范围内递增，

##### 什么情况多列索引失效
联合索引的最左匹配原则会一直向右匹配直到遇到「范围查询」就会停止匹配。也就是范围查询的字段可以用到联合索引，但是在范围查询字段的后面的字段无法用到联合索引。
</br>
select * from T1 where b = 12 and c = 14 and d = 3;-- 全值索引匹配 三列都用到
select * from T1 where b = 12 and c = 14 and e = 'xml';-- 应用到两列索引
select * from T1 where b = 12 and e = 'xml';-- 应用到一列索引
select * from T1 where b = 12  and c >= 14 and e = 'xml';-- 应用到bc两列列索引及索引条件下推优化
select * from T1 where b = 12  and d = 3;-- 应用到一列索引  因为不能跨列使用索引 没有c列 连不上
select * from T1 where c = 14  and d = 3;-- 无法应用索引，违背最左匹配原则

#### 索引下推(Index Condition Pushdown，简称ICP)
索引下推的目的是为了减少回表次数，也就是要减少IO操作。对于InnoDB的聚簇索引来说，数据和索引是在一起的，不存在回表这一说。

引用了子查询的条件不能下推；
引用了存储函数的条件不能下推，因为存储引擎无法调用存储函数。

![image](https://github.com/user-attachments/assets/ddcce5fe-f5ee-4911-8575-db36439ce886)

#### 创建多列索引的原则
建立联合索引时的字段顺序，对索引效率也有很大影响。越靠前的字段被用于索引过滤的概率越高，实际开发工作中建立联合索引时，要把区分度大的字段排在前面，
区分度= distinct(column) / count(*)
### 多个单列索引
适用场景
查询条件独立使用各列（OR 条件或分散查询）

例如：WHERE col1 = A 或 WHERE col2 = B，单列索引 col1 和 col2 可分别被优化器选择。

列选择性差异大

若某列选择性高（唯一值多），单独索引更有效（如用户 ID），而低选择性列（如性别）可能不适用。

频繁更新的列

单列索引维护成本低于多列索引，适合频繁写入的场景。

数据库支持索引合并（Index Merge）

如 MySQL 的 Index Merge 优化，可将多个单列索引的结果合并（如 WHERE col1 = A AND col2 = B）。

## 为什么选择性低的列不适合做索引
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

# 事务
## 事务的特性(ACID)
## atomicity
原子性, 事务执行的要么成功要么失败, 没有第三个状态

## consistency
一致性, 什么样的结果是一致的, 需要由应用程序去定义, 除非数据库是一个强读写一致的系统, 否则数据库的在一致性方面的表现是由应用程序去决定的. 

## Isolation
隔离性, 事务有不同的隔离级别, 理想情况不能互相影响, 事务感知不到其他事务的存在, 相当于串行执行, 但是需要一些手段去保障, 所以需要牺牲一点性能

## durability
持久性, 一旦事务提交成功可以认为已经持久化到了数据库, 但是能容忍什么故障或者多少故障, 由副本数以及副本间的同步协议等特性去决定的

## 事务隔离级别（定义了一个事务可能受其他并发事务影响的程度）
![image](https://github.com/Knight-Wu/articles/assets/20329409/691021cc-8803-47e6-94cd-2b9cff58b961)
### 如何实现四种隔离级别
![image](https://github.com/user-attachments/assets/e825a4b7-dae1-4b19-87c3-3c05da9d4d38)

## 事务解决的问题
### 脏读(dirty read)
A事务读到了B事务尚未提交的数据, 可理解为读到了脏数据, 若此时B事务回滚, 则会产生数据不一致的情况.

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

### 不可重复读(unrepeatable read)
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

### 幻象读(Phantom reads)
是不可重复读的特例, 是前后两次范围读的结果不一样. 

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

## 快照读和当前读
![image](https://github.com/user-attachments/assets/94f39f1d-2572-47d3-b1ca-698ff88df009)

### MVCC 在可重复读的隔离级别下, 使用快照读(一般select), 如何解决一般情况下的幻读的
可重复读隔离级是由 MVCC（多版本并发控制）实现的，
### MVCC 在可重复读的隔离级别下, 如何出现幻读的
* 总结

  MVCC 不能完全解决幻读问题, 如果你在事务当中只用快照读(普通select) 那不会发生幻读, 但是如果中间插入当前读(select for update, insert , update, del) 那么就会读最新数据, 从而发生第二次读到第一次不存在的数据发生幻读, 如果是这种情况, 那么第一次读就直接select for update, 就加入了间隙锁, 从而就不会发生幻读.
  
* 出自https://xiaolincoding.com/mysql/transaction/phantom.html#%E7%AC%AC%E4%B8%80%E4%B8%AA%E5%8F%91%E7%94%9F%E5%B9%BB%E8%AF%BB%E7%8E%B0%E8%B1%A1%E7%9A%84%E5%9C%BA%E6%99%AF

* 场景一, 普通select
![image](https://github.com/Knight-Wu/articles/assets/20329409/ab77d616-8a76-4484-9728-f3aaf73c3fa9)

* 场景二, 先select , 再select for update
![image](https://github.com/Knight-Wu/articles/assets/20329409/f4135850-9095-4b5d-8f66-367a1bb2c7da)

### 如何彻底解决幻像读

#### 如果在事务中只进行快照读, 例如简单select
通过MVCC 即可, 实现的方式是开始事务后（执行 begin 语句后），在执行第一个查询语句后，会创建一个 Read View，后续的查询语句利用这个 Read View，通过这个 Read View 就可以在 undo log 版本链找到事务开始时的数据，所以事务过程中每次查询的数据都是一样的，即使中途有其他事务插入了新纪录，是查询不出来这条数据的，所以就很好了避免幻读问题

#### 如果在事务中既有快照读(简单select), 又有当前读(select for update, upd, insert, del )
事务一开始直接执行select ... for update ，通过 next-key lock（记录锁+间隙锁）方式解决了幻读，因为当执行 select ... for update 语句的时候，会加上 next-key lock，如果有其他事务在 next-key lock 锁范围内插入了一条记录，那么这个插入语句就会被阻塞，无法成功插入;
否则如果一开始只是普通select , 后续又进行当前读, 然后又普通select 就会出现幻读. 


* 防止幻读所引入的范围锁(gap lock) 引起的死锁的一道题目


 ```
MySQL，InnoDB引擎，RR隔离级别下，有如下表 t1 :
id | name | score
-------|---------------|------------
15 | kobe | 24
18 | curry | 77
20 | rose | 5
30 | irvin | 91
37 | james | 22
49 | jordan | 83
50 | durant | 89


如下场景会发生什么？

【事务A执行】
time1： update t1 set score=100 where id = 25;
time3： insert into t1(id, name, score) value (25, 'jimmy', 90);

【事务B执行】
time2： update t1 set score=100 where id = 26;
time4： insert into t1(id, name, score) value (26, 'tommy', 90);

</br>
答案:</br>
关键是 time1 和 time2 都执行了对不存在的 id（25 和 26）进行 update，这两个操作都会触发 InnoDB 的 Gap Lock。

由于 t1 的主键是 id，主键索引是有序的，id 介于 20（rose）和 37（james）之间不存在值，因此：

id=25 和 id=26 实际都落在 (20, 37) 这个区间内的同一个 gap

InnoDB RR 下，update ... where id=25 会对这个 gap 加锁

![image](https://github.com/user-attachments/assets/3a6fde74-7130-4935-aaa5-c6572c18cc9f)

</br> 参考:
https://dev.mysql.com/doc/refman/8.4/en/innodb-locking.html#:~:text=A%20gap%20lock%20is%20a,of%2015%20into%20column%20t.
![image](https://github.com/Knight-Wu/articles/assets/20329409/0fd07bd6-415f-40fb-aa1c-2de31cac1196)

```
### 丢失更新(lost update)
指在一个事务读取一个数据时，另外一个事务也访问了该数据，那么在第一个事务中修改了这个数据后，第二个事务也修改了这个数据。这样第一个事务内的修改结果就被丢失，因此称为丢失修改。

![enter image description here](https://drive.google.com/uc?id=1NdpnXgkU7Q3TW0G73P0WPR_ejiUgf-Qp)

## MVCC 多版本并发控制
### 可重复读隔离级别下，A事务提交的数据，在B事务能看见吗
可重复读隔离级是由 MVCC(多版本并发控制)实现的，实现的方式是开始事务后(执行 begin语句后)，在执行第一个查询语句后，会创建一个Read View，后续的查询语句利用这个 ReadView，通过这个 Read View 就可以在 undo log版本链找到事务开始时的数据，所以事务过程中每次查询的数据都是一样的，即使中途有其他事务插入了新纪录，是查询不出来这条数据的。
### 如何实现
三个主体的交互, 一是当前事务所属id, 二是当前事务所操作的数据的事务id, 三是readView 
![image](https://github.com/Knight-Wu/articles/assets/20329409/8b4248ac-e1a3-48d4-be0d-a46d89120f26)

### 读提交和可重复读, 用 readview 实现上的区别

![image](https://github.com/user-attachments/assets/1b639486-520a-4903-b07c-64dd0e71c0d4)

### 可重复读是如何工作的, 一个例子
https://www.xiaolincoding.com/mysql/transaction/mvcc.html#%E5%8F%AF%E9%87%8D%E5%A4%8D%E8%AF%BB%E6%98%AF%E5%A6%82%E4%BD%95%E5%B7%A5%E4%BD%9C%E7%9A%84

### 读提交是如何工作的, 一个例子
https://www.xiaolincoding.com/mysql/transaction/mvcc.html#%E8%AF%BB%E6%8F%90%E4%BA%A4%E6%98%AF%E5%A6%82%E4%BD%95%E5%B7%A5%E4%BD%9C%E7%9A%84

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
