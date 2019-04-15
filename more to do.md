### TODO
* 完整的洗数流程, 数据集市的数据流向以及相关的技术组件的选取

* JStorm使用的 lmax disruptor 原理
* 记忆力的研究
* opentsdb 数据结构研究
* holt winters 算法用R 跑出例子, 用于监控系统告警阈值, 智能化预测
* flink的学习, flink 和JStorm的对比
* flink 动态伸缩jira 跟踪
* google drive add articles to read-later-list
* datanode 磁盘中某个volume 的容量快满时会有何影响
* yarn container如何用起来. 写一个使用的小demo
* spark, hadoop release notes

* https://issues.apache.org/jira/browse/HDFS-9572 这个需要等待comment 回复
* https://en.wikipedia.org/wiki/Chaos_engineering 软件工程的严谨思想
* 为何当没有用户权限的情况下, ps -ef能查其他用户进程, netstat -anp却查不到
* 多个应用服务器的性能之和如何算, 
* Build highly concurrent, distributed, and resilient message-driven applications on the JVM [http://akka.io](http://akka.io/)
* spark executor task java.net.connectException 拒绝连接
* spark.yarn.executor.memoryOverhead 结合内存理解, 如何调优
* yarn 虚拟内存和虚拟cpu 用来干啥
* yarn 用mr 作业收集container使用情况的原理和作用
* 为啥nio 要用byte buffer, 懂了io 模型之后还要知道设计api 的时候, 各种api 的好处以及作用

* zk 的一致性如何保证
* ConcurrentLinkedQueue ,blockingQueue 适用场景
* 支持事务的消息框架
* 微服务框架
* 泊松分布 https://www.zhihu.com/question/26441147
* java threadpool 如何从空闲的worker 中选择一个来执行, 
*  java write file 底层
* hbase 的设计结构和业务字段的设计
* hbase 和mysql 场景
* kafka 的架构
* 邮件发出每天目录的增长量
* 保证小文件每天能跑, 设置另一个轮训是否完成的线程
* 超时两小时 kill 的bug 解决
* hive-staging 目录更改. 
### DOING



### DONE

* 如何能快速搞懂, 例如想之前看dubbo 为什么需要有代理这层的时候, 一直搞不懂这层, 就不能像调用本地服务的时候调用远程服务, 感觉是又想快速搞懂, 跳过或者忽略看的一些代码和文档, 觉得没有关系就不看, 就像看书一样, 一扫而过, 有些不懂的地方就跳过, 导致看了一个小时还是感觉什么都不懂, 而且很混乱, 时间过了但是却没有真正搞懂, 思路却因为融合了更多的问题而变得像一团的浆糊, 陷入泥潭, 而且浪费了时间, 心态也不如一小时前, 说白了就是看了一圈下来, 感觉很多不懂, 但是却无法提出一个问题, 也就没有了具体的方向, 所以总结下来, 在碰到一个比较陌生的领域的问题的时候, 搞懂背景知识是相当重要的, 而且在学习背景知识的过程中, 不能跳过, 看一段英文或者文档, 代码都没有耐心是无法赚大钱的, 衡量什么能跳什么不能跳的前提是, 不会影响看下去感觉, 如果感觉无法看下去了, 证明是之前的某些东西没搞懂, 需要一一克服,** 然后就是如何快速搞懂那一个个小点了**

已查到资料
* 本地进程通信大量time_wait 连接
正常的, 消耗的内存和cpu都很少, 




* top 命令如何定位到某个线程的问题, 假设cpu 百分百如何排查
https://blog.csdn.net/flysqrlboy/article/details/79314521



* 为何当没有用户权限的情况下, ps -ef能查其他用户进程, netstat -anp却查不到
  mae rcte linsar, lassotu
  
* 如何进行hdfs 磁盘的balance
已查到资料
* 本地进程通信大量time_wait 连接
正常的, 消耗的内存和cpu都很少, 

* spark 集群容错的控制, 目前只知道task fail会触发 spark.task.maxFailures这个配置, 但是executor层, container层, job层, application层的配置控制还有待学习

已经总结

* maven archtype 直接构建flink-quick-start, 解决一个classNotFoundEx 

新下载了一个 maven ,就解决了这个问题, 可以用archetype:generate 去新建一个和模板一致的maven 项目top 命令如何定位到某个线程的问题, 假设cpu 百分百如何排查


* git 问题: 如果本地不小心删了一个文件, 怎么从remote 更新下来, 只更新这个文件; 如果想根据revision number 检出一个新的分支, 用于maven deploy , 如何操作: git reset revision_num; 在用 git pull 还原到origin/master 
见"git 原理" 已解决

* 如何知道hive on spark 整个application 运行阶段的内存使用情况, spark metric 好像不支持spark executor memory, 那么如何知道executor memory 或者cpu 等资源是否设置得合理? 

https://github.com/uber-common/jvm-profiler 这个可以试试

> Written with [StackEdit](https://stackedit.io/).
<!--stackedit_data:
eyJoaXN0b3J5IjpbLTI3NTExMTI1MywyMDU0MTMxNzkzLC0zMz
E5MTA2MTMsLTIxMzQ0MzI4MzgsMTg0OTE3NjQ1LDE4MjIxMDAx
NDAsNjQ3NzQ4ODExLDY0Nzc0ODgxMSwtMjEzMjkwNTIyNywtMT
c4MDc1NTkzLC0zOTUxNTgwOTIsLTEwNjk5MDg3NTgsLTE3MDY0
NTM3OTUsMTAwNjgwOTI4MCwtNDA0MjYxMDE3LDQyOTI2ODQxNi
wtNDg5MDExOTIsLTE3ODY2Njg2OTIsLTEyNTUwNzQxOTcsMTQ0
MDE4Njk1NF19
-->