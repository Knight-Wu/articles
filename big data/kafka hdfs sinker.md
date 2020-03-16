### 背景
data 组需要用flume 或spark streaming 消费kafka 数据写到hdfs ods 层, 但是这两个方式都会有一定比例的数据丢失, 没找到直接的异常或者因为比较难排查而放弃

### 目标
开发一个sinker 消费kafka 数据生成文件写到ods 层, 文件是 snappy 压缩的, 要求zuo dao


> Written with [StackEdit](https://stackedit.io/).
<!--stackedit_data:
eyJoaXN0b3J5IjpbMjA0MTg5NjI0NV19
-->