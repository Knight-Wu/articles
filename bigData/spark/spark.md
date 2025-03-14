# OOM 问题
## dataframe.sample().mapPartition() 出现 OOM 
```
sample data error, Traceback (most recent call last):
  File "/tmp/data_correctness_check_job.py", line 150, in check_sample_data
    if df1_minus_df2.count() > 0:
  File "/opt/amazon/spark/python/lib/pyspark.zip/pyspark/rdd.py", line 1521, in count
    return self.mapPartitions(lambda i: [sum(1 for _ in i)]).sum()
  File "/opt/amazon/spark/python/lib/pyspark.zip/pyspark/rdd.py", line 1508, in sum
    return self.mapPartitions(lambda x: [sum(x)]).fold(  # type: ignore[return-value]
  File "/opt/amazon/spark/python/lib/pyspark.zip/pyspark/rdd.py", line 1336, in fold
    vals = self.mapPartitions(func).collect()
  File "/opt/amazon/spark/python/lib/pyspark.zip/pyspark/rdd.py", line 1197, in collect
    sock_info = self.ctx._jvm.PythonRDD.collectAndServe(self._jrdd.rdd())
  File "/opt/amazon/spark/python/lib/py4j-0.10.9.5-src.zip/py4j/java_gateway.py", line 1321, in __call__
    return_value = get_return_value(
  File "/opt/amazon/spark/python/lib/pyspark.zip/pyspark/sql/utils.py", line 190, in deco
    return f(*a, **kw)
  File "/opt/amazon/spark/python/lib/py4j-0.10.9.5-src.zip/py4j/protocol.py", line 326, in get_return_value
    raise Py4JJavaError(
py4j.protocol.Py4JJavaError: An error occurred while calling z:org.apache.spark.api.python.PythonRDD.collectAndServe.
: org.apache.spark.SparkException: Job aborted due to stage failure: Task 144 in stage 158.0 failed 4 times, most recent failure: Lost task 144.6 in stage 158.0 (TID 80085) ( executor 1536): ExecutorLostFailure (executor 1536 exited caused by one of the running tasks) Reason: Remote RPC client disassociated. Likely due to containers exceeding thresholds, or network issues. Check driver logs for WARN messages.
```
### sample 的原理
![image](https://github.com/Knight-Wu/articles/assets/20329409/4dad4952-4d47-42b7-8b4d-07fc6e76f970)
withReplacement=true, use this sample class:
![image](https://github.com/Knight-Wu/articles/assets/20329409/b35232fb-1655-49c3-b872-31e4c6667068)

如果用的是这个代码, 抽样的比例是 fraction, sample(withReplacement=True, fraction), 用的就是这个抽样类: BernoulliCellSampler,  抽样的原理就是每个 partition id 作为一个 random seed, 保证每个 partition 的 random seed 不一样, 然后读取 partition 的每条记录, 
每条记录都会产生一个随机数从 0 到 1 之间, 如果随机数小于 fraction, 就会选择这条记录, 所以按理来说不会OOM, 但是还是有 OOM , 目前没有一个很好的解释

#### 解决办法
* 减小 partition size
```
pyspark
spark_conf.set("spark.hadoop.fs.s3a.block.size", "10M") # default is 30M
spark_conf.set("spark.sql.files.maxPartitionBytes", "10048576") # default is 134217728 (128 MB)
```

* 把数据装到内存和磁盘

dataframe.persist(MemoryAndDisk), force using disk 如果需要用的话, 因为我们机器数量 20 台, 每天内存 32 GB, 磁盘每天 100 GB 以上, 肯定够放下一天的数据, 这样比减小 partition size 要快, 在确定能放下数据的情况下, 暂时先用这个方法, 如果放不下了再选择减小 partition size. 

# 数据倾斜(data skew)
## 常见情况
某个parttion的大小远大于其他parttion，stage执行的时间取决于task（parttion）中最慢的那个，导致某个stage执行过慢, 或者 OOM
情形暂定为两种
   1. 相同的key的数据量太大(一般都是因为这个)
   2. 不同的key在同一个parttion的数据量太大

## 常见解决思路
* 提升并行程度, 适用情况2.
* 在hive端就去除倾斜的现象, 保证spark端使用的时效
* 过滤少数导致数据倾斜的key
* 将key添加随机前缀, 进行局部聚合,然后去掉随机前缀, 再进行全局聚合
* 自定义parttion
* 避免shuffle, 通过将小表广播到大表所在的节点, 进行hash-join
* 通过抽样来

## 实际场景两表 join, 一个大表的重复 key 很多

大厂大数据平台数据分析的题目
```
大表A(大概每天的增量数据有3T)
uuid url time 
url为用户每天访问的url, 有些热门网址出现的次数就非常多了, 就有数据倾斜的问题

小表B(映射表数据为大概300GB)
url result

需要将A和B join, 生成结果保存在hive
uuid time url result
```
> 思路

1. 因为是热门网址的url, 故可以将表A抽样百分之十,  取这百分之十的最热门的top100个url, 生成表C
2. 将表C作为小表广播, 与表B join, 获取top100 的url的result
3. 再将表C 广播, 与表A join, 则数据倾斜比较严重的url都已经有了result
4. 再经过一次filter, 把已经有了result的url排除掉, 剩下的就时其余比较平均的url, 生成表D
5. 将表D与表B join, 则可以获取所有的result.

## 实际场景: jdbc 读取 mysql, 分区列是时间, 有些时间段数据量很大

* 数据倾斜为何产生
因为读取 adb , 通过 jdbc 协议, jdbc 链接的时候需要指定 numOfPartition, 用作拆分成子 sql, 因为我们分区字段是时间, 就是将一年的时间拆分成 numOfPartition 这么多段, 每段查出来的数据就是 spark 的一个 partition, 如果一段里面的数据例如是流量高峰那么这个 Partition 的数据量一大, 就会造成 OOM.
### 如何解决
#### 估算单个 partition 多大来避免 OOM, 没有根本解决问题
一般是设置 partiiton 的数量, 总数据量除以partition 的数量得到partition 需要处理的数据大小, 如果这个数据超过了某个executor 的内存大小就会OOM, 实际上可以做一个数学计算估算峰值一秒的数据大小, 
然后某个partiiton 假设读取x 秒的数据, 可以得到partition 数据的大小, 所以当parition 非常大的时候, 极端情况下某个parition 只负责一秒的数据处理, 说白了就是把数据分片划分得足够多, 这样会造成partition 数量非常大, 造成任务切换开销很大, 性能比较低, 但是稳定性会好一点. 
#### 通过给 ADB 加上一个自增列
例如读 1w 条, 就给他们标上 id , 1到 1w , 然后以这列作为分区列, 那么每个 partition 所获取到的行数就是固定的, 
可以通过 mysql row_num 函数做到:

```
SELECT *, ROW_NUMBER() OVER ( ORDER BY bet_end_time) FROM pg_analytic_db.player_bet where bet_end_time >= '2022-07-26 11:08:40.165' and bet_end_time <= '2022-07-26 12:08:40.165';
```
</br> 但是这个首先需要扫描全表, 加上一个字段, 对表有变更, 如果读出来的时候再加, 也需要指定分区列, 在读出来的时候就 OOM 了, 已经测试过. 

##### 通过salting 随机
这个不行, 因为要先读出来才能在 key 加随机后缀, 进行打散, 但是在从 mysql 读出来的时候就 OOM 了
![image](https://github.com/Knight-Wu/private/assets/20329409/c42a79d9-5a66-4722-a480-e6e8d9331e60)

##### 最佳方案: 通过 streaming read jdbc, 让 spark 自动把多余数据写到磁盘
打开streaming read 之前, 是通过计算partition 的数量然后把总时间范围拆分成多个小的时间范围，等于说你要查一天的数据，用一百个分区，就把 一天的时间分成一百份，然后通过一百个子查询去查询mysql，每返回一个子查询的所有数据spark 才会判断内存数据是否需要写到磁盘，如果这个子查询的数据量太大就会造成OOM，等于说spark 还没来及的判断是否要写入磁盘就发生了OOM, 于是看了 JDBCRDD 的源码, 发现有通过游标的方式打开 streaming read, 等于之前是以子查询的方式查询mysql, 一次会拉去一个partition, 而是每次查询固定条数的数据, 就判断是否写到磁盘上. 就很均匀, 然后parition 的数量不需要很多,  因为能写到磁盘上, 减少了任务切换的开销, 性能也有显著提升. 
* 关键代码
   * spark JDBCRDD 传参, 构造 Query
     ![image](https://github.com/Knight-Wu/private/assets/20329409/22e71e84-c7c7-4f7a-9cc9-60c4ef9de1df)
   * JDBC 判断是否是 streaming read
     [jdbc doc https://dev.mysql.com/doc/connector-j/en/connector-j-reference-implementation-notes.html](https://dev.mysql.com/doc/connector-j/en/connector-j-reference-implementation-notes.html)
     ![image](https://github.com/Knight-Wu/private/assets/20329409/299ee750-cf96-4c87-990a-e7629982f39e)
   * 所以只需要把 fetchSize 设置成 Integer.MIN_VALUE 即可
   * 默认 streaming 拉取的时候一次一条,  可以这样设置streaming 一次拉取100 条
     ```
     conn = DriverManager.getConnection("jdbc:mysql://localhost/?useCursorFetch=true", "user", "s3cr3t");
     stmt = conn.createStatement();
     stmt.setFetchSize(100);
     rs = stmt.executeQuery("SELECT * FROM your_table_here");
     ```

# 代码生成技术
简单来说是这几点. 

1. 减少虚函数的调用, 虚函数是 java 的匿名实现类, 虚函数的地址不确定, 会导致 cpu 预测指令的失效, 导致性能下降, 如果生成代码都在一个类, 就不存在虚函数的调用, 
2. 减少递归的调用, 每次递归需要切换上下文, 影响性能
3. 减少类型的判断, 动态生成的代码由于可以在运行时知道数据的类型, 就避免了多余的分支判断来处理类型. 
## 参考
https://blog.csdn.net/qq_41775852/article/details/105471662

# 随机抽样
Apache Spark 是一个大规模数据处理框架，它提供了多种抽样方法以从大量数据中获取样本。在 Spark 中，数据通常以分布式数据集 (Resilient Distributed Dataset, RDD) 或数据框 (DataFrame) 的形式存在。以下是 Spark 中的一些抽样方法及其原理：

1. 简单随机抽样（Simple Random Sampling）：简单随机抽样是从数据集中随机选取样本的过程，每个元素被选中的概率是相等的。在 Spark 中，可以使用 RDD 的 `sample()` 或 DataFrame 的 `sample()` 函数实现简单随机抽样。这两个函数都接受一个布尔参数（决定是否有放回抽样）、一个抽样比例和一个随机数生成器种子。

2. 分层抽样（Stratified Sampling）：分层抽样首先将数据集划分为若干个互不重叠的子集（或称为“层”），然后从每个子集中独立进行简单随机抽样。这种方法可以保证每个层中的样本比例在最终样本中得到体现。在 Spark 中，可以使用 RDD 的 `sampleByKey()` 或 `sampleByKeyExact()` 函数实现分层抽样。这两个函数都需要一个包含键值对的 RDD，其中键表示层的标识，值表示层内的元素。

3. 随机权重抽样（Weighted Random Sampling）：在随机权重抽样中，每个元素被选中的概率与其分配的权重成正比。这种抽样方法适用于不同元素之间存在不同重要性的情况。Spark 并未直接提供实现随机权重抽样的函数，但可以通过编写自定义函数来实现。

抽样方法的选择取决于数据集的特点以及分析任务的需求。在 Spark 中，抽样方法的实现原理基本上都是基于伯努利试验或泊松抽样。这些方法通过生成随机数来决定是否选择数据集中的某个元素。在分布式环境中，各节点独立进行抽样操作，最后将抽样结果汇总起来。

## 简单随机抽样
Spark中的简单随机抽样（Simple Random Sampling，SRS）基于随机数生成器来选择DataFrame中的行。在Spark中实现简单随机抽样的原理是基于Bernoulli采样和Poisson采样。下面是详细的解释：

1. Bernoulli采样：对于每一行，生成一个[0, 1]之间的随机数。如果该随机数小于或等于预定的抽样比例（`fraction`），则选择该行；否则不选择。这种方法适用于无放回的抽样，即每行最多被抽取一次。

2. Poisson采样：对于每一行，生成一个服从Poisson分布的随机数，其中λ（Poisson分布的参数）等于预定的抽样比例乘以数据集的大小。然后，使用这个随机数作为该行被抽取的次数。这种方法适用于有放回的抽样，即每行可以被抽取多次。

在实际应用中，Spark通过以下步骤实现简单随机抽样：

1. 初始化随机数生成器：使用指定的随机数种子（`seed`）初始化一个伪随机数生成器（PRNG）。使用相同的种子可以确保每次运行时获得相同的随机抽样结果。

2. 遍历数据集：对于数据集中的每一行，使用随机数生成器生成一个[0, 1]之间的随机数。对于Bernoulli采样，根据随机数和抽样比例决定是否选择该行。对于Poisson采样，使用随机数作为该行被抽取的次数。

3. 创建新的DataFrame：将选中的行添加到新的DataFrame中，以便后续操作。

4. 返回结果：返回包含随机抽取行的新DataFrame。

需要注意的是，简单随机抽样可能会导致不均衡的数据分布，尤其是当数据具有多个类别或群组时。在这种情况下，可以考虑使用分层抽样或聚类抽样等更复杂的抽样方法。

在Spark中，你可以使用DataFrame的`sample`方法来进行简单随机抽样。这个方法会返回一个新的DataFrame，其中包含原始DataFrame中的一部分随机抽取的行。以下是如何使用`sample`方法的一个示例：

```python
from pyspark.sql import SparkSession

# 初始化SparkSession
spark = SparkSession.builder \
    .appName("Simple Random Sampling") \
    .getOrCreate()

# 加载数据
data = spark.read.csv("your_data_file.csv", header=True, inferSchema=True)

# 设置抽样比例和随机数种子
fraction = 0.1  # 这意味着大约10%的行将被随机抽取
seed = 42  # 设置随机数种子以保证结果可重复

# 进行简单随机抽样
sampled_data = data.sample(fraction, seed)

# 显示抽样结果
sampled_data.show()
```

在这个示例中，`fraction`参数表示你想从原始数据中抽取的行的比例。在这个例子中，我们抽取大约10%的行。`seed`参数是随机数生成器的种子，通过设置相同的种子，你可以确保每次运行时获得相同的随机抽样结果。

使用`sample`方法后，`sampled_data` DataFrame 将包含原始数据中随机抽取的行。你可以使用`show`方法查看抽样结果。

请注意，这个示例假设你已经安装并配置了PySpark环境。如需详细信息，请参阅官方文档：https://spark.apache.org/docs/latest/sql-programming-guide.html


## spark 和 mr 区别，快在哪里
* spark 有dag 构建了复杂任务的依赖关系，一些相同的依赖就不需要重复计算，可以缓存下来，但是mr 
模型就很难做这个事。
* 复杂任务时 MR 可能会分为多个 map reduce 过程，中间需要多次写hdfs 保存中间结果，网络和多次的io 都很慢，spark 则是懒计算的，构建DAG, 优化任务执行顺序，减少 Shuffle 次数, 而且计算结果尽可能的保存在内存. 
* mr 适用于那些不要求速度长时间运行的离线任务，基于磁盘的设计更稳定，适合处理超大规模离线数据，长期运行不易崩溃, 硬件成本也比较低
*  spark 提供 Transformation 和 Action 两大类算子（如 map, filter, join） , MR 仅支持 Map 和 Reduce 操作，扩展性有限 
#### spark执行的大致流程
参考自:https://github.com/JerryLead/SparkInternals
![enter image description here](https://drive.google.com/uc?id=1bHUSmMyvvt0AExkVkPJx9wplKzlnCWxr)

![image](https://user-images.githubusercontent.com/20329409/42255995-3835ea58-7f81-11e8-9003-78b446c332cf.png)



1. 初始化driver  
首先会初始化配置, 在用户进程下新建一个serverSocket, 默认端口0, 监听消息 application 运行的状态变化等消息 , 然后构建命令行参数, 用java 命令启动driver 进程, 进入main class, 初始化SparkContext , 初始化 driver 端通信, job 执行等一些对象, 确立driver 端的地位
2. 建立application 逻辑执行图 
 生成rdd的执行图, rdd.compute() 定义数据来了之后怎么计算, rdd.getDependencies() 定义rdd的依赖
3. 建立application 物理执行图
 只要触发了一次action 就生成一个job, 在DAGdagScheduler.runJob() 进行stage 划分, 在submitStage() 生成 shuffleMapTask 或ResultTask, 然后将task 打包成taskSet 给交给 taskScheduler 去执行, 当taskSet 可以运行就将task 的时候就交给 sparkDeploySchedulerBackend 去执行.

 4. 分配task 
sparkDeploySchedulerBackend  接受到taskSet 之后, 通过 driverActor 将序列化之后的task 发送到worker 节点executor 的CoarseGrainedExecutorBackend Actor 上
 
 5. executor 执行task
executor 将task 包装成 TaskRunner, 并放到线程池里面运行, 运行task 时先序列化task, 包括数据和rdd , 然后按序调用rdd 的compute 函数, 最后返回结果. 

下图是任务提交流程:
![enter image description here](https://drive.google.com/uc?id=1iKFppKl4pyWyEEeBoFsOvxGa0tCow64w)

* task 执行完成之后的结果如何返回给driver

executor 执行完task 之后的结果, 需要返回到driver, 如果这个结果过大, 超过spark.akka.frameSize = 10M, 就先把结果存放到executor blockManager 管理, 存储结构由storageLevel 决定, 存储大小为spark.storage.memory 决定, 并把位置信息返回, 之后由driver http 去取; 如果结果小于 10M, 则直接返回. 

task 执行的结果主要分为shuffleMapTask 和resultTask, shuffleMapTask 生成的是MapStatus, 一是该task 所在的blockManager 的blockManagerId(由executorId+host, port, nettyPort 组成), 二是task输出的FileSegment 大小, resultTask 输出的结果则是partition 最后输出的结果. 

* driver 接收到task 返回的结果后, 做什么处理
结果如果是indirectResult, 则需要调用 blockManager.getRemoteBytes 去取, 取到的结果如果是resultTask, 则用resultHandler 去算, 如果shuffleMapTask的结果则需要将输出的结果存放到 mapOutputTrackerMaster 的mapStatus 结构, 以便shuffle read 去查询. 如果driver 收到的task 是stage 中最后一个task, 则告诉dagScheduler 则可以开始下一个stage, 如果是最后一个stage, 则可以开始下一个job. 

* shuffle read 如何知道去哪里获取数据
shuffle write 输出的数据信息已经保存在driver 的mapOutputTrackerMaster, HashMap<stageId, Array[MapStatus]>, 根据stageId 就可以获取到shuffleMapTask 输出的位置信息



reducer 如何取map 输出的数据: 
 启动默认5个 fetch线程, fetch 来的数据需要在内存中做缓冲, 总的缓冲的内存空间不能超过spark.reducer.maxMbInFlight＝48MB, 


#### spark 逻辑执行图的生成
1. 根据最初的数据源生成最初的RDD, 
2. 对RDD 进行一系列的transformation 操作, 每次transformation 会产生一个或多个新的RDD 
3. 对final RDD 进行action , 对每个partition 计算后产生结果返回到driver 端


#### spark 物理执行图生成
如何根据上图, 知道具体如何划分stage 和task, task 中的数据从哪些来, 
![enter image description here](https://drive.google.com/uc?id=1CWkUowX518EV6-JBQIrlavQ5FDRLh4bh)

整个computing chain 由后向前建立, 遇到shuffle 就新建一个stage, 每个stage 中, rdd 调用parentRdd.iterator 将数据一条条拿过来. 

* task的数量
在同一个stage 中的, 由上游的partition 数量决定task 的数量, 若上游的partition 来自数据源, 由数据源的split 数量决定; 然后这个task 依次调用一个stage 的多个rdd.compute 函数, 不需要保留中间结果, 除非有cache. 例如上图中的粗线条均有一个task 完成.下游的task 的数量 spark 一般提供了参数决定.

#### spark 广播
https://github.com/JerryLead/SparkInternals/blob/master/markdown/7-Broadcast.md

* 大致流程
driver 会先建一个文件夹存放需要广播的数据, 并启动一个可以访问该文件夹的httpServer , 并同时将数据写入driver 的blockmanager. 当executor 反序列化task, 开始使用广播对象之后, 会调用广播对象的readObject 方法, 先从executor blockmanager 里面去查这个数据, 不在的话, 就会通过 HttpBroadcast或 TorrentBroadcast 两种方式之一去获取数据, 并存放在executor 的blockmanager.

* HttpBroadcast
HttpBroadcast 最大的问题就是 **driver 所在的节点可能会出现网络拥堵**，

* TorrentBroadcast
基本思想是将数据分块, 当有一些executor fetch 到了一些data blocks, 那么这台executor 就可以被当做data server了. 
driver 先把data 序列化成 byteArray, 然后切割成BLOCK_SIZE（由 `spark.broadcast.blockSize = 4MB` 设置）大小的 data block, 每个block 由TorrentBlock 对象持有, 切割完dataArray 会将其回收, 将分块信息存放到driver blockManager, 同时会通知**blockManagerMaster** , 可以被driver 和executor 访问到.



#### task、partition关系
1. stage里面的task = 当前RDD其依赖或上一次的RDD partition，若是从file生成的RDD依赖指定的partition数量
2. task由scheduler 指定partition所在的node去执行, 等于说哪个节点保存这个partition, 由这个节点去计算task.
  
 The number of tasks in a stage is the same as the number of partitions in the last RDD in the stage. The number of partitions in an RDD is the same as the number of partitions in the RDD on which it depends, with a couple exceptions: the coalesce transformation allows creating an RDD with fewer partitions than its parent RDD, the union transformation creates an RDD with the sum of its parents’ number of partitions, and cartesian(笛卡尔) creates an RDD with their product.


 RDDs produced by textFile or hadoopFile have their partitions determined by the underlying MapReduce InputFormat that's used. Typically there will be a partition for each HDFS block being read. Partitions for RDDs produced by parallelize come from the parameter given by the user, or spark.default.parallelism if none is given.

 To determine the number of partitions in an RDD
    rdd.partitions().size().



#### lineage(血统)
 each RDD has a set of partitions, which
are atomic pieces of the dataset; a set of dependencies on
parent RDDs, which capture its lineage; a function for
computing the RDD based on its parents; and metadata
about its partitioning scheme and data placement.

* With [spark.logLineage](https://jaceklaskowski.gitbooks.io/mastering-apache-spark/spark-rdd-lineage.html#spark_logLineage) property enabled, `toDebugString` is included when executing an action.


 * narrow dependency
lost partition can be recomputed in parallel on other nodes
  * wide dependency 
   node failure in the cluster may result in the loss of some slice of data from each parent RDD, requiring a full recomputation
* dependencies between RDDs
  * narrow dependencies, where
each partition of the child RDD depends on a constant
number of partitions of the parent (not proportional(比例) to its
size)
  * wide dependencies, where each partition of the
child can depend on data from all partitions of the parent.
For example, map leads to a narrow dependency, while
join leads to to wide dependencies (unless the parents are
hash-partitioned)


* stage
参考[github-DAGScheduler.scala](https://github.com/apache/spark/blob/master/core/src/main/scala/org/apache/spark/scheduler/DAGScheduler.scala)
> There are two types of stages: [[ResultStage]], for the final stage that executes an action, and [[ShuffleMapStage]], which writes map output files for a shuffle. Stages are often shared across multiple jobs, if these jobs reuse the same RDDs.


> Each stage contains as many pipelined transformations
with narrow dependencies as possible. The
boundaries of the stages are the shuffle operations required
for wide dependencies wide dependency or any cached partitions
that can short-circuit the computation of a parent RDD.

### fault-tolerant
1. driver node fail
若driver故障, 则所有executor的计算结果都会丢失, 可以指定yarn-cluster, --supervise 在其他节点重启driver.
```
./bin/spark-submit \
  --class org.apache.spark.examples.SparkPi \
  --master spark://207.184.161.138:7077 \
  --deploy-mode cluster \
  --supervise \
  /path/to/examples.jar
```
2. executor node fail
会提交到其他executor重试.
3. some tasks  fail
spark 采用event触发机制, DAGSchedulerEventProcessLoop去监听队列里面的event, 
TaskSetFailed event会中止stage和job, 如果同一个stage失败次数没超过默认的四次,根据stageAttemptId作唯一标示, (否则直接终止job) ResubmitFailedStages会再次提交失败的stage, 如果parent stage存在的话, 就可以再次计算task, 但是只会计算失败的task.
> some important config

**spark.task.maxFailures**, 默认4, Number of failures of any particular task before giving up on the job, lost partition can be recomputed in parallel on othe job. The total number of failures spread across different tasks will not cause the job to fail; a particular task has to fail this number of attempts. Should be greater than or equal to 1. Number of allowed retries = this value - 1.(同一个task最多失败的次数, 若失败超过这个次数则放弃)



>设置replication, 参考 [RDD Persistence](https://spark.apache.org/docs/latest/rdd-programming-guide.html) , 使用这个配置: MEMORY_ONLY_2, MEMORY_AND_DISK_2, etc.


* lineage与DAG的区别
> lineage 描述的是RDD的依赖关系, 依赖链, 是一个逻辑执行计划 , 如图1; 而DAG 是有向无环图, 节点是rdd, 边是rdd的转化关系, 并能区分stage,是一个物理执行计划. 如图2




```
Above figure depicts an RDD graph, which is the result of the following series of transformations:


val r00 = sc.parallelize(0 to 9)


val r01 = sc.parallelize(0 to 90 by 10)


val r10 = r00 cartesian df01


val r11 = r00.map(n => (n, n))


val r12 = r00 zip df01


val r13 = r01.keyBy(_ / 20)


val r20 = Seq(r11, r12, r13).foldLeft(r10)(_ union _)


```
![图1](https://user-images.githubusercontent.com/20329409/42252935-d3c1f7ce-7f71-11e8-9171-8cabac1c9cdb.png)
![图2](https://user-images.githubusercontent.com/20329409/42252939-d76bbe6e-7f71-11e8-9ffe-22a5bc516e72.png)






#### RDD (Resilient Distributed Dataset)
1. 在内存中计算, 内存中放不下遵循LRU(最近最少使用算法)将其余置换到disk
2. 懒计算, 直到执行action 操作, 才会去计算RDD
3. 容错性
4. 不可变性(Immutability)
> 一旦创建了rdd, 就是不能修改的, 除非生成新的 rdd, 避免了并发计算的问题, 而且每次 rdd transformation是确定的


> rdd.toDebugString() 可以打印出rdd 的依赖链


* rdd, dataframe, dataset的区别
参考 [infoQ](http://www.infoq.com/cn/articles/three-apache-spark-apis-rdds-dataframes-and-datasets)
> rdd属于最底层的抽象, 提供最复杂的操作; dataFrame, dataSet提供某些场景下定制化的api, 使代码更加明确和易用, 也提供了性能上的优化, 适用于半结构化的数据, 例如json; 而dataSet<T> 可以作为有类型的抽象, 将数据映射到java对象上,  对语法错误和解析错误提供编译时检查(说白了可以在编译时检查数据类型),  dataFrame可以当做是dataSet<Row> 是无类型的

![](https://drive.google.com/uc?id=19Vh-HgtUFwVhFLRHRtYqiVIw7Rvy-eoL)

### spark job scheduling
参见
1. [https://spark.apache.org/docs/latest/job-scheduling.html](https://spark.apache.org/docs/latest/job-scheduling.html)
> Dynamic Resource Allocation

当有等待的task在队列中, 会周期性的以2的倍数增加executor; 当executor因为空闲而超过一定时间, 会退出.

> spark job pool

*fix me*

#### spark persist
参考自 [https://github.com/JerryLead/SparkInternals/blob/master/markdown/6-CacheAndCheckpoint.md](https://github.com/JerryLead/SparkInternals/blob/master/markdown/6-CacheAndCheckpoint.md)
* checkPoint
一些运算量很大, 运算时间很长, 或者依赖很多RDD的RDD 则需要进行checkpoint 分为reliable 和local 两种.

 1. reliable

 SparkContext.setCheckpointDir(directory: String) to set the checkpoint directory, 目录必须是hdfs路径, 因为 checkPoint file实际上是保存在executor 机器上的.

 用法: RDD.checkpoint() , 当前rdd被保存, 对 parent rdd的引用都会被移除; 每个job执行完之后, 会往前回溯所有的RDD, 若需要checkpoint, 则标记为 CheckpointingInProgress, 最后启动一个新的job 完成Checkpoint. 
job完成 checkpoint之后, 会将rdd的所有 dependency释放掉, 设置该rdd的状态为 checkpoint, 并为该rdd 设置一个强依赖, 设置该rdd的 parent rdd为 CheckpointRDD, 该 CheckpointRDD 负责读取文件系统的 Checkpoint文件, 生成对应rdd 的 partition.

 checkpoint 也是lazy的, 触发了action之后, 才会往前回溯到需要checkpoint的RDD 进行checkpoint.

 checkpoint的时候, 会重新起一个新的job 去计算需要checkpoint 的rdd, 所以一般在checkpoint 之前先执行 cache, 则后续的checkpoint 的过程就直接把内存中的数据持久化到硬盘中了, 省去了重复计算. 


* cache

将中间数据存储在内存中, 以便其他job使用. 当task计算某个rdd的partition, 如果该partition需要cache, 则计算后存到内存.


* cache和checkpoint的区别 

cache之后, 整个lineage 还会保留, 但是cache的数据不能在多个application 之间共享; 但是 checkpoint 之后,会把 lineage全部删除, 因为是持久化到 hdfs的, 可以供其他job使用; 


* 与mr的checkpoint的区别

 Hadoop MapReduce 在执行 job 的时候，不停地做持久化，每个 task 运行结束做一次，每个 job 运行结束做一次（写到 HDFS）。在 task 运行过程中也不停地在内存和磁盘间 swap 来 swap 去。 可是 Hadoop 中的 task 中途出错需要完全重新运行，比如 shuffle 了一半的数据存放到了磁盘，下次重新运行时仍然要重新 shuffle。Spark 好的一点在于尽量不去持久化，所以使用 pipeline，cache 等机制。用户如果感觉 job 可能会出错可以手动去 checkpoint 一些 critical 的 RDD，job 如果出错，下次运行时直接从 checkpoint 中读取数据。但不足的是，checkpoint 需要两次运行 job 


#### spark shuffle
参考自 
1. [https://github.com/JerryLead/SparkInternals/blob/master/markdown/4-shuffleDetails.md](https://github.com/JerryLead/SparkInternals/blob/master/markdown/4-shuffleDetails.md)
2. [jcchoiling](http://www.cnblogs.com/jcchoiling/p/6440102.html)
* 简而言之, 是再次分布数据的过程.例如 reduceByKey(), 需要在所有的分区找到key的所有value ,并把所有的value聚合到一起计算. 


* 具体流程, 如下图所示
![image](https://user-images.githubusercontent.com/20329409/42148956-caa5b412-7e06-11e8-9e30-9e107ff9e1ea.png)
![https://drive.google.com/uc?id=1dU1KNWSPaHWyzVufVTgWDtjKbW0ixEhQ](https://drive.google.com/uc?id=1dU1KNWSPaHWyzVufVTgWDtjKbW0ixEhQ)

shuffle 一开始是Hash-Based Shuffle, 1.1及之后的版本默认的sort-manager 变成了Sorted-Based Shuffle, 先介绍一下Hash-Based Shuffle,

> shuffle会产生两个stage, 分别对应 shuffle write和shuffle read shuffle write 

>  shuffle write

可以当做mapper阶段, 这一步的task叫做shuffleMapTask , 假设mapper阶段的partition有m个, task中的每条记录, 通过 partitioner.partition(record.getKey())) , 会被分散到 bucket上(可以当做是磁盘上的文件, 一是减少内存消耗, 二是可以有极强的容错性), 每个task 对应的bucket的数量 ==reducer的数量 == 下一个stage的task的数量, 

> shuffle read

假设该阶段的partition的数量是r个, 
可以当做reducer阶段,会去driver 的MapOutputTrackerMaster询问shuffleMapTask 的数据输出的位置.
 
 * 什么时候reducer 进行fetch
当shuffle map 所有task都写完之后, 为了符合stage 的概念
* 边fetch 边处理还是, fetch 结束后再处理
边fetch 边处理, 
* fetch 来的数据存放在哪里
存放在reducer 的内存缓冲区, 当内存缓冲区达到一定的阈值, 再split 到磁盘, `spark.shuffle.spill = false`就只用内存

> Hash-Based Shuffle 的不足
1. 会产生大量的小文件, 数量是m*r个, mapper端和reducer端的内存消耗变大, 进而导致GC 压力很大, 可以通过 consolidateFiles的配置优化, 将文件数下降为 num(cores) * r.
如下图: 
![enter image description here](https://drive.google.com/uc?id=1dU1KNWSPaHWyzVufVTgWDtjKbW0ixEhQ)
具体参见 [https://issues.apache.org/jira/browse/SPARK-751](https://issues.apache.org/jira/browse/SPARK-751)
2. 文件句柄占用很多


> 然后介绍 Sorted-Based Shuffle

 参考资料, 按优先级
1. [https://issues.apache.org/jira/browse/SPARK-2045](https://issues.apache.org/jira/browse/SPARK-2045)
2. 源码: SortShuffleManager.scala 
3. [http://www.cnblogs.com/jcchoiling/p/6440102.html](http://www.cnblogs.com/jcchoiling/p/6440102.html)
4. **Tungsten-Sorted Shuffle**的源码: UnsafeShuffleWriter.scala

相比于Hash-Based Shuffle 的主要改进是减少了文件的数量，这样可以减少每个文件对应的一个内存buffer，减少内存，以及IO。
文件的数量是map task num * 2，主要原理是map 阶段有N 个partition，现在内存中分配n 块内存，内存容不下了，生成一个临时文件，文件头部是一个list：partitionId，startByteOffset。每次内存容不下了就生成一个这样的临时文件，临时文件数量可控制，例如有10 - 100 个临时文件时，合并成一个大的文件，头部仍然是一个list：partitionId，startByteOffset。最后map 结束，生成两个文件一个data 文件， 一个index 文件：list：partitionId，startByteOffset。
	
进一步分为sort和Tungsten-Sorted 两种shuffle.  Tungsten详情参见: 
1. [https://issues.apache.org/jira/browse/SPARK-7081](https://issues.apache.org/jira/browse/SPARK-7081)
2. [https://0x0fff.com/spark-architecture-shuffle/](https://0x0fff.com/spark-architecture-shuffle/) 


> 提升shuffle 的性能

[https://www.cloudera.com/documentation/enterprise/5-9-x/topics/admin_spark_tuning.html](https://www.cloudera.com/documentation/enterprise/5-9-x/topics/admin_spark_tuning.html)
 
* groupByKey vs reduceByKey

reduceByKey 会先进行combine，数据会少，groupByKey 不会combine

> spark和MR的shuffle的区别

MR的combine和reduce 的records必须先sort. 而且显式的分为 map(), spill, merge, shuffle, sort, reduce几个阶段. 而spark 只有不同的 stage 和一系列的 transformation(), 说白了是编程模式的不同.



#### 如何设置并行度
1. spark.defalut.parallelism , 是全局默认的并行度, 如果没有指定则使用这个并行度.
    spark.sql.shuffle.partitions

2. 源数据在hdfs上面, 
 ```
 sparkContext.textFile(String filePath,int minPartition)
//  minPartition 决定了这个文件被分成几片, 
split size  = 总文件大小/ minPartition,
actual split size = Math.max(mapred.min.split.size,Math.min(split size,file block size(默认128MB)))
再为每个split 创建一个task
```
3. If the stage is receiving input from another stage, the transformation that triggered the stage boundary accepts a  numPartitions  argument:

``` 
val rdd2 = rdd1.reduceByKey(_ + _, numPartitions = X)
```
Determining the optimal value for  X  requires experimentation. Find the number of partitions in the parent dataset, and then multiply that by 1.5 until performance stops improving.
4. 推荐 executor instances * core per executor 的两到三倍 = num of task , 避免cpu浪费(某些task 已经跑完了, 可以跑剩下的task)




#### spark 资源分配
* executor core 
    --executor-cores in shell(or in conf spark.executor.cores) 5 means that each executor can run a maximum of five tasks at the same time。
    > A rough guess is that at most five tasks per executor can achieve full write throughput, so it’s good to keep the number of cores per executor below that number

* num of executor
     
 --num-executors command-line flag or spark.executor.instances configuration property control the number of executors requested.
 
 Starting in CDH 5.4/Spark 1.3, you will be able to avoid setting this property by turning on dynamic allocation with the spark.dynamicAllocation.enabled property. Dynamic allocation enables a Spark application to request executors when there is a backlog of pending tasks and free up executors when idle.

 YARN may round the requested memory up a little. YARN’s yarn.scheduler.minimum-allocation-mb and yarn.scheduler.increment-allocation-mb properties control the minimum and increment request values respectively.


* 跑批任务中(hive on spark, hive on mr)的资源分配
> spark.executor.instance(实例数), spark. executor.memory(单个实例内存), 这两个参数可以根据实际情况进行相应调整。当处理数据量较大逻辑较为简单时可以增大实例数减小内存，提高批处理的并行数；当处理数据量不大但是逻辑较为复杂的任务时可以提高单个实例的内存，减小实例数，提升每个实例的处理能力。


参考自
1.  [https://spark.apache.org/docs/latest/tuning.html](https://spark.apache.org/docs/latest/tuning.html)
2. [distribution_of_executors_cores_and_memory_for_spark_application](https://spoddutur.github.io/spark-notes/distribution_of_executors_cores_and_memory_for_spark_application.html)


* 内存


![enter image description here](https://drive.google.com/uc?id=1Hk4apOubApExzejHwMx96J5dAi9j4KQi)

spark-1.6 之后使用UnifiedMemoryManager, 之前使用StaticMemoryManager, 由配置**spark.memory.useLegacyMode** 指定, UnifiedMemoryManager 由execution和storage, other以及system reserved组成, execution用于computation in shuffles, joins, sorts and aggregation, storage用于caching和传播数据到集群中. execution和storage共享一块内存M, 当execution不使用它所用的内存时, storage可以抢占, 反之亦然; 只有当storage的内存使用量低于R时, execution 才能 evict(驱逐) storage, 否则R的内存使用量是会一直保持给storage使用的.

> 抽象化的内存计算

参见
1. [内存有限的情况下 Spark 如何处理 T 级别的数据？](https://www.zhihu.com/question/23079001)
**实际上真实情况下大部分数据都可以加入进内存中, 或者数据的子集, 但是针对子集都无法加入到内存中的情况**, spark使用iterator的方式, 流式处理数据, 将一小批数据从源数据读入, iterator应用算子计算后再落盘, 空间复杂度是O(1), 暂不考虑shuffle 中间结果保存的情况. 



> 计算实际内存的消耗

简单的方法是把RDD放到cache里, 然后看web UI的storage的占用

> spark.memory.fraction

上文的M, 计算方式是(JVM heap space - 300MB) * (default 0.6).其余是留给: user data structures, internal metadata in Spark, and safeguarding

> spark.memory.storageFraction

上文的R, 在M中所占的比例, 默认是0.5, 给storage一直使用的,不会被execution抢占.可以看spark-history-server中storage使用的情况, 如果很小, 可以把这个值调低.

> spark.yarn.executor.memoryOverhead

指的是 off-heap memory per executor, 用来存储 VM overheads, interned strings, other native overheads, 默认值是 Max(384MB, 10% of spark.executor-memory), 所以每个executor使用的container的物理内存需要囊括spark.yarn.executor.memoryOverhead 和executor memory两部分.

> spark on yarn 内存实际使用计算

executor实际使用的内存是: total = spark.executor.memory + spark.yarn.executor.memoryOverhead, 
而yarn 实际分配的内存, 也就是executor所在container分配的内存: 
```
if(total <= yarn.scheduler.minimum-allocation-mb){
	 total =yarn.scheduler.minimum-allocation-mb(调度时一个container能够申请的最小资源，默认值为1024MB)
}else{
total = yarn.scheduler.minimum-allocation-mb+ yarn.scheduler.increment-allocation-mb的整数倍(container内存增量，每次增加申请的内存是该值的整数倍，默认值为1G)
}
```
参考资料:

1. [https://wongxingjun.github.io/](https://wongxingjun.github.io/2016/05/26/Spark%E5%86%85%E5%AD%98%E7%AE%A1%E7%90%86/)



> 内存性能优化
1. 尽量少使用类, 减少不必要的对象空间, 尽量使用基本数据类型, The [fastutil](http://fastutil.di.unimi.it/) library provides convenient collection classes for primitive types that are compatible with the Java standard library.
2. 尽量用int作为key, 而不是string
3. 把对象序列化存储, 使用MEMORY_ONLY_SER, 大大减少存储空间, 但是读的时候需要反序列化, 消耗cpu
4. 使用kryo序列化, 需要预先注册, 并设置kryo的缓存大小




> spark GC 调优

思想是减少java object所使用的比例, 尽量使用基本数据类型, 并且使用 MEMORY_ONLY_SER去存储对象, 此时对于每一个RDD的partition 只有一个object(a byte array)
1. 如果经常full GC, 则说明内存太小了.需要调大
2. 如果minor gc比较多, 则可以调大eden为原来的三分之四.
3. Try the G1GC garbage collector with `-XX:+UseG1GC`. It can improve performance in some situations where garbage collection is a bottleneck. Note that with large executor heap sizes, it may be important to increase the [G1 region size](http://www.oracle.com/technetwork/articles/java/g1gc-1984535.html) with `-XX:G1HeapRegionSize`

4. As an example, if your task is reading data from HDFS, the amount of memory used by the task can be estimated using the size of the data block read from HDFS. Note that the size of a decompressed block is often 2 or 3 times the size of the block. So if we wish to have 3 or 4 tasks’ worth of working space, and the HDFS block size is 128 MB, we can estimate size of Eden to be `4*3*128MB`.
* 衡量GC
使用参数: -verbose:gc -XX:+PrintGCDetails -XX:+PrintGCTimeStamps

>  Broadcasting Large Variables

例如一个 static lookup table 可以把它作为广播变量, 
Spark prints the serialized size of each task on the master, so you can look at that to decide whether your tasks are too large; in general tasks larger than about 20 KB are probably worth optimizing.

> data locality

最理想的情况是code和data在一个jvm ,其次是在一个node(需要一个node的不同进程间的通信), 最坏情况是不同机架的情况. spark是怎么做尽量保证data和code的本地化的呢? 先等本地的cpu一段时间, 如果超时仍然非空闲, 则会把task转移到其他executor 空闲的cpu. see the`spark.locality` parameters on the [configuration page](https://spark.apache.org/docs/latest/configuration.html#scheduling) for details


> 其他参考

[美团点评spark基础篇](https://tech.meituan.com/spark-tuning-basic.html)
1. 对多次使用的RDD持久化, rdd.checkpoint() 可以在多个application共用


#### spark join
分为 shuffle hash join、broadcast hash join以及sort merge join


> hash join 和 broadcast hash join

 将小表作为Build Table，大表作为Probe Table; 先将小表的join key, hash到bucket中, 构建hashtable, hashtable如果太大, 会放到磁盘上; 再讲大表的join key进行hash到同一个bucket中, 再判断两者的key是否相同. hash join的时间复杂度: O(a+b), 而传统的笛卡尔积是 O(a*b)
,之所以选择小表作为build table, 生成的hashTable比较小, 能够完全放到内存中. 而broadcast hash join 则是将小表广播到大表的分区节点上, 和各个分区并发的进行hash join , 必须满足 : 被广播小表必须小于spark.sql.autoBroadcastJoinThreshold，默认为10M。经总结: 两张小表适用于hash join, 一张大表和一张极小表适用于broadcast hash join, 而一张大表和一张不太小表适用于  Shuffle Hash Join


> Shuffle Hash Join

> 将两个表按照join key进行重分区 , 则两个表的相同key的数据会被分配到同一个节点上, 再在各个节点上进行hash join


3. Sort-Merge Join
可以看下 [spark-join-pull过程](https://github.com/apache/spark/pull/3173)
> 将两个表先进行shuffle,因为目前spark的shuffle后的数据是排序好的, 再根据join key 进行merge. 适用两个大表的情况




### spark monitor

> some tips
1. To review per-container launch environment, increase `yarn.nodemanager.delete.debug-delay-sec` to a large value (e.g. `36000`), and then access the application cache through `yarn.nodemanager.local-dirs` on the nodes on which containers are launched. This directory contains the launch script, JARs, and all environment variables used for launching each container. This process is useful for debugging classpath problems in particular.


#### spark log
> 分为三部分: driver log, executor log, spark history server log

* driver log
> 使用 yarn logs -applicationId xxxId > tmp.log
* executor log
> 在下图目录下, 需要去hdfs 找applicationId下的目录
如下图在hadoop /tmp/logs/${usr}/logs/${applicationId}
```
yarn.nodemanager.remote-app-log-dir 		/tmp/logs
yarn.nodemanager.remote-app-log-dir-suffix 	logs
```

* spark history server log
注意不要搜完整的application name, 有些是未完成的, 会有后缀, 用
grep:  hadoop fs -ls history-server-log-path |grep application-name
```
spark.history.fs.logDirectory 	file:/tmp/spark-events
spark.eventLog.enabled 			true
spark.eventLog.dir 				path/log
```

* 可以使用github 上面的uber 开源的 jvm-profiler 来监控每个executor 所使用的内存和cpu 的情况, 最高值等



#### spark-shell 执行sparkSql


///显示当前数据库的表，是一个transformation操作，生成RDD  
scala> sqlC.sql("select count(1) from crm_app.cust_group_detail").collect;





#### spark on yarn 
> 问题排查

 先看rm日志, 再看nm日志,再看nm的stdout, stderr, 
详细见 [https://community.mapr.com/docs/DOC-1323](https://community.mapr.com/docs/DOC-1323)

> some tips
1. 避免每次spark on yarn 都把spark的jar包上传到yarn, 可以加配置spark.yarn.jars为hdfs目录或者本地目录, 例如: hdfs://path, local:/path.
2. jar包上传
```
spark.driver.extraClassPath=./antlr-runtime-3.4.jar
spark.executor.extraClassPath=./antlr-runtime-3.4.jar  spark.yarn.dist.files=/opt/cloudera/parcels/CDH-5.12.1-1.cdh5.12.1.p0.3/jars/antlr-runtime-3.4.jar
```

其中前两个引用的相对目录，指的是YARN 的container的进程的工作目录， 第三个配置就是把jar包上传到container的工作目录, 也可以引用hdfs的目录
3. external shuffle service
> 作用

* Spark系统在运行含shuffle过程的应用时，Executor进程除了运行task，给其他Executor提供shuffle数据. 当Executor进程任务过重，导致GC而不能为其他Executor提供shuffle数据时，会影响任务运行. 这里实际上是利用External Shuffle Service 来提升性能，External shuffle Service是长期存在于NodeManager进程中的一个辅助服务。 通过该服务来抓取shuffle数据，减少了Executor的压力，在Executor GC的时候也不会影响其他 Executor的任务运行。
* 当executor因为失败而退出, 或者在资源动态收集的情况下, 因为空闲而退出, 一些计算结果可能会因为无法获取而重新计算, 特别是shuffle的时候, map端executor计算的结果可能因为executor 退出而无法获取, 所以此时需要一个常驻的service 运行在nodemanager(使用yarn的情况)

配置参见: 
1.[http://zhm8.cn/2017/08/30/spark%20shuffle%20%E8%B0%83%E4%BC%98/](http://zhm8.cn/2017/08/30/spark%20shuffle%20%E8%B0%83%E4%BC%98/)
5. [https://spark.apache.org/docs/latest/running-on-yarn.html](https://spark.apache.org/docs/latest/running-on-yarn.html)





### spark sql和presto的区别
* spark sql更方便进行数据的分析, 适合做OLAP(On-Line Analytical Processing)
* presto更适合做实时的响应, 更适合做交互式查询, 在数据量和内存差不多的时候性能较好, 因为完全基于内存做计算,适合做OLTP(on-line transaction processing)
* presto的社区人数相比spark而言比较少, 出现问题不容易解决


### hive on spark, hive on MR 和spark sql的区别
1. hive on spark 和spark sql 计算引擎都是spark, 只是从sql翻译成执行计划不一样, 并且可以使用hive的一些优化特性, 特别是对sql的优化, 这方面hive 的经验更丰富, 历史更久, 相当于对用户提供了两个项目的优化特性, 
2. hive on spark 最后启动一个spark session, 是一个常驻任务, 可以运行多次application
3.  spark sql 默认使用hive metastore去管理metadata, 使用spark 自身的sql 优化器: catalyst.


* 若要讨论spark和MR的区别
  1. spark 容易编程, 不需要过多的抽象;MR 则不够灵活, 相同的操作; spark支持多种算子, 而MR只支持map和reduce, 功能没有spark 丰富和易用.
  2. spark支持内存和硬盘以及混合存储三种方式, 而mr只支持hdfs一种, 这个是spark比较快的一个重要原因.
  3. spark的任务分配是更细粒度的, 例如划分了多个rdd, 中间有任务失败不需要从头开始计算; 而mr需要从头
  4. spark 默认是lazy compution, 触发了action 才会计算, 可以对中间过程进行很多优化

  
  * 性能对比， hive on spark , hive on mr, spark sql ,hive on tez
  [2014-benchmark](https://www.slideshare.net/hortonworks/hive-on-spark-is-blazing-fast-or-is-it-final), 
总而言之, hive on spark在large join时性能突出, spark sql在map join时性能突出, hive on mr比较慢

### hive on spark

#### spark-submit
* shell 脚本
```
#!/bin/sh
set -o nounset
# 第一个错误, shell终止执行
set -o errexit 
export SPARK_HOME=
lib_path=
tempView=mblTempView
outputDir=/user/wutong/mblOutput
mblFile=mbl.txt
hdfs dfs -rm -r -f ${outputDir}
${SPARK_HOME}"/bin/spark-submit" \
--class PushMblToMerchantShellMain \
--master yarn \
--deloy-mode client \
--queue root \
--driver-class-path ${lib_path}"*" \
--jars ${lib_path}"" \
--conf spark.default.parallelism=200 \
--conf spark.sql.shuffle.partitions=400 \
--conf spark.executor.cores=3 \
--conf spark.executor.memory=450m \
--conf spark.executor.instances=4 \
${lib_path}"cms-sparkintegration.jar" "$@" \
-file_name 
-insertSql "insert overwrite table crm_appl.mbl_filter select * ${tempView}" \
-tempView ${tempView} \
-outputSql "select * from crm_app.mbl_filter" \
-showSql "select * from ${tempView}" \
-outputDir ${outputDir} \
-inputPartitionNum 200 \
-outputPartitionNum 200 \
-inputFlag true \
-appNme PushMblToMerchantShellMain \
-local_file_path ${lib_path}${mblFile}
-remote_file_path /user/wutong/mblFile \
-countForShortMbl 3000
-encrypt_type md5


```
#### spark sql 执行的大致流程
![image](https://user-images.githubusercontent.com/20329409/42792823-b5ed02f6-89a9-11e8-995b-a68e53a4f1c8.png)


> 参考自
> 1. [sparksql-catalyst](http://hbasefly.com/2017/03/01/sparksql-catalyst/)
> 2. mit.edu-sigmod_spark_sql.pdf , 百度云网盘/book




>  首先将sql 语句通过parser模块解析成语法树, 称作 unresoloved logical plan, 这个plan再通过 Analyzer借助元数据解析为 logical plan, 再通过基于规则的优化, 得到 optimized logical plan, 此时执行计划依然是逻辑的, 并不能被spark 理解, 还需要转化为 physical plan.


1. parser

将一长串sql 解析为一个语法树, 相当于是划分成一个个token, 并指定了token执行的先后顺序, 该模块基本都使用第三方类库 antlr实现. 
2. analyzer

 遍历第一步提到的语法树, 将一个个token替换为具体的函数, 例如token为sum(), 则替换为具体的聚合函数; 并做数据类型的绑定, 定位到那个表的哪个列.
3. optimzer

分为基于规则优化和基于代价优化(目前支持还不好, 具体说来是比较多种规则优化下哪个的时间代价越小, 就采用哪些规则进行优化), 规则优化例如谓词下推（Predicate Pushdown）, 例如是filter下推到join之前, 先进行过滤再join, 减少大量数据; 常量累加（Constant Folding）, 将中间值事先计算, 得出一个中间结果; 和列值裁剪（Column Pruning）, 例如只要一列数据的, 就只传递该列数据, 减少大量的io和带宽. 

4. 将逻辑执行计划转化为物理执行计划, 转化为特定的算子进行计算.


* 查看spark sql执行计划
5. 使用queryExecution方法查看逻辑执行计划，使用explain方法查看物理执行计划, 在spark-sql 命令行, 执行
```
spark.sql("xxxsql").queryExecution()
spark.sql("xxxsql").explain()
```
6. 使用Spark WebUI进行查看

#### spark sql 编写技巧
* hive support
enableHiveSupport 之后, 使用的是hive 作为 spark.sql.catalogImplementation
* 小数操作
string 类型可以直接加减乘除, 结果自动转化为小数, 然后再用cast (col  to decimal(5,4)) 取精度, decimal(5,4) 代表总数位为5, 小数点后保留四位

#### spark shell REPL
* 检验是否开启了hive support
https://stackoverflow.com/questions/45209771/how-to-enable-or-disable-hive-support-in-spark-shell-through-spark-property-spa


> spark-2.2
```
ctrl + l // 清空屏幕快捷键
ctrl + u // 清空当前输入

spark-shell --help 
scala> sc.getConf.getAll.foreach(println) // 打印所有配置

scala> :type spark
org.apache.spark.sql.SparkSession

// Learn the current version of Spark in use
scala> spark.version
res0: String = 2.1.0-SNAPSHOT


scala> :type sc
org.apache.spark.SparkContext

```

---



#### spark 容错
* spark.task.maxFailures 
这个参数，是在spark运行task时候的task失败的最大重试限制，如果Task重试了4次失败，会导致整个job失败 spark.task.maxFailures 4 Number of individual task failures before giving up on the job. Should be greater than or equal to 1. Number of allowed retries = this value - 1. 比如下面日志就显示了4次重试 

> 86807: 17/10/17 02:00:20 ERROR Executor: Exception in task 3.0 in stage 0.0 (TID 3) 86808: java.io.FileNotFoundException: File does not exist: hdfs://Neptune/project/ecpdockets/datasets/hive/novusdoc/20171010/collection_name=N_STDCKLNK45 86904: 17/10/17 02:00:21 ERROR Executor: Exception in task 3.1 in stage 0.0 (TID 4) 86905: java.io.FileNotFoundException: File does not exist: hdfs://Neptune/project/ecpdockets/datasets/hive/novusdoc/20171010/collection_name=N_STDCKLNK45 86970: 17/10/17 02:00:21 ERROR Executor: Exception in task 3.2 in stage 0.0 (TID 9) 86971: java.io.FileNotFoundException: File does not exist: hdfs://Neptune/project/ecpdockets/datasets/hive/novusdoc/20171010/collection_name=N_STDCKLNK45 87005: 17/10/17 02:00:21 ERROR Executor: Exception in task 3.3 in stage 0.0 (TID 12) 87006: java.io.FileNotFoundException: File does not exist: hdfs://Neptune/project/ecpdockets/datasets/hive/novusdoc/20171010/collection_name=N_STDCKLNK45

* spark.yarn.max.executor.failures 
Spark executor失败的最大上限，如果达到，整个Job失败 

> 5742:xx.xx.xx.xx.xx.xx.xx.xx.xx.[Reporter] INFO org.apache.spark.deploy.yarn.ApplicationMaster - Final app status: FAILED, exitCode: 11, (reason: Max number of executor failures (3) reached) 


* spark.yarn.maxAppAttempts 和 `yarn.resourcemanager.am.max-attempts` YARN应用级别的最大尝试次

如果达到，整个YARN应用失败 前者(spark层)会override后者(yarn层），前者是应用层面，后者是YARN服务层面，因此设置如果大于后者是不起作用的。 相关日志会有这样的情况， 

>  2018-11-26 12:11:55,490 INFO org.apache.hadoop.yarn.server.resourcemanager.rmapp.attempt.RMAppAttemptImpl: appattempt_1543187098835_0006_000001 State change from FINAL_SAVING to FAILED on event = ATTEMPT_UPDATE_SAVED 2018-11-26 12:11:55,490 INFO org.apache.hadoop.yarn.server.resourcemanager.rmapp.RMAppImpl: The number of failed attempts is 1. The max attempts is 2 2018-11-26 12:11:55,491 INFO org.apache.hadoop.yarn.server.resourcemanager.scheduler.fair.FairScheduler: Application appattempt_1543187098835_0006_000001 is done. finalState=FAILED 2018-11-26 12:11:55,491 INFO org.apache.hadoop.yarn.server.resourcemanager.ApplicationMasterService: Registering app attempt : appattempt_1543187098835_0006_000002 


https://spark.apache.org/docs/latest/configuration.html https://spark.apache.org/docs/latest/running-on-yarn.html

有关Spark容错机制的文章，有博客。 https://blog.cloudera.com/blog/2017/04/blacklisting-in-apache-spark/ 


#### 疑问
* 基准测试 benchmark
* 手动尝试node fail, task fail等场景触发spark fail-tolerance
 * spark 应用程序如何估算自己的所需的内存, 是否只能通过实际跑查看webUI才能看到.




---
### 工作踩坑指南



#### spark 任务提交
* spark和scala的包都指定为provided，在客户端上指定SPARK_HOME和SCALA_HOME就可依赖到，避免版本问题
    * 如果SPARK_HOME有问题则，会出现 
    > ClassNotFound: sparkSession
    
    * 出现 xx.scala.xx error, 一般是scala 包有冲突
* 环境变量配置好后，用source生效，然后打开一个新的终端去重启应用才可以取到新的环境变量
* 需要配置分离，不用打包就可以改配置
* hive元数据库(mysql),hive存储文件的位置(hdfs)在切换集群时都需要更换
* 指定hadoop版本
> 参考 [Stack Overflow](https://stackoverflow.com/questions/30906412/noclassdeffounderror-com-apache-hadoop-fs-fsdatainputstream-when-execute-spark-s)
> 参考 [hadoop-provided](https://spark.apache.org/docs/latest/hadoop-provided.html)


* 启动脚本
> 在spark-class2.cmd 有一个 LAUNCHER_OUTPUT, 默认启动完会删除, 可以保留看下启动的脚本, 帮助排查问题


* windows环境下启动spark, 有可能脚本报错, 然后报错信息是乱码, 可以把spark的输出指定 GBK解码, 可能是因为中文环境下windows cmd的编码是中文编码, 所以用utf-8解码是乱码, 看到报错信息之后就很好排查问题了.




**待看**
* [http://www.cnblogs.com/jcchoiling/p/6440102.html](http://www.cnblogs.com/jcchoiling/p/6440102.html)
* book: sigmod_spark_sql.pdf , 百度云网盘


**经典资料**
1. [https://jaceklaskowski.gitbooks.io/mastering-apache-spark/](https://jaceklaskowski.gitbooks.io/mastering-apache-spark/)
2. [lhttps://github.com/JerryLead/SparkInternals](https://github.com/JerryLead/SparkInternals) 
<!--stackedit_data:
eyJoaXN0b3J5IjpbLTYyMDE2NjQ3OSwxNjU4MTIzMTkxLDE2MT
Y0NTE5NTksLTE5NjQ5MTU0MzIsMzcxNDE1MjUyLDE5Mzk3NTk3
MTEsNjg5MTY3NTIzXX0=
-->
