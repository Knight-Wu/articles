![enter image description here](https://drive.google.com/uc?id=18Q78zGYd8mHmY1xahDSvHf_Aq70cNx-R)

![](https://drive.google.com/uc?id=1FMdqU1d6Le7gTq_YyRjRvI0aMfEk8vim)

#### hbase 特点
* 底层存储基于hdfs, 随数据规模易扩展
* 分布式存储, 查询分散, 基于region可拆分,compaction等特性,良好的读写性能
* 主要基于列存储, 非常易扩展,可存储非结构化数据

#### rowKey 的设计
* tall-Narrow 模式
例如查找查找一个用户的所有邮件
可以这样设计
userId-date-messageId-attachmentId
可以用prefixFilter查询

* flat-wide mode
userId为rowKey, valueId为Column key, 很多相同的rowKey, 

* 为什么tall模式比wide模式快
因为HFile的下一级存储模式是key-value, 
结构如下:
![](https://ws1.sinaimg.cn/large/ac4c5b29gy1fm4rmm9sk8j20l4059glu.jpg)
可以更快的获取到column qualifier

####  hbase 命令 

```
// scan meta for the table, get region info
 scan 'hbase:meta',{FILTER=>"PrefixFilter('table')"}
 
//  filterlist
f_keyonly = org.apache.hadoop.hbase.filter.KeyOnlyFilter.new();
f_firstkey = org.apache.hadoop.hbase.filter.FirstKeyOnlyFilter.new();
flist = org.apache.hadoop.hbase.filter.FilterList.new([f_keyonly, f_firstkey]);
scan 'mytable', {STARTROW => 'myStart', ENDROW => 'myEnd', FILTER =>  flist }

// rowcount
hbase org.apache.hadoop.hbase.mapreduce.RowCounter <tablename>

Usage: RowCounter [options] 
    <tablename> [          
        --starttime=[start] 
        --endtime=[end] 
        [--range=[startKey],[endKey]] 
        [<column1> <column2>...]
    ]

// 重启
sh hbase-daemon.sh restart regionserver
```

### hbase 架构
RS下有多个region, 根据rowkey的分布均匀分布在多个region; 一个table的数据分布在多个region, 一个CF对应一个store, 一个memstore, 一个store下面对应多个storeFile,一个storeFile由多个hdfs的block组成


### hbase 读流程
1. 从zk上获取hbase:meta表的所在的RS,可以通过zookeeper命令(get /<hbase-rootdir>/meta-region-server)查看该节点信息
2. 从meat表获取row所在的region ,并且meta表的信息会被客户端加载到缓存 可以用 scan 'hbase:meta' 来获取该表的信息.meta表的结构
3. region信息被更新, 例如split等后, 会更新meta表



###  hbase写流程
* 大致流程
1. 本地put, autoFlush默认为true, 每次put都会执行一次RPC ,把数据送到RS, 可把autoFlush改为false, 达到一定的writeBuffer, 才会把数据批量提交
2. RS接收到请求进行校验后, 先写入WAL(保证可靠性), 再写memstore(提升效率), 把随机写变成了一次内存写和顺序写
3. memstore达到一定大小再异步flush进入hdfs

[Hbase 写入流程(网易-范欣欣)](http://hbasefly.com/2016/03/23/hbase_writer/)




* 写性能优化
  * 根据业务要求, 看吞吐量和数据准确性之间的权衡, 默认是同步写WAL, 可以改成异步写或者不写.
  * put可以改成同步批量提交, 减少到RS的RPC连接数; 也可以改成异步批量提交, autoFlush(默认为true)改为false, 但是客户端异常会导致数据丢失
  * 提高region的数量, 在Num(Region of Table) < Num(RegionServer)的场景下切分部分请求负载高的Region并迁移到其他RegionServer
  * 考虑写入的请求是否不均衡, 看rowKey的设计是否合理, 或者可以采用预分区的策略
  * 在写入很大的keyValue时,例如文件等数据, 导致RS的handler耗尽, 目前hbase-2.0已经采用hbase MOB,极大的提升了性能

#### hive数据批量写入hbase 
* 目前采用的是ftp文件到hbase客户端批量put的方式, 稳定性差, 容易引起split和compaction, 产生大量对象, GC频繁, 影响在线系统的查询, 可以采用hbase自带的bulkload, 通过将hive 的底层存储文件格式转化为hfile 导入hbase, 



#### Memstore Flush
> 参考自  [link](http://hbasefly.com/2016/07/13/hbase-compaction-1/)

> HBase会在如下几种情况下触发flush操作，需要注意的是MemStore的最小flush单元是HRegion而不是单个MemStore。可想而知，如果一个HRegion中Memstore过多，每次flush的开销必然会很大，因此我们也建议在进行表设计的时候尽量减少ColumnFamily的个数。

* Memstore级别限制
当Region中任意一个MemStore的大小达到了上限（hbase.hregion.memstore.flush.size，默认128MB），会触发Memstore刷新。

* Region级别限制
当Region中所有Memstore的大小总和达到了上限（hbase.hregion.memstore.block.multiplier * hbase.hregion.memstore.flush.size，默认 4* 128M），会触发memstore刷新。
* Region Server级别限制
当一个Region Server中所有Memstore的大小总和达到了上限（hbase.regionserver.global.memstore.upperLimit ＊ hbase_heapsize，默认 40%的JVM内存使用量），会触发部分Memstore刷新。Flush顺序是按照Memstore由大到小执行，先Flush Memstore最大的Region，再执行次大的，直至总体Memstore内存使用量低于阈值（hbase.regionserver.global.memstore.lowerLimit ＊ hbase_heapsize，默认 38%的JVM内存使用量）。当一个Region Server中HLog数量达到上限（可通过参数hbase.regionserver.maxlogs配置）时，系统会选取最早的一个 HLog对应的一个或多个Region进行flush

* HBase定期刷新Memstore
默认周期为1小时，确保Memstore不会长时间没有持久化。为避免所有的MemStore在同一时间都进行flush导致的问题，定期的flush操作有20000左右的随机延时。
* 手动执行flush
用户可以通过shell命令 flush ‘tablename’或者flush ‘region name’分别对一个表或者一个Region进行flush。
   



#### Compaction
* 作用
通过将一些hfile 合并, 减少了IO, 控制读延迟在一定的范围内, 但是compaction 的时候会出现读毛刺(因为IO 压力和带宽压力), 和写阻塞.

写阻塞: 可能写很多, 生成hfile 的速度高于compaction 的速度, 导致hfile 很多, 读延迟增大, 所以当hfile 数量达到一定则会限制写速度, 阻塞一定的时间.  

* 流程
目的选择文件进行合并, 思想是选择文件小且io负载重的文件, 有几个文件选择算法: RatioBasedCompactionPolicy、ExploringCompactionPolicy和StripeCompactionPolicy

* 触发时机
1. memstore flush, flush之后, region下面的所有store会判断各自的storeFile的数量是否大于某个值,若大于, 则会compaction.
2. 周期性检查, 先检查文件数量是否大于某个值, 若不满足, 则检查是否满足major compaction的条件, 判断是否是MajorCompaction 首先基础间隔是 hbase.hregion.majorcompaction = 七天 , 并根据majorcompaction.jitter 作浮动, 根据 storeFiles中文件最早的修改时间距离今天已经超过了间隔时间, 则进行MC 可通过配置 hbase.hregion.majorcompaction = 0 来全局关闭 MC.
3. 手动触发

* 文件选择策略
1. 排除正在compaction的文件
2. 排查单个过大的文件, 如果文件大小大于hbase.hzstore.compaction.max.size（默认Long最大值），则被排除，否则会产生大量IO消耗
> 经过排除的文件称为候选文件, 然后会再判断是否满足major compaction, 如果满足则选择全部文件进行合并, 只要满足以下三个条件中的一个, 就会执行MC
1. 用户强制执行major compaction
2. 长时间没有进行compact（CompactionChecker的判断条件2）且候选文件数小于hbase.hstore.compaction.max（默认10）
3. Store中含有Reference文件，Reference文件是split region产生的临时文件，只是简单的引用文件，一般必须在compact过程中删除

如果不满足major compaction条件，就必然为minor compaction，HBase主要有两种minor策略：RatioBasedCompactionPolicy和ExploringCompactionPolicy，


* major compaction和minor compaction有何区别?
>1.Minor操作只用来做部分文件的合并操作以及包括minVersion=0并且设置ttl的过期版本清理，不做任何删除数据、多版本数据的清理工作。

>2.Major操作是对Region下的HStore下的所有StoreFile执行合并操作，最终的结果是整理合并出一个文件。

>There are two types of compactions: minor and major. Minor compactions will usually pick up a couple of the smaller adjacent StoreFiles and rewrite them as one. Minors do not drop deletes or expired cells, only major compactions do this. Sometimes a minor compaction will pick up all the StoreFiles in the Store and in this case it actually promotes itself to being a major compaction.

After a major compaction runs there will be a single StoreFile per Store, and this will help performance usually. Caution: major compactions rewrite all of the Stores data and on a loaded system, this may not be tenable; major compactions will usually have to be done manually on large systems.

* hbase major和minor compaction源码解析(待测试和根据博文[hbase compaction ](http://hbasefly.com/2016/07/13/hbase-compaction-1/) 深入研究,了解几个策略的不同)
* 前言
本来只想围绕 触发条件, 触发周期, 内部流程, 外部影响, 如何控制几个方面来看这个代码, 忽略不重要的细节, 以了解整体流程为主要目的, 相信这也是看源码的一个指导方针吧(不然只见树木不见森林),但是从入口这里 线程池的抽象挺有意思.

* ChoreService,chore原意为工人,这里可以把这个类理解为包工头,内部有个线程池和一个管理着每个工人返回结果的map
工人默认每隔十秒钟检查一次, compactionChecker里面有个mutilper, 变成每隔 compactionchecker.interval.multiplier(默认1000) * thread.wakefrequecy(默认10 * 1000) 毫秒执行一次check

* 判断是否进行minor compaction
基于compaction policy来判断, 当前1.2.0版本的默认policy 是 RatioBasedCompactionPolicy, 根据当前的 num of storeFiles - num of compaction files > minFilesToCompact(默认为3)

* 判断是否是MajorCompaction 
首先基础间隔是 hbase.hregion.majorcompaction = 七天 , 并根据majorcompaction.jitter 作浮动, 根据 storeFiles中文件最早的修改时间距离今天已经超过了间隔时间, 则进行MC
可通过配置 hbase.hregion.majorcompaction = 0 来全局关闭 MC.

* 可在 建表的时候指定 COMPACTION => false来关闭所有的compaction
```
hbase shell>
create 'cdp_table_test', {NAME =>'crm', VERSIONS => 2 , COMPRESSION => 'SNAPPY'}, {SPLITS => ['8888|']}, {METADATA => {'SPLIT_POLICY' => 'org.apache.hadoop.hbase.regionserver.ConstantSizeRegionSplitPolicy'}}, {MAX_FILESIZE => '107374182400'}
```


#### region split 
* 产生条件
smaller(设置的最大的region size(默认10 GB), current region number的立方 * 2 * memory flush size)

* compaction和split为何会影响 IO? 亦或是其他?需要实验





#### 关闭 auto split
> 建表时设置拆分策略为 ConstantSizeRegionSplitPolicy, 并指定最大的region size 为100 GB,所有的store files size 总和超过才拆分, 若想导数快点, 则可先预分区, 待导数完毕后再关闭自动拆分.

#### 手动触发major compaction
![hbase-majoralltable.sh](https://user-images.githubusercontent.com/20329409/41819270-3831dcc6-77f0-11e8-8c2a-12b3586ace83.png)


### zookeeper 的作用
* 存储hbase元数据(hbase:meta表所在的RS的信息)
* 负责RS 和Hmaster 的协调, 某个RS down 能让Hmaster 感知到. 

### Hmaster 的作用
* 负责给RS 分配region, 负责RS 的负载均衡
* 某个RS down 之后, 负责将他的region 分配到其他RS. 
* 负责hbase schema 的更新, 例如表的增删查改等
所以Hmaster 的下线短时间内对集群读写没有影响, 对region split 也不参与, 但是会影响region 的合并. 



#### hbase important configuration
* zookeeper.session.timeout
默认3分钟, 意思是超过这个时间, HMaster才会发现然后恢复, 设置得太小,容易导致GC也会被认为 RS down

### hbase 表结构设计
1. 将业务性强, 区分度高的字段联合起来, 统一作为rowKey, 
[https://www.ibm.com/developerworks/cn/analytics/library/ba-1604-hbase-develop-practice/index.html](https://www.ibm.com/developerworks/cn/analytics/library/ba-1604-hbase-develop-practice/index.html)

#### 问题
* 阿里巴巴订单数据, 热数据存mysql, 冷数据存hbase, 然后合并, 三年后, 
存储暴增, 然后再根据订单时间定义冷热数据, 通过compaction, 通过介质, 压缩算法, 等分开冷热数据


> Written with [StackEdit](https://stackedit.io/).
<!--stackedit_data:
eyJoaXN0b3J5IjpbLTQxOTAyNzA3MCwtMTI2MzMwMTYzLC0zOT
cxNzE1MjcsMTUxNjA3MDA4NSw1NDUxOTI3LC0xMTk4NTcwMDcz
LDI5MTk4MDg2NCw5NjE1ODU4NzNdfQ==
-->