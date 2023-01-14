**一开始引入的问题是如何存储 key val,**
# 最简单的模型, hash index and append only file
hash index: key, value(实际val 在文件中字节的偏移量)
append file: 一开始一直往一个文件里面 append, append 到一定大小, 进行文件 compaction, 只保留最新一个 key 的 val, 形成一个 segment, 然后可以在后台线程进行多个 segments 的合并, 合并完成之后再切换读和写, 并删除合并前的文件. 

* crash recovery

可以存储 hash index 的 snapshot 在磁盘上, 避免 crash 之后重新扫描文件, 否则崩溃恢复是不可接受的, 非常慢. 

* 如何删除某个key

在hash index 上更新val 为特殊标记

* 如何防止磁盘空间用满呢

按照一定大小组织磁盘的文件, 超过就新建一个文件, 然后定期合并, 相同key 保留更新的val, 然后删除旧的文件

* 为什么不直接在 file 进行 update key, 而是append key, 然后再后台线程进行合并和去重呢? (why append only file ? )

因为就算在 SSD 随机读写性能也不好, append only file 很好的利用了这个特性, update key 会造成随机读写. 

* 缺陷
1. hash index 如果不能完全放在内存中, 磁盘随机 IO 会很慢
2. hash index 不能友好的支持范围查询


# SSTables( Sorted string table) and LSM-Trees( Log-Structured Merge-Tree)
* LSM 核心思想

即使有许多微妙的东西，LSM树的基本思想 —— 保存一系列在后台合并的SSTables —— 简
单而有效。即使数据集比可用内存大得多，它仍能继续正常工作。由于数据按排序顺序存
储，因此可以高效地执行范围查询（扫描所有高于某些最小值和最高值的所有键），并且因
为磁盘写入是连续的，所以LSM树可以支持非常高的写入吞吐量。

* SSTables 即 key 是排序的, 跟上面的 hash index 相比好处在哪呢, 

1. 排序之前, 两个 file segments 进行合并, 一个有 n 行, 另一个有 m 行, 合并多个重复的 key, 合并成一个 segment 的时间复杂度是 O(m*n), 排序之后是归并排序, 是 O(m+n)
2. index 不需要保留每个 key, 只需要能断定 key 所在的范围, 再查找
3. 因为一个 index key 能得到一组 key, 所以可以把这一组 key 在存入磁盘前进行压缩. 
那如何使插入的 key 是有序的呢, 使用红黑树等数据结构, 

* 现有的 storage engine 描述如下

1. 写入 key , val 首先到 memTable(红黑树等数据结构)
2. 当 memTable 膨胀到一定程度, 将有序的 key, val, 写入到磁盘 SSTable( 每个 SSTable 对应一个 sparse index(稀疏 index) ? )
3. 读请求的时候, 先查找 memTable, 再按时间顺序倒序查找 SSTable
4. 查找 SSTable 的时候因为key 是分段保存的, 简单来说某个有序的文件只需要保存最大key 和最小key, 然后搜索某个key 先范围匹配到多个文件, 再在具体文件中二分查找 key
4. 在后台周期性合并 SSTables, 因为同一个时间内只写一个memotable, 然后映射成一个有序的文件, 所以多个文件的key 时间是不一样的不会出现冲突, 保留更新的key即可
5. 为了防止 memTable 的数据在未写入磁盘的时候崩溃, 需要先写 write ahead log
6. 如果查找不存在的key, 在现有的数据结构下代价会很高, 需要先查找 memTable, 因为刚写入的在这里, 按照顺序查, 最新的key 查到了就直接返回, 查不到再依次查找SSTable, 可以使用布隆过滤器, 迅速判断某一个 key 不存在. 

思想: 一次写操作由一次顺序 IO(WAL log append)和一次内存写就能完成, 大大提升了写性能, 写包括更新和删除, 这两个都是在内存中记一个标记, 待后续合并的时候就知道数据被更新了.

* 为什么一个sstable 文件只需要一个简单的索引呢, 例如最大最小key.

因为你无论如何都要把相关联的key 读到内存中, 在内存中遍历一个几千字节的文件是很快的, 远快于在磁盘中找到这个文件. 

* WAL 日志何时被丢弃

当把memotable 持久化到sstable 时就可以

* 查找不存在的key时间复杂度很高, 要扫描很多文件

可以用bloom 过滤器
# 列式存储
有时候查询只需要一行中的某几列, 若用行存储, 则需要把拥有几百列的几行全部查出来才做过滤, 但是列存储, 把每一列都按照相同行顺序放在一个文件里. (parquet 就是一种列式存储)

* 列压缩( 96 页 )
适用于列中不同值的数量远小于行数,  将这一列的每一个不同值用位图编码, 每个不同的值分别用一个 bit 数组表示, 如果这个 bit 数组中零很多, 也可以用游程编码(run-length encoded), 进一步减小存储空间.  
例如 id 列 1, 2 , 3, 9
name 列: a, b, c, d
1 这个值对应 bit 数组 [1, 0, 0, 0], 游程编码: 1 个 one, 3 个 zero
2 对应 bit 数组 [0, 1, 0, 0]
a 对应 bit 数组 [1, 0, 0, 0]
where name = "a" and id=2, 将 [0, 1, 0, 0] 与 [1, 0, 0, 0] 按位与, 结果等于 1 的行即返回, 大大减少了存储和传输的带宽.

# B tree

是在磁盘上维护有序结构的数据结构, key 也是按顺序排列的, 
如图:
