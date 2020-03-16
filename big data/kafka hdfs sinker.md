### 背景
data 组需要用flume 或spark streaming 消费kafka 数据写到hdfs ods 层, 但是这两个方式都会有一定比例的数据丢失, 没找到直接的异常或者因为比较难排查而放弃

### 目标
开发一个sinker 消费kafka 数据生成文件写到ods 层, 文件是 snappy 压缩的, 不超过但是要接近 hdfs block size ,为了避免小文件, 要求做到 exactly once, 消息不重复不丢失

### 调研
看了下kafka exactly once, 
1. 采用记录produce msg 序号的方式, 避免消息重复
2. 如何 producer , consumer end to end exactly once, 则需要事务保证一批消息的原子性, 要么全部消费到要么一条都不消费到

### 初步方案
1. 一个消费线程消费kafka partition,并在该消费线程直接写消息到sinker 的本地, 形成多个文件 file1, file2 ..., 把consume offset 写到文件名中, 接近hdfs block size, 就上传hdfs, 若此时sinker down 则读取zk 从consume max offset +1 消费
2. hdfs response 成功就checksum 对比本地和hdfs 是否一致, 若一致则3, 若不一致或resp 不成功则4; 若resp 超时, 则checksum, 一致则3, 不一致则del hdfs file, 执行4
3. 将文件名, checksum, consume max offset, filename 写到zk
4. 重新上传hdfs, 直到2 一致.
5. 因为是单线程消费, 则不会同时产生很多文件, 所以只会产生一个未完成的文件在hdfs 目录, 可以将checksum, consume max offset 记录到文件头, sinker 重启后可以从hdfs 目录最后一个成功的hdfs 文件获取 consume max offset +1

> Written with [StackEdit](https://stackedit.io/).
<!--stackedit_data:
eyJoaXN0b3J5IjpbNzIzMDc4OTUyLDEyOTg3NDg4NjUsNjcwOD
g1MjIxLDI0NTcxNjI2NiwyMTM4Nzc0NDAwLC0xNzM0NjMwMTA4
XX0=
-->