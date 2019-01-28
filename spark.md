#### spark执行的大致流程

![enter image description here](https://drive.google.com/uc?id=1bHUSmMyvvt0AExkVkPJx9wplKzlnCWxr)

![image](https://user-images.githubusercontent.com/20329409/42255995-3835ea58-7f81-11e8-9003-78b446c332cf.png)



1. 初始化driver  
首先会在用户进程下新建一个serverSocket, 默认端口0, 用户进程监听消息, 然后构建命令行参数, 用java 命令启动driver 进程, 进入main clas, 初始化 driver 端通信, job 执行等一些对象, 确立driver 端的地位
2. 建立job 逻辑执行图 
 driver 中的transformation(), 建立血统, rdd的执行图, rdd.compute() 定义数据来了之后怎么计算, rdd.getDependencies() 定义rdd的依赖
3. 建立job 物理执行图
 只要触发了一次action 算子会就生成一个job, 在DAGdagScheduler.runJob() 进行stage 划分, 在submitStage() 生成 shuffleMapTask 或ResultTask, 然后将task 打包成taskSet 给交给 taskScheduler 去执行, 当taskSet 可以运行就将task 的时候就交给 sparkDeploySchedulerBackend 去执行.

 4. 分配task 分配
sparkDeploySchedulerBackend  接受到taskSet 之后, 通过自带的 driverActor 将序列化之后的task 发送到worker 节点executor 的CoarseGrainedExecutorBackend Actor 上
 
 5. executor 执行task
executor 将task 包装成 TaskRunner, 并从线程池抽出里抽取一个线程运来执行task. 一个 CoarseGrainedExecutorBackend 进程有且仅有一个 executor 对象。

下图是任务提交流程:
![enter image description here](https://drive.google.com/uc?id=1iKFppKl4pyWyEEeBoFsOvxGa0tCow64w)

* task 执行完成之后的结果如何返回给driver

executor 执行完task 之后的结果, 需要返回到driver, 如果这个结果过大, 超过spark.akka.frameSize = 10M, 就先把结果存放到executor blockManager 管理, 存储结构由storageLevel 决定, 存储大小为spark.storage.memory 决定, 并把位置信息返回, 之后由driver http 去取; 如果结果小于 10M, 则直接返回. 

task 执行的结果主要分为shuffleMapTask 和resultTask, shuffleMapTask 生成的是MapStatus, 一是该task 所在的blockManager 的blockManagerId(由executorId+host, port, nettyPort 组成), 二是task输出的FileSegment 大小, resultTask 输出的结果则是partition 最后输出的结果. 

* driver 接收到task 返回的结果后, 做什么处理
结果如果是indirectResult, 则需要调用 blockManager.getRemoteBytes 去取, 取到的结果如果是resultTask, 则用resultHandler 去算, 如果shuffleMapTask的结果则需要将输出的结果存放到 mapOutputTrackerMaster 的mapStatus 结构, 以便shuffle read 去查询. 如果driver 收到的task 是stage 中最后一个task, 则告诉dagScheduler 则可以开始下一个stage, 如果是最后一个stage, 则可以开始下一个job. 

* shuffle read 如何知道去哪里获取数据
shuffle write 输出的数据信息已经保存在driver mapOutputTrackerMaster, HashMap<stageId, Array[MapStatus]>, 根据stageId 就可以获取到shuffleMapTask 输出的位置信息Runner. 



 To summarize,paon 化 逻辑执行图
rie t o hedurrun se  sitag) stae shfeapas etas tas taet tasceders sparDeleducler 启动默认5个 fetch线程, fetch 来的数据需要在内存中做缓冲, 总的缓冲的内存空间不能超过spark.reducer.maxMbInFlight＝48MB, 


#### spark 逻辑执行图的生成
1. 根据最初的数据源生成最初的RDD, 
2. 对RDD 进行一系列的transformation 操作, 每次transformation 会产生一个或多个新的RDD 
3. 对final RDD 进行action , 对每个partition 计算后产生结果返回到driver 端


#### spark 物理执行图生成
如何根据上图, 知道具体如何划分stage 和task, task 中的数据从哪些来, 
![enter image description here](https://drive.google.com/uc?id=1CWkUowX518EV6-JBQIrlavQ5FDRLh4bh)

整个computing chain 由后向前建立, 遇到shuffle 就新建一个stage, 每个stage 中, rdd 调用parentRdd.iterator 将数据一条条拿过来. 

* task的数量
在同一个stage 中的, 由上游的partition 数量决定task 的数量, 若上游的partition 来自数据源, 由数据源的split 数量决定; 然后这个task 依次调用一个stage 的多个rdd.compute 函数, 不需要保留中间结果, 除非有cache. 例如上图中的粗线条均有一个task 完成.下游的task 的数量 spark 一般提供了参数取决定.

#### spark 广播
https://github.com/JerryLead/SparkInternals/blob/master/markdown/7-Broadcast.md

* 大致流程
driver 会先建一个文件夹存放需要广播的数据, 并启动一个可以访问该文件夹的httpServer , 并同时将数据写入driver 的blockmanager. 当executor 反序列化task, 开始使用广播对象之后, 会调用广播对象的readObject 方法, 先从executor blockmanager 里面去查这个数据, 不在的话, 就会通过 HttpBroadcast或 TorrentBroadcast 两种方式之一去获取数据, 并存放在executor 的blockmanager.

* HttpBroadcast
HttpBroadcast 最大的问题就是 **driver 所在的节点可能会出现网络拥堵**，因为 wsubmits a job to compute all needed
RDDs. That job will have one or more stages, which Scheduleracend. tassarepleleaced tasket  rieAtor task arseandxecut a cluster
 3. Stages are processed in orkder 上的 executor 都会去 driver 那里 fetch 数据。

* TorrentBroadcast
基本思想是将数据分块, 当有一些executor fetch 到了一些data blocks, 那么这台executor 就可以被当做data server了. 
driver 先把data 序列化成 byteArray, 然后切割成BLOCK_SIZE（由 `spark.broadcast.blockSize = 4MB` 设置）大小的 data block, 每个block 由TorrentBlock 对象持有, 切割完dataArray 会将其回收, 将分块信息存放到driver blockManager, 同时会通知**blockManagerMaster** , 可以被driver 和executor 访问到.

#### spark 具体使用一些算子, 才会体会, with individual tasks launching to compute segments
of the RDD. Once the final stage is finished in a joboace eecor task asRnne, tas pareueroe


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


> rdd.toDebugString() 可以打印出rdd 的生成链


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

cache之后, 整个lineage 还会保留, 但是cache的数据不能在多个driver program之间共享; 但是 checkpoint 之后,会把 lineage全部删除, 因为是持久化到 hdfs的, 可以供其他job使用; 


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

相比于Hash-Based Shuffle 的主要改进是减小了大量shuffle的中间文件, 减少了memory的使用, GC的压力以及减小了文件句柄, shuffle map端产生的临时文件, 当内存不够时会dump到磁盘形成临时文件, 然后再merge成最终的data 文件. 每一个shuffleMapTask只产生两个文件, 一个data文件, 一个index文件用来存储数据文件的partition信息.spark-2.X版本中已经没有hashShuffle了, 只有sort和Tungsten-Sorted 两种shuffle. Tungsten详情参见: 
1. [https://issues.apache.org/jira/browse/SPARK-7081](https://issues.apache.org/jira/browse/SPARK-7081)
2. [https://0x0fff.com/spark-architecture-shuffle/](https://0x0fff.com/spark-architecture-shuffle/) 

Sorted-Based Shuffle 

>Sort-Based Shuffle 的弱点

1.  如果 Mapper 中 Task 的数量过大，依旧会产生很多小文件，此时在 Shuffle 传数据的过程中到 Reducer 端，Reducer 会需要同时大量的记录来进行反序例化，导致大量内存消耗和GC 的巨大负担
2. 通常的shuffle 算子是不需要进行排序的, 如果按照key 进行排序, 例如sortByKey, 则需要进行map, reduce的两次排序, 具体参见: 

>    When conducting an operation that requires sorting the records within partitions, we end up sorting the same data twice: first by partition in the mapper, and then by key in the reducer.
>    [https://blog.cloudera.com/blog/2015/01/improving-sort-performance-in-apache-spark-its-a-double/](https://blog.cloudera.com/blog/2015/01/improving-sort-performance-in-apache-spark-its-a-double/)

[https://blog.csdn.net/duan_zhihua/article/details/71190682](https://blog.csdn.net/duan_zhihua/article/details/71190682)

> 提升shuffle 的性能

[https://www.cloudera.com/documentation/enterprise/5-9-x/topics/admin_spark_tuning.html](https://www.cloudera.com/documentation/enterprise/5-9-x/topics/admin_spark_tuning.html)
 
* 使用高效的算子

groupByKey when performing an associative reductive operation. For example, rdd.groupByKey().mapValues(_.sum) produces the same result as rdd.reduceByKey(_ + _). However, the former transfers the entire dataset across the network, while the latter computes local sums for each key in each partition and combines those local sums into larger sums after shuffling

* 在map 端进行数据压缩, 减少传输的数据量
* **还有呢? **






> spark和MR的shuffle的区别

MR的combine和reduce 的records必须先sort. 而且显式的分为 map(), spill, merge, shuffle, sort, reduce几个阶段. 而spark 只有不同的 stage 和一系列的 transformation(), 说白了是编程模式的不同.

> 问题
* Sort-Based Shuffle map端的聚合好像只在combineByKey 才有, 不是能减少map 端的数据输出量吗, 为何不是一旦有shuffle, 就在map端进行聚合, 换言之combineByKey 和reduceByKey 有何不同
* spark 算子选择



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
2. 如果minor gc比较多, 则可以按照调大eden为原来估计的三分之四.
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

> 问题
1. spark sql join策略在什么情况会选哪个?




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

* Spark系统在运行含shuffle过程的应用时，Executor进程除了运行task，还要负责写shuffle 数据，给其他Executor提供shuffle数据. 当Executor进程任务过重，导致GC而不能为其他Executor提供shuffle数据时，会影响任务运行. 这里实际上是利用External Shuffle Service 来提升性能，External shuffle Service是长期存在于NodeManager进程中的一个辅助服务。 通过该服务来抓取shuffle数据，减少了Executor的压力，在Executor GC的时候也不会影响其他 Executor的任务运行。
* 当executor因为失败而退出, 或者在资源动态收集的情况下, 因为空闲而退出, 一些计算结果可能会因为无法获取而重新计算, 特别是shuffle的时候, map端executor计算的结果可能因为executor 退出而无法获取, 所以此时需要一个常驻的service 运行在nodemanager(使用yarn的情况)

配置参见: 
1.[http://zhm8.cn/2017/08/30/spark%20shuffle%20%E8%B0%83%E4%BC%98/](http://zhm8.cn/2017/08/30/spark%20shuffle%20%E8%B0%83%E4%BC%98/)
5. [https://spark.apache.org/docs/latest/running-on-yarn.html](https://spark.apache.org/docs/latest/running-on-yarn.html)


### 数据倾斜
某个parttion的大小远大于其他parttion，stage执行的时间取决于task（parttion）中最慢的那个，导致某个stage执行过慢
情形暂定为两种
   1. 相同的key的数据量太大(一般都是因为这个)
   2. 不同的key在同一个parttion的数据量太大

> 解决办法
* 提升并行程度, 适用情况2.
* 在hive端就去除倾斜的现象, 保证spark端使用的时效
* 过滤少数导致数据倾斜的key
* 将key添加随机前缀, 进行局部聚合,然后去掉随机前缀, 再进行全局聚合
* 自定义parttion
* 避免shuffle, 通过将小表广播到大表所在的节点, 进行hash-join
* 通过抽样来

> 例题

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

1.  因为是热门网址的url, 故可以将表A抽样百分之十,  取这百分之十的最热门的top100个url, 生成表C
2. 将表C作为小表广播, 与表B join, 获取top100 的url的result
3. 再将表C 广播, 与表A join, 则数据倾斜比较严重的url都已经有了result
4. 再经过一次filter, 把已经有了result的url排除掉, 剩下的就时其余比较平均的url, 生成表D
5. 将表D与表B join, 则可以获取所有的result. 


### spark sql和presto的区别
* spark sql更方便进行数据的分析, 适合做OLAP(On-Line Analytical Processing)
* presto更适合做实时的响应, 更适合做交互式查询, 在数据量和内存差不多的时候性能较好, 因为完全基于内存做计算,适合做OLTP(on-line transaction processing)
* presto的社区人数相比spark而言比较少, 出现问题不容易解决


### hive on spark, hive on MR 和spark sql的区别
1. hive on spark 和spark sql 计算引擎都是spark, 只是从sql翻译成执行计划不一样, 并且可以使用hive的一些优化特性, 特别是对sql的优化, 这方面hive 的经验更丰富, 历史更久, 相当于对用户提供了两个项目的优化特性, 
2. hive on spark 最后启动一个spark session, 是一个常驻任务, 可以运行多次application
3.  spark sql 默认使用hive metastore去管理metadata, 使用spark 自身的sql 优化器: catalyst.


* 若要讨论spark和MR的区别
  1. spark 容易编程, 不需要过多的抽象;MR需要较为复杂的抽象; spark支持多种算子, 而MR只支持map和reduce, 功能没有spark 丰富和易用.
  2. spark支持内存和硬盘以及混合存储三种方式, 而mr只支持hdfs一种, 这个是spark比较快的一个重要原因.
  3. spark的任务分配是更细粒度的, 例如划分了多个rdd, 中间有任务失败不需要从头开始计算; 而mr需要从头
  4. spark默认是lazy compution, 可以对中间过程进行很多优化
  
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
--class com.mucfc.cms.spark.job.mainClass.PushMblToMerchantShellMain \
--master yarn \
--deloy-mode client \
--queue root \
--diver-class-path ${lib_path}"*" \
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
> 1. 使用queryExecution方法查看逻辑执行计划，使用explain方法查看物理执行计划, 在spark-sql 命令行, 执行
```
spark.sql("xxxsql").queryExecution()
spark.sql("xxxsql").explain()
```
> 2. 使用Spark WebUI进行查看

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
eyJoaXN0b3J5IjpbNTU2NzM4NTY1LC04Mzk0MDQ0ODAsMTgwOD
YyMDc3OSwxNTk2Njk4NjI2LC0xNjE5NzE3NzgyLDE0NTY5Mzk0
NiwtMTkxOTY0ODgyMywxNDEwMTUxODc5LC00MTA2ODc0MjYsMT
EzOTA5NzIzNCwtMTM1NDY5ODc5NCw4NDI2NTEzMTgsLTEzMzc1
MjY5NTIsMTc2NzQ1OTYzNiwtMTkxMDAyOTIyMSwtNDg0NjM1OT
M4LC0xNDk5ODk3NDI0LDEyMzA4NzU4NjIsLTIyNjM3MjAxOSwt
MTQ4MTM5NDIwMl19
-->