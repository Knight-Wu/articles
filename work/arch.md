metasvr: 
 meta-kafka --> metasvr es: 消费元数据kafka, chunk 和 index 的一些元信息写入es
 query-worker --> metasvr: 接受查询, 转换为查询es 
 metasvr --> es: 找出符合查询条件 chunkId , 通过时间等字段过滤
 metasvr --> indexsearch: 通过 chunkId 找到匹配的 indexId, 通过indexId 拉取 oss 数据, 目前使用bloom 过滤器, 将查询语句分词后查询是否命中bloom, 若命中返回chunkId
 metasvr --> queryworker: 返回命中的chunkId

indexsearch:
meta-kafka -> indexsearch: 因为日志搜索时间越近的搜索概率越大, 所以消费元数据的kafka 写入磁盘缓存, 保持缓存最新, 提高缓存命中率. 

client(queryworker) --> client-server: 发起请求, indexId 作为key, 使用一致性hash, 保证每次都命中同一台机器, 一致性hash 返回worker server ip and port

worker client --> worker server: 通过ip:port, grpc 查询, 先查询缓存若不命中, 则查询oss, 每个 indexId 单独给一个search thread 执行. 因为每个indexId 的chunkid 列表不可能重复. 每个chunkid 由一个download thread 进行下载, 多个chunkid 并行, 最后返回每个词对应的 chunkid list.

一致性hash 有心跳检测 rpc, 返回失败, 则过滤掉失败的节点


querymaster:

browser -> querymaster: open query. 
querymaster -> browser: return queryid.
browser -> querymaster: 周期性查询query status, 如果没有异常, 就fetch 数据, 类似分页. 交互方式是流式返回的, 如果是聚合类的查询, 查询结果会一直更新, 只返回最多 200MB 的数据, 查询出来的数据会缓存在内存中, 如果内存撑不住会输出到外部存储. 

* 如何把逻辑计划转换为物理执行计划
例如starttime 和 endtime, 是如何转为es 的range 查询的, 等于说根据key 拿到逻辑执行计划的开始时间和结束时间, 然后再构造es 查询函数 range, 最后转成json 查询es.

了解逻辑执行计划

antlr:
语法和文法. 语法指的是go 等编程语言的语法, 而文法指的是能精确描述语法的语言, 能判断此段语法是否有错. 

indexer:
log agent --> kafka: 从业务机器本机采集日志
kafka --> indexer(processor): 消费日志写chunk 和index , chunk 使用流式编码压缩写入, 不存原文省内存, chunk 格式为列式; index 目前使用bloom 算法, 支持无锁并发流式写入, 分词后过滤, 经过改良, 比google 官方的bloom 反序列化时间少百分之二十, 
indexer --> meta-kafka: chunk 和 index 的一些元信息写入元数据的kafka, 可以理解为一级索引, 不用查询原始数据就只用时间戳等一些字段信息就过滤chunk
indexer --> oss: chunk 和 chunk的索引信息(以下称为index) 存入对象存储

### 优化点
1. chunkwriter 使用对象锁, 可以改成每个列一个锁. 充分并发
2. chunkwriter 里面各种编码对象可以采用reset, 而不是new 一个, 减少gc
3. 

### 亮点
1. bloom 改良
  a. 内部采用字节存储优化了序列化和反序列化性能, 比google 官方的bloom 反序列化时间少百分之三十, 就是因为google 内部使用的是long 数组需要把8个字节的依次还原long, 但是我们是字节数组, 读整块大的字节数组就会快, 但是中间涉及到复杂的位操作, 需要先算bitIndex, 再算byteIndex, offsetInByte, 再转成 哪一个int, 以及int 里面哪一个byte, byte 里面哪一个bit, 存到 AtomicIntegerArray 里面, 最后反序列化的时候直接反序列化成一个byte 数组, 不需要按照依次 8 字节的转成 long 数组, 反序列化就快, 读取就快. 

  b. bloom 采用流式写入, 预先按照key 多少分配内存, 内部统计不同的key 的个数接近了预先key 个数才刷新, 提高bloom 存储率, 实际就是某一bit 位之前是0, 置1 的话就是一个新的key. 
  c. bloom 大小是原文日志的百分之0.1

2. 写入的时候是尽量减少内存数组的resize 和拷贝, 只拷贝了两次, 写入从kafka msg 拷贝到内存中, flush 到uss 的读取的时候拷贝了一次. 使用的是list bytebuffer

