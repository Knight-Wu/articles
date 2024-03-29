# 怎么保证日志不漏采集, 保证不了不重复

## INotify + 轮训 保证能监听到文件的新增和改动

轮训是一个兜底的方式, INotify 可能某些场景有遗漏, 
* inode 在不同时刻重用的问题咋解决的呀, 这一时刻用着inode 是1, 文件被删了, 然后新创建的inode 又是1, 这种情况咋解决的

可以对于每个文件的前128B生成一个hash值，作为这个文件多一个维度的标识

* 文件被采集之前被删除的情况呢

INotify 能感知文件的新增, 文件新增时就立马open, 保证对这个文件的引用, 只要进程对文件的引用不为零就不会被OS 回收, 真正删除

## 保留每个文件采集到的offset , 类似checkpoint
key 是deviceName + INode Id, 因为INode Id 只在单个文件系统内唯一, val 是offset , 采集到的文件字节数; 因为这个更新是延迟更新, 定期保存在本地或者发送到远程存储, 肯定有延迟, 
在读取文件内容未发送之前, 突然程序中断, 就会导致数据重复

## 采集器数据链路上的内部组件都保证优雅关闭, 可以减少重复


## 日志库的滚动, 导致文件的滚动
日志库的滚动行为, 滚动有两种方式, 一种是mv+create, 一种是copy+truncate, 后者对大多数采集器都不友好, 可能导致丢数据

## mv+create 的滚动步骤

1. 重命名程序当前正在输出日志的文件。因为重命名只会修改目录文件的内容，而进程操作文件靠的是inode编号，所以并不影响程序继续输出日志。
2. 创建新的日志文件，文件名和原来日志文件一样。虽然新的日志文件和原来日志文件的名字一样，但是inode编号不一样，所以程序输出的日志还是往原日志文件输出。
3. 通过某些方式通知程序，重新打开日志文件。程序重新打开日志文件，靠的是文件路径而不是inode编号，所以打开的是新的日志文件。

# 采集进程的部署
* 用iac执行部署脚本。不过公司的基础设施不可靠，我们自己增加了一套远程自升级机制作为补充
* 直接二进制放在文件服务器上，管理服务下发地址给采集器，采集器自己去下载。

# 参考
1. https://cloud.tencent.com/developer/article/1517898