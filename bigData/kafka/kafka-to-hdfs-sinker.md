### 前言
距离sinker 上线已经快一个月了, 相当可靠没有一个error( 我把error 日志接入了企业微信, 前几天是有些提心吊胆的) , 于是终于不犯懒去写一篇回顾, 趁着还记得, 这是非常有必要的
### 背景
flume 接tracking 的数据写到hdfs 的hive 外表, 一是会丢消息, 集中在凌晨跑批 hdfs 繁忙的时候, 没有找到相关异常, 也没有在flume 找到issue, 一直没解决, 严重的时候有百分之十以上的丢消息, 过滤完之后的tracking kafka 平均 12w qps, 峰值 25w qps, 活动峰值 35w qps, 大约每天有百分之十的丢消息, 以12w qps 算, 一天消息有100 亿条, 百分之十有10 亿条, 这个量级已经无法忍受了; 二是hdfs 挂掉的时候, flume 会有大量没有正常close 的文件, hdfs 恢复之后, 在lease hard limit 到期之前, 这些文件不会被recover lease, 这些文件会导致hive 无法正常读取, 需要手动执行命令修复: 
```
hdfs debug recoverLease -path pathA
```
当文件数很多的时候, 过程会很慢, 长达几个小时, 
 故提出了两个思路, 一是解决flume 的异常(只是丢数的问题) , 二是新开发一套系统替换flume . 

### 目标
经后续与关注flume 异常的同学以及领导讨论, 判断新开发一套系统的时间会比较短, 以及可预期, 于是开始设计新系统. 新系统需要满足以下目标: 
1. 外部系统kafka, hdfs 挂掉, 不会丢消息, 系统block 住, 直到故障恢复, 不需手动干预
2. 在kafka exactly once 的情况下, 不丢不重复消息. 
3. 满足现有活动的max qps

### 设计
一开始认为hdfs 上传是一个原子操作, 要不就上传失败无法被其他客户端看到, 要不就上传成功, 但是经过实验: 上传一个很大的文件上去, 中途 ctrl+c, 文件只上传了部分, 并经过一个小时(hard recover limit), 文件大小又增加了一些( close 触发了flush) , 证明hdfs 上传不是一个原子性质的, 故在文件名上加上了上传前的文件长度, 以判断文件的完整性. 文件名: partitionNum#consumeMaxOffset#fileLen

* 流程图
![enter image description here](https://drive.google.com/file/d/1fiDEGMxJmxxdXbhpOPy5dbZLEykKs25H/view?usp=sharing)


![enter image description here](https://drive.google.com/uc?id=1Kq1N5-yNbLI1dCdcRCRDCoLJQfpt8S6X)

### 难点
* write successFile 

因为需要告诉下游某一个path 的数据已经全部到达, 需要有多个client 在不同机器同时append partition number 到一个文件(LOADING ), 然后最后一个partition 线程rename 成 SUCCESS, 所以不同client 同时append 会出现lease 竞争的问题, 采取的策略就是碰到该异常重试, 但是在close file 也出现异常的时候会导致lease 没有释放, 导致其他客户端无法正常append, 最后采取的策略是若close 出现异常, 则执行 recoverLease(path) 这个函数

* hdfs crc exception

一个path 下只有一个线程写一个文件, 一开始文件后缀是startOffset, 然后需要先在本地 rename 成 lastOffset 再上传, 一开始只rename 了数据文件, 导致遗留了很多crc 文件, 并且上传一段时间就出现 crcException, 后面把crc file 也rename 就没出现这个异常. 

* 类似 hdfs nn 双主现象, 一个partition 被同时两个实例消费

在测试环境模拟 hdfs 和kafka 挂掉的现象, 通过屏蔽某一台 local 机器上 hdfs 8020, kafka 9092 端口, 该实例的partition 已经被其他实例消费, 但是由于他自身无法连接kafka 和hdfs 他并不知道, 当hdfs 首先恢复的时候, 会上传partition 的部分文件, 导致消息重复, 解决办法: 当上传重试时间超过 kafka max poll ms 的时候就放弃上传, 如何发现这个问题: 把partition 作为字段信息写到hive 表里, 发现有某些partition 里面有重复的消息, 通过查看多个实例的日志, 发现该partition 已经被其他实例接管之后, 仍然有旧实例上传文件. 

* rebalance 

首先kafka-2.2.1 rebalance  过程: stop fetch msgs -> onPartitionsRevoked(previous assigned partitions) -> reassign -> onPartitionsAssigned(partitions that are now assigned to the consumer) -> start to fetch msgs      

本来一开始是在 onPartitionsRevoked() 这个函数中做剩余文件的上传等清理工作, 但是考虑到实现的复杂性以及实际上并不会加快很多rebalance 的过程, 就改为全部抛弃内存和磁盘中的文件, 在 onPartitionsAssigned() 中做offset 的寻找等工作, 如下图显示, 首先找到都有successFile 的上一个hour, 例如 country/dt/hour=16 都有successFile了, 那么就从 hour=17 开始找, 则先找出每一个path 的maxOffset, 例如找出 country=ID/dt=2020-06-01/hour=17  的最大的offset, 再找出 min of (path1MaxOffset, path2MaxOffset...) 作为seek offset, 为什么第一次是max, 第二次是min 呢, 第一次是max 容易理解, 因为每个path 只有一个线程在消费, offset 递增的, 但是第二次是max , 因为是无法确保max offset 之前, 所有path 的文件都顺利上传到hdfs 了, 因为在  onPartitionsRevoked() 做的是del, 就算做的是upload 若hdfs 挂了, 则会遗漏一部分消息在磁盘上, 所以以hdfs 的offset 为准, 选一个min , 然后每个pathx 对应一个thread, 当consume offset 大于 pathxMaxOffset 才开始写入文件, 此时又碰到另一个极端情况, 假设程序回到了一天前开始消费, successFile 之后有好几个path, 那么需要很久时间才能追得上, 每一个 hour 对应的thread 的清理时间都是二十多分钟左右(若一个path 有二十多分钟没有消息写入, 则清理对应的thread), 那么超过二十分钟的情况下, 实际消费的offset 又在此时之前呢, 所以需要判断此时消费的offset 大于path 对应的offset , 才能del; 这只是del 的第一个条件, 同时需要判断程序并没有因hdfs 和kafka crash 而导致的block, 因为此时block 了, 程序不能清理thread 也不能写successFile,    

* 如何保证文件上传前后的一致性

以下几个情况是会导致文件不一致的:
1. 使用线程池上传文件, 每个线程上传一个文件, 若文件上传过长时间, 则在程序退出的时候可能会被interrupt, 导致上传不完整, 解决办法: 加锁, 若上传正在进行, 则不能被 shutdownNow()
2. 进程被kill -9, 导致: file is not a snappy file 或者文件长度和上传前不一致
3. hdfs crash(可能)

故需要增加fileLength 到文件名, 通过某种方式触发, 1通过加锁解决, 2 会触发rebalance, 则会check 文件名, 判断实际长度是否一致, 3 通过调下kafka max poll ms, 每次hdfs crash 导致的主线程block 都会触发rebalance. 
                                                                                                                                                                                                  
> Written with [StackEdit](https://stackedit.io/).

<!--stackedit_data:
eyJoaXN0b3J5IjpbLTQzODAzMDc5MF19
-->
