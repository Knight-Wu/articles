### 数据的存储和分析
* 关系型数据库和MR的区别?
  * 因为硬件的瓶颈,数据读取和写入时间在大数据时代很慢,所以需要分布式并行处理
  * MR适合批处理, 一次写入多次读取; 关系型数据库适合交互式, 多次读和写
  * 数据本地化,由于带宽很贵, 计算节点就近处理数据.
 

## HDFS 
参考自 [hadoop官网-HdfsDesign](http://hadoop.apache.org/docs/r2.7.6/hadoop-project-dist/hadoop-hdfs/HdfsDesign.html)
#### Assumptions and Goals
1. 集群容错, 针对硬件等的失败, 能够监测并自动恢复, 保持数据可用.
2. 流式数据处理, 更适用于批量的数据处理, 而不是交互式的数据读取, 例如mysql
3. 用于存储海量数据集, 具备良好扩展性
4. 适用于一次写入多次读取, 支持append和trancate, 不支持任意点的对数据的更改.
5. "移动计算比移动数据性价比更高", 可以把计算移动到它所需要的数据旁边, 移动到同一个节点计算.

#### NameNode and DataNodes
* namenode
> 执行filesystem namespace的opening, closing, 和修改文件和目录, 并保存了block和datanode的映射

* datanode
> 负责处理客户端的读写请求, 包括对block的创建, 删除和复制等操作.

#### Replica Placement: The First Baby Steps
如果默认的三份拷贝(replication factor is three), local rack(本地机架)有两份, 其他机架有一份, 机架失效相比于node失效的几率是非常小的, 所以将两份数据都放在同一个机架, 可以充分利用同一个机架的带宽大于不同机架间的带宽, 提高读写效率.

#### The Persistence of File System Metadata
![enter image description here](https://drive.google.com/uc?id=15wVin8_CImgR1NzsEXNKxp0U9XTvnAwU)

> hdfs 在做文件系统变更的时候, 先把修改信息保存在EditLog中, 再更新内存中的fs. 并且会定期进行checkPoint, 将内存的fs持久化到磁盘上形成FSImage, FSImage的文件名例如: fsimage_${end_txid}, nn启动的时候会进行数据恢复, 加载内存fs, 先把FSImage加载, 再把end_txid后的editLog回放到内存的fs上.

* EditLog
> 结构:正在写入的EditLog: edits_inprogress_${start_txid}, 写入完成的: edits_${start_txid}-${end_txid}.



#### Robustness
> The three common types of failures are NameNode failures, DataNode failures and network partitions

* Data Disk Failure, Heartbeats and Re-Replication
> dataNode会周期性的发送心跳和blockreport 给nn, 若心跳失效则不转发IO请求给dn, 并按照之前的副本数新增额外的副本; blockreport 报告dn所拥有的 blocks

* Cluster Rebalancing
> 如果某个dn的存储空间低于某个阈值, 则会自动转移数据; 如果对某个文件的请求大量增加, 则会增加副本, 并把请求分到其他节点上, **截止到hadoop-2.6.5这个功能还没实现**

* Data Integrity
> 通过计算block的checksum, 来保证数据完整性.

>  stores these checksums in a separate hidden file in the same HDFS namespace. 

* Metadata Disk Failure
> FsImage, EditLog的损坏会导致hdfs不可用, 但是nn会保持这两个东西是多份的, 但是副本间的同步更新会降低nn的tps

* Snapshots
> 快照功能, 可以使hdfs可以rollback到历史版本

#### Data Organization
* data block
* Staging
> 客户端写文件的请求, 并不会直接把数据写到nn, 而是写到一个本地的临时目录, 直到数据量达到了一个block, client才会把数据转移到目标dn和block; 如果nn在文件关闭之前down, 则文件会丢失.
* Replication Pipelining
> 像上一节所说, 文件开始写到dn的时候, 像管道一样, 先写第一个dn, 并且同时写第二个dn, 第二个接收到数据的时候, 会同时写第三个dn.

#### Space Reclamation
* File Deletes and Undeletes
> 被删除的文件不会立马物理删除, 先保存在 /trash 目录, 直到一定的时间, nn delete it from HDFS namespace, 然后对应的block 被释放.

* Decrease Replication Factor
>  The next Heartbeat transfers this information to the DataNode. The DataNode then removes the corresponding blocks and the corresponding free space appears in the cluster. Once again, there might be a time delay between the completion of the setReplication API call and the appearance of free space in the cluster(在setReplication() 完成和实际物理空间的释放间存在延迟) 

#### HDFS High Availability 
* Architecture
![ha image](https://drive.google.com/uc?export=view&id=1RkWgG-NnTobIGzHt_NE8PRjDRwxYynzH)
>Active NameNode 和 Standby NameNode：两台 NameNode 形成互备，一台处于 Active 状态，为主 NameNode，另外一台处于 Standby 状态，为备 NameNode，只有主 NameNode 才能对外提供读写服务。

>主备切换控制器 ZKFailoverController：ZKFailoverController 作为独立的进程运行，对 NameNode 的主备切换进行总体控制。ZKFailoverController 能及时检测到 NameNode 的健康状况，在主 NameNode 故障时借助 Zookeeper 实现自动的主备选举和切换，当然 NameNode 目前也支持不依赖于 Zookeeper 的手动主备切换。

>Zookeeper 集群：为主备切换控制器提供主备选举支持。

* HA 实现细节

![https://drive.google.com/uc?view=export&id=1oDzb5mpfoNKLULIdY5JUZGF5GMGTlC0T](https://drive.google.com/uc?view=export&id=1oDzb5mpfoNKLULIdY5JUZGF5GMGTlC0T)
> NameNode 主备切换主要由 ZKFailoverController、HealthMonitor 和 ActiveStandbyElector 这 3 个组件来协同实现：

>ZKFailoverController 作为 NameNode 机器上一个独立的进程启动 (在 hdfs 启动脚本之中的进程名为 zkfc)，启动的时候会创建 HealthMonitor 和 ActiveStandbyElector 这两个主要的内部组件.

>HealthMonitor : 监测磁盘存储资源和HA状态(如active or standby or stop)


* ActiveStandbyElector(完成自动的主备选举)
1. 在zk上创建锁节点
> 如果healMonitor检测到nn状态正常, 则该nn有资格参加下一个主备选举, 两台nn 的ActiveStandbyElector 会尝试在zk上创建一个临时节点: /hadoop-ha/${dfs.nameservices}/ActiveStandbyElectorLock, Zookeeper 的写一致性会保证最终只会有一个 ActiveStandbyElector 创建成功, 创建成功的 ActiveStandbyElector 对应的 NameNode 就会成为主 NameNode，ActiveStandbyElector 会回调 ZKFailoverController 的方法进一步将对应的 NameNode 切换为 Active 状态。而创建失败的 ActiveStandbyElector 对应的 NameNode 成为备 NameNode，ActiveStandbyElector 会回调 ZKFailoverController 的方法进一步将对应的 NameNode 切换为 Standby 状态. 

2. 注册watcher监听
> 两个节点的 ActiveStandbyElector 都会向zk 注册一个watcher 来监听节点的状态变化.

3. 自动触发主备选举
> 如果active nn上面的 healthMonitor监测到nn状态发生变化, ZKFailoverController 会主动删除临时节点 /hadoop-ha/${dfs.nameservices}/ActiveStandbyElectorLock, 这样standby的zk watcher会收到nodeDel事件, 会再次进行一次主备选举, 如果是 active nn的整个节点down, 由于临时节点的特性, 也会让standBy 的zkWatcher 监测到, 从而接管

4. 防止双主现象
> 由于zk的敏感, 可能nn那台的full GC导致zk任务active nn挂掉了, 从而启动主备切换, 原先standby nn已经变成了active nn, 但是原先active那台full GC恢复后, 感知到与zk 的session已经断开, 但是在一定的时间窗口内还是active的

> 解决办法: fencing(隔离), 除了前面的临时节点外, 还创建另外一个路径为/hadoop-ha/${dfs.nameservices}/ActiveBreadCrumb 的持久节点，这个节点里面保存了这个 Active NameNode 的地址信息, 下一个active 的nn会监测到上一个active nn的信息, 在做接管时做隔离

> 隔离方法: 
> 1. 调用这个旧 Active NameNode 的 HAServiceProtocol RPC 接口的 transitionToStandby 方法，看能不能把它转换为 Standby 状态。
> 2. 如果失败, 执行配置的隔离措施, 默认是sshfence(通过 SSH 登录到目标机器上，执行命令 fuser 将对应的进程杀死)
> 3. 只有在成功地执行完成 fencing 之后，选主成功的 ActiveStandbyElector 才会回调 ZKFailoverController 的 becomeActive 方法将对应的 NameNode 转换为 Active 状态，开始对外提供服务。




#### Using the Quorum Journal Manager 作为共享存储系统
* 架构图
![image](https://drive.google.com/uc?id=1rXK3GdpZEjHeglVtrwF3R2pJ-JoopdjP)


> QJM主要保存EditLog, FSImage保存在nn的磁盘上. 两台nn, 一台active, 一台standby, 通过独立的daemon JN去完成同步,一台nn写editLog到JN上, 另一台读取editLog并组成自己的namespace,  保证namespace是同步的; 为了保证standBy能够实时获取所有的block信息, 所有的dn都会向两台nn发送block report和心跳

* Hardware resources
> 奇数n 台JN machine, 可以容忍 (n-1)/2个错误; JN daemon 是非常轻量级的.因为根据paxos算法, 只要nn能成功写入多数 JN时,这次写入就算成功, 所以三台JN, 最多能容忍一个JN节点的挂掉. 

![](https://drive.google.com/uc?id=18xoyWuP_2iOJXKVHviBm8wq-aXYSSC_S)
* 数据同步

> 只要active nn写大多数jn 成功即返回成功, 同步等待的形式, 如果没达到就会产出nn切换, 由stangby 接管.

* 数据恢复
> 当需要由standby nn接管的时候, 首先需要从EditLog 同步fs 到内存中, 但是有可能此时的各台jn 由于active nn写失败, 处于不一致的状态, 当恢复一致以后, standby nn 再把落后的EditLog补齐到内存中.

> 多台JN间如何恢复数据一致(分布式系统间如何协调多台机器数据不一致的情况): 各个JN向NN返回自己的最后一个EditLog, 然后NN选择一个最新的EditLog, 或事务数更多的作为基准EditLog, 然后各个JN都会同步到这个EditLog.

![](https://drive.google.com/uc?id=1g_qraSemEiM3DC1z0zNKo-Re636Dk5kK)
* 生成一个新的Epoch 
> Epoch 是一个单调递增的整数，用来标识每一次 Active NameNode 的生命周期，每发生一次 NameNode 的主备切换，Epoch 就会加 1。active nn写jn的时候, 每个jn 会返回最近的(lastPromisedEpoch), 然后nn 把该数加一, 再返回给jn, jn 判断比本地的 (lastPromisedEpoch)大才进行写入, 否则返回写入失败. 

> 如果原来的 Active NameNode 恢复正常之后再向 JournalNode 写 EditLog，那么因为它的 Epoch 肯定比新生成的 Epoch 小，并且大多数的 JournalNode 都接受了这个新生成的 Epoch，所以拒绝写入的 JournalNode 数目至少是大多数，这样原来的 Active NameNode 写 EditLog 就肯定会失败，失败之后这个 NameNode 进程会直接退出，这样就实现了对原来的 Active NameNode 的隔离了




* dfs.ha.fencing.methods 
> 用于隔离active和standby nn, 当需要切换nn的时候, 需要进行三个方面的隔离:

1. 共享存储fencing：确保只有一个NN可以写入edits。QJM中每一个JournalNode中均有一个epochnumber，匹配epochnumber的QJM才有权限更新JN。当NN由standby状态切换成active状态时，会重新生成一个epoch number，并更新JN中的epochnumber，以至于以前的ActiveNN中的QJM中的epoch number和JN的epochnumber不匹配，故而原ActiveNN上的QJM没法往JN中写入数据（后面会介绍源码），即形成了fencing, 即上面说的"生成一个新的epoch "

> 2和3应该用到的是在zk上面除了注册一个临时节点外, 还增加了一个持久节点保存当前active nn的信息, 当standBy接管时会向持久节点的NN发送一个请求, 请求变成standBy, 如果请求失败, 则会使用默认的 sshfence, ssh到那台机器上杀死NN进程.

2. 客户端fencing：确保只有一个NN可以响应客户端的请求。

3. DataNode fencing：确保只有一个NN可以向DN下发命令，譬如删除块，复制块，等等。

##### Automatic Failover
* Components
> two new components to an HDFS deployment: a ZooKeeper quorum, and the ZKFailoverController process (abbreviated as ZKFC)
> 失败监测: nn持有一个对zk的持久会话, nn失效, zk会知道
> nn的选举: 提供对active nn的选举.

* ZKFC的职责
> 1. 是一个zk client, 负责监控和管理nn, 和nn在一台机器.

> 2. Health monitoring, zkfc会ping nn, 了解nn的健康状况

> 3. ZooKeeper session management, zkfc 持有一个对nn的session, 
> 4. 基于zk的选举, 

#### HDFS Federation
> 多个namenode独立, 分别管理各自的namespace, 解决了namenode需要的内存过大的问题, 可以水平扩展.

> Federation架构中，NameNode相互独立，NameNode元数据、DataNode中块文件都没有进行共享，如果要进行拆分，需要使用DistCp，将数据完整的拷贝一份，存储成本较高；数据先被读出再写入三备份的过程，也导致了拷贝效率的低效；


#### HDFS Snapshots
> 用于数据备份, 数据保护, 和容灾恢复.

> 1. 创建snapshot并没有copy block, 只是记录的block list和block size.
> 2. 如果在snapshot之后有修改, 则修改按时间倒序, 获取snapshot的时候把修改减去即可.

#### HDFS and permission


#### hdfs block 的作用, 为何设置的如此之大
  * 使寻址时间远小于传输时间
  * 对大文件抽象处理
> In the Apache Hadoop the default block size is 64 MB and in the Cloudera Hadoop the default is 128 MB. If block size was set to less than 64, there would be a huge number of blocks throughout the cluster, which causes NameNode to manage an enormous amount of metadata.


### hdfs 数据一致性模型
* 当前写入的block对其他reader不可见, 除非调用
>hflush()	This API flushes all outstanding data (i.e. the current unfinished packet) from the client into the OS buffers on all DataNode replicas(只需等待dfs.replication.min的复本数据写入完成(默认为1)).
保证被其他reader可见
>hsync()	This API flushes the data to the DataNodes, like hflush(), but should also force the data to underlying physical storage via fsync (or equivalent).
调用close(),相当于hsync,并关闭了stream.

> hadoop-2.0之前有单点失败问题，namenode down恢复时间很久，现在有active nm1和standby nm2两台机器，both nodes communicate with a group of separate daemons called "JournalNodes",nm1有修改时 nm1写入log 并通过JN持久化，nm2读取日志做出相同的改变（日志的读写细节）


#### hadoop 日志
* 可以把错误打到system.error来debug
* 见中文版 P229


#### hdfs 命令
* fsck
> 查看文件系统信息, hadoop fsck path -files -blocks -locations 可以查看文件对应的块信息

> hadoop fsck -blockId blk_xxx 不需要后缀, 查看block属于的文件和副本情况.

#### hdfs 读写流程
1. 写流程, client通过 FileSystem.open()获取一个RPC 请求到nn, 然后创建一个文件,  并获取lease 保证只有一个writer, 多个reader(只有ack的packet可以读), 并且nn 检查client的权限等. 最后返回一个  FSDataOutputStream 给client写入数据,通过socket去写datanode

2. 真正开始写: client 直接与dn 通信, 通过 FSDataInputStream  发送一个写请求, 当本地的临时目录超过一个block大小后, 才会把数据发往dn, 这个过程中先把数据 packets(默认 64KB)  先写到本地的 data queue, 这个队列通过 DataStreamer去消费 , 这个dataStreamer 向nn收集新的 block的信息. 而dn组成了 pipeline,  假设副本数是3, pipeline 里面就是三个dn , 一个dn写完再把packets 交给下一个dn. 当packets写入第一个dn之后, 会转移到一个ack队列, 去保证数据的三份副本都写入成功后, 才把ack从队列中移除, 并且是等待一个block的所有ack返回之后, 才会发送下一个block.

3. 当client把最后一个block 提交到dn之后, 最后通过 DFSInputStream.close() 去关闭连接, 会将 actual generation stamp and the length of the block上报给nn, 并会轮训 nn 进行一系列的检查, 包括文件副本最小数必须大于1(默认配置), 否则抛出异常给client.
如下图![enter image description here](https://drive.google.com/uc?id=1btvOQAEX6xNWRQvmm1yOxgi-VKtbp8Al)

* 详细流程见
 [notebook-link](http://note.youdao.com/noteshare?id=1db8cf2911deed6b89523bd3feab696e&sub=A63DC7C4A24A4C759435BB12479B6BDB)
原贴地址 [http://itm-vm.shidler.hawaii.edu/HDFS/ArchDocDecomposition.html](http://itm-vm.shidler.hawaii.edu/HDFS/ArchDocDecomposition.html)


#### Understanding HDFS Recovery Processes
[https://blog.cloudera.com/blog/2015/02/understanding-hdfs-recovery-processes-part-1/](https://blog.cloudera.com/blog/2015/02/understanding-hdfs-recovery-processes-part-1/)
1. lease recovery

* 作用
1. 为了互斥写, 保证一个时刻只有一个写, 多个读
2. 保证客户端写失败的时候, lease能够释放, 并检查写的最后一个块是不是COMPLETE的, 不是的话就进行block recovery

> Before a client can write an HDFS file, it must obtain a lease, which is essentially a lock. This ensures the single-writer semantics. The lease must be renewed within a predefined period of time if the client wishes to keep writing. If a lease is not explicitly renewed or the client holding it dies, then it will expire. When this happens, HDFS will close the file and release the lease on behalf of the client so that other clients can write to the file. This process is called lease recovery. 

>The lease manager maintains a soft limit (1 minute) and hard limit (1 hour) for the expiration time (these limits are currently non-configurable), and all leases maintained by the lease manager abide by the same soft and hard limits. Before the soft limit expires, the client holding the lease of a file has exclusive write access to the file. If the soft limit expires and the client has not renewed the lease or closed the file (the lease of a file is released when the file is closed), another client can forcibly take over the lease. If the hard limit expires and the client has not renewed the lease, HDFS assumes that the client has quit and will automatically close the file on behalf of the client, thereby recovering the lease.

>The lease recovery process is triggered on the NameNode to recover leases for a given client, either by the monitor thread upon hard limit expiry, or when a client tries to take over lease from another client when the soft limit expires. It checks each file open for write by the same client, performs block recovery for the file if the last block of the file is not in COMPLETE state, and closes the file. Block recovery of a file is only triggered when recovering the lease of a file.(lease 过期之后, 检查文件的最后一个块是不是COMPLETE(必须有minimum replication number of FINALIZED replicas, 必须 minimum replication number是1, 那么当有一个副本是 FINALIZED ,这个block就是complete了), 如果不是就进行block recovery)

2. block Recovery
> If the last block of the file being written is not propagated to all DataNodes in the pipeline, then the amount of data written to different nodes may be different when lease recovery happens. Before lease recovery causes the file to be closed, it’s necessary to ensure that all replicas of the last block have the same length; this process is known as block recovery. Block recovery is only triggered during the lease recovery process, and lease recovery only triggers block recovery on the last block of a file if that block is not in COMPLETE state (defined in later section).

* block recovery和 lease recovery 的流程
>Below is the lease recovery algorithm for given file f. When a client dies, the same algorithm is applied to each file the client opened for write.

1. Get the DataNodes which contain the last block of f.
2. Assign one of the DataNodes as the primary DataNode p.
3. p obtains a new generation stamp from the NameNode.
4. p gets the block info from each DataNode.
5. p computes the minimum block length.
6. p updates the DataNodes, which have a valid generation  stamp, with the new generation stamp and the minimum block length.
7. p acknowledges the NameNode the update results.
8. NameNode updates the BlockInfo.
9. NameNode remove f’s lease (other writer can now obtain the lease for writing to f).
10. NameNode commit changes to edit log.

>Steps 3 through 7 are the block recovery parts of the algorithm. If a file needs block recovery, the NameNode picks a primary DataNode that has a replica of the last block of the file, and tells this DataNode to coordinate the block recovery work with other DataNodes. That DataNode reports back to the NameNode when it is done. The NameNode then updates its internal state of this block, removes the lease, and commits the change to edit log.
* 问题

1. 如果只针对最后一个block才能采用block recovery, 那么如果一个文件的中间block也出现副本不一致的问题, 该如何修复
2. 客户端写文件,不是先写一个副本, 然后第一个dn再写另一个dn的第二个副本吗, 为什么客户端失败会导致副本间数据不一致


3. pipeline recovery 
> During write pipeline operations, some DataNodes in the pipeline may fail. When this happens, the underlying write operations can’t just fail. Instead, HDFS will try to recover from the error to allow the pipeline to keep going and the client to continue to write to the file. The mechanism to recover from the pipeline error is called pipeline recovery.

> When one DataNode is bad, it removes itself from the pipeline. During the pipeline recovery process, the client may need to rebuild a new pipeline with the remaining DataNodes. (It may or may not replace bad DataNodes with new DataNodes, depending on the DataNode replacement policy described in the next section.) 


* dfs.client.block.write.replace-datanode-on-failure.best-effort
> which defaults to false. With the default setting, the client will keep trying until the specified policy is satisfied. When this property is set to true, even if the specified policy can’t be satisfied (for example, there is only one DataNode that succeeds in the pipeline, which is less than the policy requirement), the client will still be allowed to continue to write.

####  datanode pipeline process
![enter image description here](https://drive.google.com/uc?id=1btvOQAEX6xNWRQvmm1yOxgi-VKtbp8Al)
> 一旦有一个packet的第一个ack 回来,就可以发送下一个packet了
> 2和3的时序具有不确定性
* 问题
1. 针对[http://itm-vm.shidler.hawaii.edu/HDFS/ArchDocDecomposition.html](http://itm-vm.shidler.hawaii.edu/HDFS/ArchDocDecomposition.html)
> Communication between the Client and the NameNode uses RPC whereas communication between the Client and a DataNode uses streaming I/O through a socket.

两个协议有啥区别
> 如何并发的append, 和write有何不同, 多个副本的顺序不同?

> packet写第一个dn成功后就马上开始下一个packet的写, 还是等待所以dn ack之后才会开始?
答案: data queue 中的packet发送之后会转移到ackQueue, 然后等待一整个block里面的数据都完全ack之后, 才会发送下一个block

> pipeline中有一个dn写失败, 是会等待使用新的dn去替换还是用继续写剩余的pipeline中的dn, 然后并发的去找一个dn去补足pipeline中的dn数量(补到副本数)
答案: pipeline recover发生的时候会在满足replace-datanode-policy的时候会重建整个pipeline, 并且会阻塞发送, 新的pipeline可能和之前的pipeline有很大不同除非已经接受到数据的dn, 因为可能要考虑节点远近等原因.
#### write Consistency support
> When a client reads bytes from an rbw(replica being writen) replica, the DataNode that it reads from may not make all the bytes that it received visible to the client.(只有ack的那部分packet 可见)

> Each rbw replica maintains two counters:

1. BA: number of bytes that have been acknowledged by the downstream DataNodes. Those are the bytes that the DataNode makes visible to any reader.
2. BR: number of bytes that have been received for this block, including the bytes written to its block file and in‐DataNode‐buffer bytes

#### block report
>Each block report contains two lists: one for finalized replicas, one for rbw. The finalized replicas list include finalized replicas and underrecovery replicas whose old state are finalized. The rbw replica list includes rbw replicas, rwr replicas, and rur replicas whose old state are not finalized. Anrbw replica’s length is its bytes received (BR). The length of an rwr is a negative number.

> replica’s state : <DataNode, blck_id, blck_GS, blck_len, isRbw>


#### Handling-Small-Files-on-Hadoop-with-Hive-and-Impala
* 威胁
1. nn消耗大量内存
2. nn GC时间变长
3. hive 查询时间变长

* 小文件的来源
4. Trickling data – Data ingested incrementally and in small batches can end up accounting for a large number of small files over time. Consider running regular jobs to compact the small files.
5. Large number of mappers/reducers – MapReduce jobs and Hive queries with large number of mappers or reducers can generate a number of files on HDFS proportional to the number of mappers (for Map-Only jobs) or reducers (for Map-Reduce jobs)
6. Over-partitioned tables – Partitioned Hive tables with a small amount of data per partition. 

解决手册: 
1. [如何在Hadoop中处理小文件-续](https://mp.weixin.qq.com/s?__biz=MzI4OTY3MTUyNg==&mid=2247493657&idx=1&sn=1ed62012b73bc966949a7f5df469106a&chksm=ec293810db5eb10688beabd5495c6ee84aa73161d76283add0aee347c03d28f5bab802789e76&mpshare=1&scene=1&srcid=1127fhupWJ01oq4rKRgBRlYi#rd)
2. [如何在Hadoop中处理小文件](https://mp.weixin.qq.com/s?__biz=MzI4OTY3MTUyNg==&mid=2247492796&idx=1&sn=5e15a9060f234d24c1fb112d9158970c&chksm=ec2934b5db5ebda381ced8a99e0170229e57c0692a03d73a087853f99a09a3ee9d4e83d57267&mpshare=1&scene=1&srcid=112702C9Rz8xmapnKTs6NAc8#rd)

### hdfs 配置调优
https://hadoop.apache.org/docs/r2.7.1/hadoop-project-dist/hadoop-hdfs/hdfs-default.xml

 
* dfs.client.block.write.retries(默认3) 
>The number of retries for writing blocks to the data nodes, before we signal failure to the application

>  这个配置是DFSclient写入datanode的重试次数， 当超过这个次数，不再重试， 如果dfs.client.block.write.replace-datanode-on-failure.policy设置为ALWAYS，这时NN会被通知去寻找下一个可用的datanode并加入已有的Pipeline中，继续写入操作

* dfs.client.block.write.replace-datanode-on-failure.enable(默认true)
> If there is a datanode/network failure in the write pipeline, DFSClient will try to remove the failed datanode from the pipeline and then continue writing with the remaining datanodes. As a result, the number of datanodes in the pipeline is decreased. The feature is to add new datanodes to the pipeline. 

* dfs.client.block.write.replace-datanode-on-failure.policy(默认 DEFAULT)
>This property is used only if the value of dfs.client.block.write.replace-datanode-on-failure.enable is true. ALWAYS: always add a new datanode when an existing datanode is removed. NEVER: never add a new datanode. DEFAULT: Let r be the replication number. Let n be the number of existing datanodes. Add a new datanode only if r is greater than or equal to 3 and either (1) floor(r/2) is greater than or equal to n; or (2) r is greater than n and the block is hflushed/appended. **(这个existing dn指的是集群中的所有dn)**

* dfs.client.block.write.replace-datanode-on-failure.best-effort(默认false)
> This property is used only if the value of dfs.client.block.write.replace-datanode-on-failure.enable is true. Best effort means that the client will try to replace a failed datanode in write pipeline (provided that the policy is satisfied), however, it continues the write operation in case that the datanode replacement also fails. Suppose the datanode replacement fails. false: An exception should be thrown so that the write will fail. true : The write should be resumed with the remaining datandoes. Note that setting this property to true allows writing to a pipeline with a smaller number of datanodes. As a result, it increases the probability of data loss.

> 假设上面那个policy 满足的情况下, 并且替换的dn也失败的情况下,若 best-effort == true, 则会一直找所有dn, 并重试写; 若best-effort ==false, 则直接抛出异常

* what is hdfs safemode?
> On startup, the NameNode enters a special state called Safemode. Replication of data blocks does not occur when the NameNode is in the Safemode state. The NameNode receives Heartbeat and Blockreport messages from the DataNodes. A Blockreport contains the list of data blocks that a DataNode is hosting. Each block has a specified minimum number of replicas. A block is considered safely replicated when the minimum number of replicas of that data block has checked in with the NameNode. After a configurable percentage of safely replicated data blocks checks in with the NameNode (plus an additional 30 seconds), the NameNode exits the Safemode state. It then determines the list of data blocks (if any) that still have fewer than the specified number of replicas. The NameNode then replicates these blocks to other DataNodes.**(nn 启动的时候会进入safemode, 该模式只允许只读, 该模式下, dn会报告所有的block replica的状态, nn将之和内存中block的状态比较, 如果达到最小副本数(默认是1)的block的比例达到 dfs.safemode.threshold.pct, 默认是 99.9%, 则退出安全模式, 否则会一直等到这个值达到)**

> 强制退出safemode 命令: sudo -u hdfs hdfs dfsadmin -safemode leave

#### hdfs 磁盘容量问题

* 单个dn的单个volume的容量几乎用完, 会有何影响? 这个如何优化? 

使用available-space- volume-choosing- policy来对单个Datanode的不同volume进行再平衡 https://www.cloudera.com/documentation/enterprise/5-12-x/topics/admin_dn_storage_balancing.html 

* 某些dn 容量快满, 这个如何优化, 使用balance ?

如果其他DN还有很多空间， 您可以使用balance， 但是如果您的上层有HBASE服务的话， 请在合适（业务不太多，对hbase性能不敏感的时候）的时候进行rebalance， 因为HDFS reblanace会破坏HBase的localility，进而影响hbase性能。

#### 疑问


* 如何上传大文件到hdfs?
  * flume, spark local file(解析本地文件的问题下次再看了), mapreduce
  * 可以把大文件分成多个小文件.
  * 主要受限于带宽.
* TCP 的time_wait 理解
* client 写文件的时候, 不同 block 写到同一个dn上面是并行的吗 ?
答: 不是, 一个block到一个block




### MapReduce
* 大致过程
  * map 
      
    输入被分片到多个map, 每个map的输出先写到内存, 缓冲区满后会写入磁盘, 写磁盘之前会根据reducer划分为多个partition(partition数量和reducer task的数量一致, 可实现自定义的Partitioner)  每个partition中会根据key进行排序,若有combiner,则随后运行(运行条件: 溢出文件达到 min.num.spills.for.combine ), 把一个map(注意是一个map)中的输出进行reduceByKey, 能减少数据量, 然后再写到磁盘中.
  * reduce
    
    先把map的输出复制, 一旦有一个map的输出完成, 就进行复制, 复制完成后, 进行merge, 将多个map输出合并
* How Many Maps?

 由block的数量决定 
* how many reducers ? 
>The right number of reduces seems to be 0.95 or 1.75 multiplied by (<no. of nodes> * <no. of maximum containers per node>).With 0.95 all of the reduces can launch immediately and start transferring map outputs as the maps finish. With 1.75 the faster nodes will finish their first round of reduces and launch a second wave of reduces doing a much better job of load balancing.Increasing the number of reduces increases the framework overhead, but increases load balancing and lowers the cost of failures.

#### map的输入
默认的基类是FileInputFormat, 根据block size来split, 剩余的大小不足一个block size也会占据一个block, 且一个文件占据一个block;  一个split对应一个map的输入; 
* 分片大小的计算
> splitSize = max(minSize, min(maxSize, blockSize))
* small files problem
  * 问题
    * 每个文件都是一个对象, nm需要存储文件的元数据在内存中, 会占用大量的nm内存, 降低实际的存储能力.
    * 用作计算的话, 会降低性能, 花费大量的寻址时间和任务的启动和资源的释放时间.
  * 解决办法
  若输入是大量的小文件, 可以切换成CombineFileInputFormat(需要自己实现? ), SequenceFile(将多个小文件合并成一个seqFile), hadoop har 将多个小文件作为一个map的输入, 

* 多个文件一个分片(一个map处理速度非常快)
提高分片大小
* 超过分片大小的文件, 不想拆分
提高分片大小; 重写isSplitAble(), 返回false

* 一个文件作为一条record, 例如将多个小文件合并成一个顺序文件
以上问题用spark如何处理


### yarn
> 架构

![](https://drive.google.com/uc?id=1F8GwegZOrNUYmnBu5_RK-iFoH-YQ_pPI)
主要参考 [http://hadoop.apache.org/docs/r2.6.5/hadoop-yarn/hadoop-yarn-site/YARN.html](http://hadoop.apache.org/docs/r2.6.5/hadoop-yarn/hadoop-yarn-site/YARN.html)

* 大致流程
首先yarn 分配container来启动AM ; 然后AM注册到RM上, 让client可以知道AM的存在, 直接与AM通信; 然后AM 与RM请求资源分配; 然后AM启动container 并把相关信息提供给NM; application code在container上面执行, 并报告状态和进度等信息给AM; application执行完之后, AM从RM上面注销并返回container等资源.

* resourceManager
分为 scheduler和 ApplicationsManager 两个组件
scheduler 只负责资源的收集, 分为  CapacityScheduler and the FairScheduler, 不负责application的监控等
ApplicationsManager 负责接收 application的提交请求, 并分配AM, 并且负责监控AM的状态, 失败重启等.

* ResourceRequest and Container
<resource-name, priority, resource-requirement, number-of-containers>. 
resource-name is either hostname, rackname or * to indicate no preference. In future, we expect to support even more complex topologies for virtual machines on a host, more complex networks etc.
priority is intra-application priority for this request (to stress, this isn’t across multiple applications).
resource-requirement is required capabilities such as memory, cpu etc. (at the time of writing YARN only supports memory and cpu).
number-of-containers is just a multiple of such containers.
	
* CapacityScheduler
> 可继承的队列
> Capacity Guarantees: 每个队列有资源的硬限制和软限制.
> Security: 每个队列有独立的ACL
> 弹性(Elasticity) : 队列可以获取到超过其容量的资源, 如果集群有空闲资源的话

> 多租户(Multi-tenancy): 保证每个application, user, queue都不能独占整个集群的资源.
> Runtime Configuration : 可支持运行时配置.
> Drain applications : 可以控制queue的启动和停止, 停止的时候不能接受新的application. 

* Fair Scheduler
> 单个application会占满所有资源，当有更多的application会逐步分配资源，达到fair

* nodeManager
> 每个节点一个, 负责监控container 的状态, 监控资源的使用情况并上报给RM

* ApplicationMaster
每个application一个, 负责与RM协商并获取 resource container的情况, 并监控container.
解决了集群规模上升导致RM出现瓶颈的问题, RM只负责协调资源, 每个应用由AM监控, 大大减小了RM的压力. 
* ApplicationMasterProtocol
> The protocol between a live instance of ApplicationMaster and the ResourceManager.
This is used by the ApplicationMaster to register/unregister and to request and obtain resources in the cluster from the ResourceManager.

* ContainerManagementProtocol
>The protocol between an ApplicationMaster and a NodeManager to start/stop containers and to get status of running containers.
If security is enabled the NodeManager verifies that the ApplicationMaster has truly been allocated the container by the ResourceManager and also verifies all interactions such as stopping the container or obtaining status information for the container.

* container
> It represents a resource on a single node at a given cluster.
A container is supervised by the node manager, scheduled by the resource manager

#### hdfs 命令
* 目录授权
> hadoop fs -chown root:root /hive 把hive目录授权给root

**问题**
* MR和spark shuffle的过程, 以及调优
  * spark shuffle见 spark笔记
  * MR shuffle
* hive 分区分桶
* hive ACID的支持
* hive和 mysql的区别

<!--stackedit_data:
eyJoaXN0b3J5IjpbMTUwMzYyOTEzOSwtMjMxOTExMDE3LDYzNj
cwNjEzMiw5NDcyNjEwMywtMTQ2MDc3NDkxLDEzMjk4NjI3MTgs
LTExMzg4NjY4OTYsODk5OTUyNjAsNTg0ODcwMDQ1LDEyODI3Mz
UzODRdfQ==
-->