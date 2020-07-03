### 存储 key val 最简单的模型, hash index and append only file
存储的是 key 和 val, 
hash index: key, value(文件中字节的偏移量)
append file: 一开始一直往一个文件里面 append, append 到一定大小, 进行文件 compaction, 只保留最新一个 key 的 val, 形成一个 segment, 然后可以在后台线程进行多个 segments 的合并, 合并完成之后再切换读和写, 并删除合并前的文件. 

crash recovery: 可以存储 hash index 的 snapshot 在磁盘上, 避免 crash 之后重新扫描文件

* 为什么不直接在 file 进行 update key, 而是用 append 


> Written with [StackEdit](https://stackedit.io/).
<!--stackedit_data:
eyJoaXN0b3J5IjpbMTQ4MzAxNjI3MF19
-->