3. fst 构建chunkid 到分词的倒排索引, 根据分词找chunkid ,实际相当于bloom, key, val 的但是存储大小是bloom(fpp=1%) 的五倍, 是原文日志的百分之0.5 . 

分为前缀和不用前缀两种方式: 
前缀: 
例如 32个词使用一个前缀, term index 树形结构最后得到的是前缀, val 是词典块的地址, 然后找到词典块, 再二分查找得到具体词的postings, chunkid 列表

不用前缀: term index 树形结构最后得到的是整个完整的词, val 是词的posting 对应的文件偏移, 就没有词典里面二分查找的过程了. 但是term index 大小会比用前缀大很多. 

term index 在内存中可以理解为 term -> dictionary address 的sortedmap, 但是由于是树形结构 key 的前缀是共用的, 所以内存要少得多. 但是查找的时候效率就不是 O(1) 了, 是 O(term 的长度). 
term dictionary 在内存中理解为 term -> chunkid, 在磁盘中term 会前缀压缩. 

下图中就是前缀的表示, 图出自: https://zhuanlan.zhihu.com/p/33671444
![pasted-image](images/arch/20221115161459.png_ not show)


4. 为什么按照chunkmeta 为key 来写入

就是为了在es 中能进行快速过滤, 虽然合并了host, source 等字段, 但是依然可以通过es 全文搜索搜索到, 只是效率稍微低了点, 例如搜host1, 因为chunk 里面还含有其他host 的信息, 但是在多个chunk 并发搜索的情况下, 只返回前多少条, 影响很小, 只要把chunkmeta 的个数降低到合理能接受的范围内, 就够了. 

5. 如何避免小chunk 问题, chunk 个数过多的问题. 
再一个chunk 中, 合并host, source(文件路径), 等重复度高的字段, 

6. 为什么需要滑动窗口满了, 就强制刷新

因为滑动窗口实际是一个数组, 按bufferId 作为下标直接插入, 不限制大小数组会无限大, 爆内存.

7. chunk 格式的优劣和对比
主要参考的apache arrow, 一些编码方式也是从里面的代码单独扣出来的. 

8. 自定义索引列. 
需求背景: 用户搜索请求耗时大于 x 的日志, 因为99% 的日志都有这个关键字, 需要扫描chunk , 一行行的过滤, 扫描的数据量太大. 查询超时, 如果能在chunk 级别, 数值类型的加个最大最小的索引, 能不用解析chunk, 只需要扫描es , 过滤效果就很好. 
所以数值类型加了最大最小值的索引存在es, string 类型存hashset , 然后拼接存es, 使用全文扫描. 解决orderid 的一些场景. 

9. 实时查询. 
需求背景: 现在没有刷新到uss 的chunk 是搜不到的, 存在indexer 内存里面, 如果刷新频率过快, chunk 过多, 查询扫描效率也会下降. 所以需要可以搜索在内存中没有被刷新的数据. 从queryworker -> processorserver -> indexer, processorsvr 和
indexer 采用grpc 双向stream 连接, 因为processorsrv 是请求发起方, 但是没办法广播请求到所有的pod, k8s 没有原生支持, 需要记录ip port, 有变更要同步, 很麻烦, 所以采用indexer 连processorsvr, 然后processorsvr 发起请求, indexer 回包. 
中间克服了nginx 代理, 连接超时频繁断开, 数据包大小受nginx 限制的问题, 都通过测试和观察, nginx 加配置, 双向ping 解决了. 

10. objid 分布式唯一的uuid, 其实只是bucket里面唯一即可, bucket1 和bucket2 objid 有可能有重复, 实际上是bucket/objid 这个路径唯一即可, 一个实例在消费某个topic 的三个partition, 1,3,5, 那么 5 当做snowflake 算法的workerid , 因为由kafka 保证, 不会有一个parititon 同时被两个进程消费. 



# 系统设计
## feed 流系统
1. 推模式
表设计如图, 适合用户的粉丝数不多的情况, 例如微信朋友圈, 朋友数一般就是几千个最多了, 发一条微博时, 插入一条微博id, 用户id 的数据进入数据库, 相当于是往用户的收件箱发一条信息, 
并且用户查询自己的feed 流式, 获取到微博id, 仍然需要校验这条微博是否删除, 是否还在关注列表等;
实现简单.但是发微博时由于写入量很大, 可能发的慢. 不适用粉丝数较多的场景. 
2. 拉模式
![pasted-image](images/arch/20221121000608.png)