### 存储 key val 最简单的模型, hash index and append only file
存储的是 key 和 val, 
hash index: key, value(文件中字节的偏移量)
append file: 一开始一直往一个文件里面 append, append 到一定大小, 进行文件 compaction, 只bao'liu


> Written with [StackEdit](https://stackedit.io/).
<!--stackedit_data:
eyJoaXN0b3J5IjpbLTc5MTg5MzYxN119
-->