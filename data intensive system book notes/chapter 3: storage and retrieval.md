**一开始引入的问题是如何存储 key val,**
### 最简单的模型, hash index and append only file
hash index: key, value(文件中字节的偏移量)
append file: 一开始一直往一个文件里面 append, append 到一定大小, 进行文件 compaction, 只保留最新一个 key 的 val, 形成一个 segment, 然后可以在后台线程进行多个 segments 的合并, 合并完成之后再切换读和写, 并删除合并前的文件. 

crash recovery: 可以存储 hash index 的 snapshot 在磁盘上, 避免 crash 之后重新扫描文件

* 为什么不直接在 file 进行 update key, 而是append key, 然后再后台线程进行合并和去重呢? (why append only file ? )

因为就算在 SSD 顺序读写也是更快的, append only file 很好的利用了这个特性, update key 会造成随机读写. 

* 缺陷
1. hash index 如果不能完全放在内存中, 磁盘随机 IO 会很慢
2. hash index 不能友好的支持范围查询


### SSTables( Sorted string table) and LSM-Trees( Log-Structured Merge-Tree)
SSTables 即 key 是排序的, 跟原来相比好处在哪呢, 
1. 排序之前, 两个 file segments 进行合并, 一个有 n 行, 另一个有 m 行, 合并多个重复的 key, 合并成一个 segment 的时间复杂度是 O(m*n), 排序之后是归并排序, 是 O(m+n)
2. index 不需要保留每个 key, 只需要能断定 key 所在的范围, 再查找
3. 因为一个 index key 能得到一组 key, 所以可以把这一组 key 在存入磁盘前进行压缩. 

那如何使插入的 key 是有序的呢, 使用红黑树等数据结构, 

现有的 storage engine 描述如下: 
1. 写入 key , val 首先到 memTable(红黑树等数据结构)
2. 当 memTable 膨胀到一定程度, 将有序的 key, val, 写入到磁盘 SSTable( 每个 SSTable 对应一个 sparse index(稀疏 index) ? )
3. 读请求的时候, 先查找 memTable, 再按时间顺序倒序查找 SSTable
4. 在后台周期性合并 SSTables
5. 为了防止 memTable 的数据在未写入磁盘的时候崩溃, 需要先写 write ahead log
6. 如果查找不存在的key, 在现有的数据结构下代价会很高, 需要先查找 memTable, 再依次查找SSTable, 可以使用布隆过滤器, 迅速判断某一个 key 不存在. 

思想: 一次写操作由一次顺序 IO(log append)和一次内存写就能完成, 大大提升了写性能, 写包括更新和删除, 这两个都是在内存中记一个标记, 待后续合并的时候就知道数据被更新了.

* 问题
1. 内存中的数据如何写到磁盘
2. 磁盘的文件如何 merge

### B tree
读和写都需要多次 IO 定位到具体的 key 上, 

### B tree 和 LSM tree 的区别
 LSM tree 的优点: 写入吞吐量更大, 合并导致磁盘碎片更小, 
 缺点: 当进行合并或压缩时, 会占用磁盘的资源导致读写变慢, 

### 列式存储
有时候查询只需要一行中的某几列, 若用行存储, 则需要把拥有几百列的几行全部查出来才做过滤, 但是列存储, 把每一列都按照相同行顺序放在一个文件里. 

* 列压缩

适用于某一列中重复值特别多的, 将这一列用位图编码, 每一个独特的列值对应一个 bit 数组, col in (x,y,z) 转化为三个 bit 数组按位与, 即可得到满足要求的行数, 大大减少了存储和传输的带宽.
> Written with [StackEdit](https://stackedit.io/).
<!--stackedit_data:
eyJoaXN0b3J5IjpbMTUyNzEzODE5NCwxNjgzODk5NTQzLDY5OD
YxMzEwNywyMTMzODI1OTM1LC05MzY1NDQ3NzksMzg0MzMyNjY4
LDExNDM5MDgxMTQsMTU3NTU5ODc0NSwtMzIyMDU2NzgyXX0=
-->