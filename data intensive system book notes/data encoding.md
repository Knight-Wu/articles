向后兼容(back compatibility): 指的是新的代码能兼容旧的数据. 
向前兼容(forward compatibility): 旧代码能兼容新数据, 主语都是代码或程序

### 编码
json 和 xml , csv 等原生不能表达二进制数据, 均用 Unicode, 占用空间, 且均有一些不够兼容的小问题, 例如 xml 和 csv 无法区分数字和仅有数字代表的字符串, 

### Thrift , Protocol Buffer
* 编码中如何压缩
int64 不一定占用八字节, 将每个字节的首位用于标识是否还有下一字节, 

* 如何向前兼容和向后兼容
向前兼容: 旧代码读新数据, 新数据中包含一个新的字段, 用一个新的 tag, 旧代码使用的是旧的 schema 不会识别出新的 tag , 即忽略新的字段
向后兼容: 新代码读取老数据, 新的 schema 的改动不能引起读取老数据的错误, 新的 schema 新加一个 tag (字段) ,旧的数据没有即读不到, 但是不能改变旧字段, 所以建议字段均是 optional(防止改成 required 校验失败), 删除字段即新代码不会读到旧代码的相关字段而已

> Written with [StackEdit](https://stackedit.io/).
<!--stackedit_data:
eyJoaXN0b3J5IjpbLTE1MDcxNzcxODYsMTY5NTU2NjAxMywtMT
E2MTcxMzAwNSwtMTgzMzA0MzkzM119
-->