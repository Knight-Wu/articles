### TODO
* 完整的洗数流程, 数据集市的数据流向以及相关的技术组件的选取
* paxos 原理再次理解
* nio的buffer为何不能一次读取全部数据
* JStorm使用的 lmax disruptor 原理
* jstorm 为什么把rebalance 做成 thrift 的模式
* kafka 的学习成果总结
* 记忆力的研究
* opentsdb 数据结构研究
* holt winters 算法用R 跑出例子, 用于监控系统告警阈值, 智能化预测
* flink的学习, flink 和JStorm的对比
* google drive add articles to read-later-list
* centOS redhat内核关系, 以及在修改hostname的时候命令的不同
* datanode 磁盘中某个volume 的容量快满时会有何影响
* yarn container如何用起来. 写一个使用的小demo
* linux 文件权限和用户权限的彻底理解
* spark 集群容错的控制, 目前只知道task fail会触发 spark.task.maxFailures这个配置, 但是executor层, container层, job层, application层的配置控制还有待学习
* spark, hadoop release notes
* logback 性能测试 https://github.com/ceki/logback-perf
* top 命令如何定位到某个线程的问题, 假设cpu 百分百如何排查

* 为何当没有用户权限的情况下, ps -ef能查其他用户进程, netstat -anp却查不到
* git 问题: 如果本地不小心删了一个文件, 怎么从remote 更新下来, 只更新这个文件; 如果想把

### DOING
* maven archtype 直接构建flink-quick-start, 解决一个classNotFoundEx 


### DONE
* 如何进行hdfs 磁盘的balance
已查到资料
* 本地进程通信大量time_wait 连接
正常的, 消耗的内存和cpu都很少, 
> Written with [StackEdit](https://stackedit.io/).
<!--stackedit_data:
eyJoaXN0b3J5IjpbMzQ1NDQ3NzYwLC00MjY3ODMwOTMsLTE0NT
IxMDQ2Miw0MjkzOTQyOSwtNTQxOTYwNzM5LC0xMjcxNTU1NDA5
LC0yODM1MDM5MzcsMTM4NDQ0MDk3NCw5MjEwOTUwMCwtODM1Mz
c2MjcyXX0=
-->