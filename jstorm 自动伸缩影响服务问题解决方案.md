### 架构
目前公司用的监控系统的架构是: 客户端收集日志吐到kafka, 然后jstorm 消费kafka的消息, 存储到opentsdb, 用grafana 可视化

> jstorm 架构

日志监控只有一个topology, 如果有系统需要新增对日志的监控, 则通过配置页面配置一条清洗过滤的规则, 满足规则的日志被存储到opentsdb, 一个系统所有的清洗日志对应一个component, 具体来说是对应一个spout, 满足条件的再下发到bolt(opentsdb 处理), 这个架构的优劣暂且不谈, 历史原因, 还没重构.

> 带来的问题
1. 一个系统对应一个集合的清洗规则, 对应一个spout, 对应一个kafka topic, 新增一个系统, 需要新增spout, 目前jstorm 不支持对一个topology新增spout和bolt, 只支持动态更改topology的配置, 并选择对哪些spout/bolt生效
2. 一个系统对应一个kafka的topic, 若消息量突然增大, 则这个spout不能满足需求, 需要动态新增task和worker, 这个jstorm 本身是支持动态缩容的, [http://jstorm.io/ProgrammingGuide_cn/AdvancedUsage/DynamicAdjust/Parallel.html](http://jstorm.io/ProgrammingGuide_cn/AdvancedUsage/DynamicAdjust/Parallel.html), 但是经过我们初步生产测试会影响整个topology 的sendTPS, 就是会影响其他系统的关键图表, 造成抖动, 这个无法接收

> 目标
1. 针对**问题2** 因为官网文档中介绍, 不加入 -r 参数是不会引起task的重分配的, 所以性能问题让我们很疑惑, 想深入了解并调优JStorm, 做到不影响topology的性能,

> 解决思路

之前不了解JStorm的时候走的弯路就不说了, 正确的解决思路应该如下:  task分配到worker的策略不能变, 老的task还是分配到原先的worker, 否则会造成旧有的task shutdown, 新的task create必然有性能消耗. 

> JStorm 架构

简单说下, 由nimbus, supervisor, worker, task 组成, topology和spout/bolt暂且不说, 跟这个问题关系不大. 
nimbus: 相当于集群的领导者, 跟yarn的RM 地位一致, 处理客户端提交的请求, 例如命令行执行的各种命令; 处理supervisor, worker, task的分配策略等
supervisor: 可以理解为yarn的NM, 负责管理各个工作节点, 分配worker
worker: topology的执行进程, 由各个执行线程task组成, 例如更新topology的配置, 是由nimbus执行配置的更新, 并更新到zk的节点, worker watch到event的变化之后, 更新到具体的执行进程中去.

> 解决步骤

首先是要深入了解JStorm的rebalance的过程, 我采取远程调试nimbus, 看日志, 看源码的三种方式结合

一. 远程调试nimbus
一开始看源码还比较陌生, 就想通过远程调试测试服务器的方式, 能够看到一些关键变量的值, 排除一些看不懂的代码的干扰.
jstorm使用 jstorm nimbus启动nimbus , 后面想到可以直接用 ps -ef|grep nimbus 的输出信息直接结合远程调试的命令启动, 如下图, 
![enter image description here](https://drive.google.com/uc?id=14DXapVXhDpSOK6bpzqdgdPmM9CMVeMKV)
idea的配置如下: 
![enter image description here](https://drive.google.com/uc?id=1nKP1VbmsOfFf7kI7HoTSCkUYbHHws5tN)

nimbus 启动之后会向服务器的5005端口启动一个进程 a, idea随时可以启动remote debug attach到a 进程进行调试

二. 查看日志
日志主要看的是worker和nimbus两个, 都在jstormHome/logs下面, rebalance 的主要流程是nimbus 接受到客户端提交的rebalance的命令, 生成的新的assign, 并更新到zk, worker watch到对应的event, 根据task的变更, 有可能需要create或者shutdown task. 

三. 源码解析
不是关键的代码就一笔带过了.  zhon

1. thriftClient 客户端提交rebalance命令, rebalance.main 方法提交.
2. nimbus 接受到状态变化, StatusTransition 初始化statusTransitionCallback, 关键是DoRebalanceTransitionCallback, 生成TopologyAssignEvent 推送到 TopologyAssign 处理
3. 






> Written with [StackEdit](https://stackedit.io/).
<!--stackedit_data:
eyJoaXN0b3J5IjpbLTE0OTA0ODIwNTAsODkxMDQ0MDg5LC0xMz
M4MzQwNywtMTgwODYxNjk0MCwtMTA5MTk0MjYyMCwxMDM1MTI5
NjYzLC0xMDQ2MzQwMzk0XX0=
-->