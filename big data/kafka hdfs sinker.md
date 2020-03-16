### 背景
data 组需要用flume 或spark streaming 消费kafka 数据写到hdfs ods 层, 但是这两个方式都会有一定比例的数据丢失, 没找到直接的异常或者因为比较难排查而放弃

### 目标
开发一个sinker 消费kafka 数据生成文件写到ods 层, 文件是 snappy 压缩的, 不超过但是要接近 hdfs block size ,为了避免小文件, 要求做到 exactly once, 消息不重复不丢失

### 调研
看了下kafka exactly once 

### 初步方案
1. 一个消费线程消费kafka partition, 然后多个IO 线程写消息到sinker 的本地, 形成多个文件 file1, file2 ..., 接近hdfs block size, 就上传hdfs
2. hdfs response 成功就checksum 对比本地和hdfs 是否一致, 
3. 若一致, 则将文件名, checksum, consume max offset 写到zk

> Written with [StackEdit](https://stackedit.io/).
<!--stackedit_data:
eyJoaXN0b3J5IjpbLTE3MzQ2MzAxMDhdfQ==
-->