## 架构图

查询：
![diagram-3717781545449084827](https://user-images.githubusercontent.com/20329409/209085348-bebd369b-f803-4796-a99d-53c72c0339db.png)

写入：
![diagram-12618553849534760398](https://user-images.githubusercontent.com/20329409/209085441-b8c1fba0-3ced-4c8d-8eea-454178d0d3d0.png)

## metasvr
metasvr: 
 meta-kafka --> metasvr es: 消费元数据kafka, chunk 和 index 的一些元信息写入es
 query-worker --> metasvr: 接受查询, 转换为查询es 
 metasvr --> es: 找出符合查询条件 chunkId , 通过时间等字段过滤
 metasvr --> indexsearch: 通过 chunkId 找到匹配的 indexId, 通过indexId 拉取 oss 数据, 目前使用bloom 过滤器, 将查询语句分词后查询是否命中bloom, 若命中返回chunkId
 metasvr --> queryworker: 返回命中的chunkId
## indexsearch
indexsearch:
meta-kafka -> indexsearch: 因为日志搜索时间越近的搜索概率越大, 所以消费元数据的kafka 写入磁盘缓存, 保持缓存最新, 提高缓存命中率. 

client(queryworker) --> client-server: 发起请求, indexId 作为key, 使用一致性hash, 保证每次都命中同一台机器, 一致性hash 返回worker server ip and port

worker client --> worker server: 通过ip:port, grpc 查询, 先查询缓存若不命中, 则查询oss, 每个 indexId 单独给一个search thread 执行. 因为每个indexId 的chunkid 列表不可能重复. 每个chunkid 由一个download thread 进行下载, 多个chunkid 并行, 最后返回每个词对应的 chunkid list.

一致性hash 有心跳检测 rpc, 返回失败, 则过滤掉失败的节点

## querymaster
pipeline search, 管道查询类似linux 的管道, 流式返回, 传统基于ELK的查询是查询完所有日志后才返回, 浪费资源而且很慢, 用户的搜索大多是探索式的, 逐步修改查询条件, 需要快速返回; 也有部分是统计性质的, 需要聚合, 解析等功能. 每天生成4PB的数据

querymaster:

browser -> querymaster: open query. 
querymaster -> browser: return queryid.
browser -> querymaster: 周期性查询query status, 如果没有异常, 就fetch 数据, 类似分页. 交互方式是流式返回的, 如果是聚合类的查询, 查询结果会一直更新, 只返回最多 200MB 的数据, 查询出来的数据会缓存在内存中, 如果内存撑不住会输出到外部存储. 

* 如何把逻辑计划转换为物理执行计划
例如starttime 和 endtime, 是如何转为es 的range 查询的, 等于说根据key 拿到逻辑执行计划的开始时间和结束时间, 然后再构造es 查询函数 range, 最后转成json 查询es.
而逻辑执行计划说白了就是把各种算子, 排序,解析等序列化传输到物理执行引擎, 常见的就是json, 然后物理执行引擎, 再构造具体的查询代码. 

* 如何流式返回
<img width="1348" alt="image" src="https://user-images.githubusercontent.com/20329409/204509608-0b988423-ef8d-48a0-83af-51c56c95bc8b.png">
<img width="1348" alt="image" src="https://user-images.githubusercontent.com/20329409/204510782-1a940917-7c60-4d9a-82fb-f5195866b9b3.png">


* antlr
语法和文法. 语法指的是go 等编程语言的语法, 而文法指的是能精确描述语法的语言, 能判断此段语法是否有错. 

* 向量化计算, 单指令多数据流（Single Instruction Multiple Data），简称SIMD。
目前应该是没有实现的查询这边, 向量化计算实际上是需要语言的api 有支持, 原理实际上是将多个可并行化的指令批量执行, 例如for 循环累加之前步长是1, 向量化后就是一个指令做多个累加, 但是需要考虑向量化前后结果的一致. 
* logreduce
目前就是每条日志按空格等分割成token, 然后数字变成 *** , 形成一个template, 然后再根据template, 提炼出一个字典树, 每个token 为一个node, children 为下一个token 组成的tokenlist. 

## indexer

indexer:
log agent --> kafka: 从业务机器本机采集日志
kafka --> indexer(processor): 消费日志写chunk 和index , chunk 使用流式编码压缩写入, 不存原文省内存, chunk 格式为列式; index 目前使用bloom 算法, 支持无锁并发流式写入, 分词后过滤, 经过改良, 比google 官方的bloom 反序列化时间少百分之二十, 
indexer --> meta-kafka: chunk 和 index 的一些元信息写入元数据的kafka, 可以理解为一级索引, 不用查询原始数据就只用时间戳等一些字段信息就过滤chunk
indexer --> oss: chunk 和 chunk的索引信息(以下称为index) 存入对象存储

### 优化点
1. chunkwriter 使用对象锁, 可以改成每个列一个锁. 充分并发
2. chunkwriter 里面各种编码对象可以采用reset, 而不是new 一个, 减少gc
3. 把kafka 去掉, 与indexer 结合到一起, 数据缓存到本地磁盘, 并支持查询, 固定实例消费. 

### 亮点
1. bloom 改良
  a. 内部采用字节存储优化了序列化和反序列化性能, 比google 官方的bloom 反序列化时间少百分之三十, 就是因为google 内部使用的是long 数组需要把8个字节的依次还原long, 但是我们是字节数组, 读整块大的字节数组就会快, 但是中间涉及到复杂的位操作, 需要先算bitIndex, 再算byteIndex, offsetInByte, 再转成 哪一个int, 以及int 里面哪一个byte, byte 里面哪一个bit, 存到 AtomicIntegerArray 里面, 最后反序列化的时候直接反序列化成一个byte 数组, 不需要按照依次 8 字节的转成 long 数组, 反序列化就快, 读取就快. 

  b. bloom 采用流式写入, 预先按照key 多少分配内存, 内部统计不同的key 的个数接近了预先key 个数才刷新, 提高bloom 存储率, 实际就是某一bit 位之前是0, 置1 的话就是一个新的key. 
  c. bloom 大小是原文日志的百分之0.1

2. 写入的时候是尽量减少内存数组的resize 和拷贝, 只拷贝了两次, 写入从kafka msg 拷贝到内存中, flush 到uss 的读取的时候拷贝了一次. 使用的是list bytebuffer

3. fst 构建chunkid 到分词的倒排索引, 根据分词找chunkid ,实际相当于bloom, key, val 的但是存储大小是bloom(fpp=1%) 的五倍, 是原文日志的百分之0.5 . 

分为前缀和不用前缀两种方式: 
前缀: 
例如 32个词使用一个前缀, term index 树形结构最后得到的是前缀, val 是词典块的地址, 然后找到词典块, 再二分查找得到具体词的postings 就是chunkid 列表

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

## 之前架构的问题和优化点
### es
* es 写入时指定routing key, 随机生成的, 因为查询时用不到, 一批数据只写入一个节点, 避免一个慢节点拖慢所有写入, 请求直接写入数据节点, 省去写入客户端节点的不必要的转发
* shard 数过多, es 官方推荐的是单个节点, 1GB 20个shard, 如果五个节点, 每个节点32gb, 大概3000 个shard , 
* master 单线程处理task, 本质是因为es 的整个集群状态是一个数据结构体, 集群状态是你的群集中每个节点上的每个索引包含的每个分片的所有字段信息. 目前是单个master 单线程执行的, 例如 create index, update mapping, allocate(计算某个节点迁入迁出多少shard ) 等很多任务都是es master 单线程执行的, 复杂度是单个节点shard 数乘集群shard 数, 当集群shard 数达到大于5w 时, 就非常耗时, 拖慢了其他task. 解决方法是修改代码, 提前算出目前节点中的shard 数, 时间复杂度由O(m*n), 降低到O(m), m 为全平台的shard 数,大概十几万, n 为单个节点的shard 数, 将近一万. 
* 冷索引查询过慢, 冷索引指的是通过 freeze api 冻结的索引, 不占内存, 但是由于查询时需要首次加载就很慢, 新版本es 可以把fst 改为用文件系统缓存, 不冻结, 直接引入这部分代码解决了这个问题, fst 内存数据: 原始es 数据的比例 = 1.5:1000.
* 查询提前返回, 通常是按照时间戳倒序排序, 设置文档按照时间戳排序, 例如只需要一千条数据, 那么每个segment(lucene 索引) 只需要访问最大时间戳的一千条, 再归并排序即可; 并且设置 track_total_hits为false, size: 1000, terminate_after: 1000，提前返回. 
* 为什么不用日期当做rounting key, 从而查询的时候可以用来路由, 过滤掉部分shard, 因为最重要的是写入多, shard 数量主要根据写入能力来配置.如果根据日期来路由, 某个流量大的时候都会写入到某一个shard, 造成热点. 目前是一批数据用一个随机的id 当做rounting key. 
* es 重启很慢, 在hdd 磁盘上 iops 成为瓶颈, 因为每个shard 是lucene 的segment, 每个seg 有大概差不多十个文件元数据需要读取, shard 数量过多的时候就会很慢, 磁盘使用率已经满了, 但是只在hdd 机器上出现, 只是冷节点, 没有数据写入的, 理论上可以控制单台机器的节点重启并发度来解决, 面试的时候最好别提这点, 容易说不清. 

### 机器成本省在哪了
1. 存储的数据少了, 之前架构存储数据和日志数据基本1:1, es 有各种索引都必要且占空间, 目前这个比例在1:5 到1:15 之间, 根据数据类型的不同而不同, 重复率高的数据, 就是差不多 15 . 
2. 单核的写入性能也提升了, 之前的写入是分processor 和 es, 单核处理性能 2MB, 目前随着压缩比例不同, 性能从 5 到 20 MB/s 不等, 大概有五倍的提升. 

### TODO
1. es 改源码复习
2. lucene segment 新增策略
3. es 新版本优化点，解决了按照shard 数量分配了吗
4. java 新版本特性
5. 查询性能的提升，首屏时间比之前es 的快了吗，因为整个过程查询都是流式的。
6. 编码格式


### Sinker
 1.  是一个接收 Kafka 写入 Hdfs 的 exactly once 组件，解决了 Flume 在处理大数据量的时候经常会有百分之十的数据丢失和文件 EOF 等长达半年没解决的问题，能容忍 hdfs 和 kafka 集群异常，大促期间每天数据 100TB，上线的一年时间里没有出现过数据量过多或过少的问题
 2. 自己看：当时用的是先写本地再上传 hdfs ，上传成功后再 rename 的模式，但是其实直接流式上传，最后 rename 也可以，因为流式上传也是先上传到一个临时文件，最后自己调用 hdfs rename，关键靠的是hdfs 上传的代码返回成功从而判断文件是通过校验之后是完整的，最后利用 rename 才让文件可见。

### es 集群采用节点组合算法负载均衡

* es 集群里面如何均衡shard

通过平衡每个节点的shard 数量。

* es 根据shard 数来平衡各个节点的压力有什么问题

一是shard 属于各个index，index 索引属于不同业务，那么虽然一个索引里面的shard 写入可以认为是均衡的，但是不同索引的shard 写入压力不一样。二是index 滚动之后，旧索引是不写入的，新索引的shard 才承担写入压力。

* 如何分配shard 到es 的某些节点上

通过更改index 的配置即可，滚动时候新配置就生效了

* 之前的办法，根据写入权重如何分配，又有什么问题。

一个es 节点的写入权重由这个节点的shard 的写入权重总和而成，某个shard 的权重由shard 的大小除以写入时间而得，比如 多少MB每秒， 是流量单位。 那么现在可得所有es 节点的写入权重了，按照升序排列，写入权重最小的节点就是第一个要选择的节点， 例如你要新增一个index， 此时没有监控数据，就只能默认给个shard 4，等到第一次滚动之后就有监控数据了，假设就是 n 个shard， 就遍历目前所有节点n 次， 每次取写入权重最小的节点，最后把这 n 个节点写入index 配置中。那么你如何感知集群的各个节点的写入权重变化呢，创建index 的时候会知道shard 到节点的分配关系，维护一个索引维度的cache，只要索引不新增，滚动可以认为节点的写入权重没变，如果有新增索引，就再查询shard 大小和所在节点，再全局算一遍。

但是这种办法难以应对，某个业务的流量突然增加，导致某个index 的shard 的写入量都增加，那么这些节点的cpu 就会飙升。就会需要集群增加节点，但是有些节点的cpu 还是低，就利用率低，成本增加。
就算可以监控shard 写入流量来做到自动扩容，processor 扩容之后还没用，就扩shard 但是写入不均衡只是减少了而已，没有根本解决问题

* 按照 4,8,16,以及小于4 的这group 分组之后的优点
如图：
![image](https://user-images.githubusercontent.com/20329409/209163513-47164d6a-54ee-4ea2-ae6d-4113388aa31e.png)

减少不均衡程度，理论上完全均衡就是shard 数是节点数的整数倍，然后每个索引都分配同样数量的shard，但是会带来shard 数增加，master负担过重，
不规则group 用于小于4 shard 索引，可能就用几台机器写入量都比较小，这些机器均衡不均衡影响的机器数很少。
还有个问题是，目前假设节点数8 个，那么你四个节点的shard 数为4 的一个group 和另外四个节点的shard 数为4的group 承担的写入流量不一样，导致这四个节点比那四个节点负载高，但是因为除以了4 就大大减少了不均衡程度。

* 如何估算shard 数量，实时扩容呢
一开始接入没有历史数据的情况下先给个预估值，例如4，单个shard 写入能力主要看机器，磁盘和cpu，单个机器32 个shard可以达到150MB每秒，4个shard可以达到110MB每秒，所以实际生产上一个shard能有多少吞吐取决于当时能分到多少机器磁盘和cpu 的资源，如果不够，只能扩到够为止，还是不够证明要加机器，在没有bug 等异常情况下。所以给了预估值之后，如果有延迟 kafka lag，证明是消费不过来，先检查一下partition 数量，一般4个就足够百分之90的情况了，除非这个业务有大几百MB 的流量每秒，一般来说因为kafka 本身机器资源原因导致消费不过来很少，一般是下游处理慢，一个是消费的实例数量，在partition 数量合理，lag 连续几分钟递增，证明消费一直不足，然后扩消费实例 ，根据消费速率，生产速率，lag 增加的速率可以估算扩多少消费实例，如果扩完还是有lag，再扩shard，因为是2 的n次方 预设，所以直接翻倍。再不行就有异常，资源不够了等等，需要人工介入。

* 假设一个集群8台机器，一共32个节点，怎么分

先分8个四节点的，然后两两凑成16节点，再依次类推。剩下几台给小于4 shard 的索引。然后8个4节点的group怎么选呢，先按吞吐量排序，再按shard个数排序，选最低的。

* 如果加入几个节点，group 重算？

加入几个节点，会新增几个group 原有group 不变，新的group 在索引滚动时会计算group 写入压力，重新选择合适的group 来写入。

* 某个索引的shard 个数不能太少，否则单个shard 写入太占cpu也容易导致不均衡。举例单个shard 占10%cpu和20%cpu 对分配到某个节点来说压力肯定是不一样的。



* 如何判断es 集群需要扩容


* 如何判断es 集群负载不均衡

# TODO
1. java 新功能
2. es 之前存在的问题
3. 成本到底省在哪了
4. queryworker 如何在内存中做全文搜索的


# 系统设计
## feed 流系统
1. 推模式
表设计如图, 适合用户的粉丝数不多的情况, 例如微信朋友圈, 朋友数一般就是几千个最多了, 发一条微博时, 插入一条微博id, 用户id 的数据进入数据库, 相当于是往用户的收件箱发一条信息, 
并且用户查询自己的feed 流式, 获取到微博id, 仍然需要校验这条微博是否删除, 是否还在关注列表等;
实现简单.但是发微博时由于写入量很大, 可能发的慢. 不适用粉丝数较多的场景. 
2. 拉模式
![pasted-image](images/arch/20221121000608.png)
