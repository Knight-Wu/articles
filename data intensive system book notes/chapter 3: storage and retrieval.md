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
3. 因为一个 index key 能得到一组 key, 所以可以把这一组 key 在存入ci'p

> Written with [StackEdit](https://stackedit.io/).
<!--stackedit_data:
eyJoaXN0b3J5IjpbMjExMzc3MDgyMiwxMTQzOTA4MTE0LDE1Nz
U1OTg3NDUsLTMyMjA1Njc4Ml19
-->