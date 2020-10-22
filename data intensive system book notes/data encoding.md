向后兼容(back compatibility): 指的是新的代码能兼容旧的数据. 
向前兼容(forward compatibility): 旧代码能兼容新数据, 主语都是代码或程序

### 编码
json 和 xml , csv 等原生不能表达二进制数据, 均用 Unicode, 占用空间, 且均有一些不够兼容的小问题, 例如 xml 和 csv 无法区分数字和仅有数字代表的字符串, 

### Thrift , Protocol Buffer
* 编码中如何压缩
int64 不一定占用八字节, 将每个字节的首位用于标识是否还有下一字节, 

* 如何向前兼容和向后兼容
向前兼容: 新代码读老数据, 根据 tag 某些新加的字段读取为空, 

> Written with [StackEdit](https://stackedit.io/).
<!--stackedit_data:
eyJoaXN0b3J5IjpbMTY5NTU2NjAxMywtMTE2MTcxMzAwNSwtMT
gzMzA0MzkzM119
-->