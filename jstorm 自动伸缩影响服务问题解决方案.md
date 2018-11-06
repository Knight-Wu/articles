### 架构
目前公司用的监控系统的架构是: 客户端收集日志吐到kafka, 然后jstorm 消费kafka的消息, 存储到opentsdb, 用grafana 可视化

> jstorm 架构

日志监控只有一个topology, 如果有系统需要新增对日志的监控, 则通过配置页面配置一条清洗过滤的规则, 满足规则的日志被存储到opentsdb, 一个系统所有的清洗日志对应一个component, 具体来说是对应一个spout, 满足条件的再下发到bolt(opentsdb 处理), 这个架构的优劣暂且不谈, 历史原因, 还没重构.

> 带来的问题
1. 一个系统对应一个集合的清洗规则, 对应一个spout, 对应一个kafka topic, 新增一个系统, 需要新增spout, 目前jstorm 不支持对一个topology新增spout和bolt, 只支持动态更改topology的配置, 并选择对哪些spout/bolt生效
2. 一个系统对应一个kafka的topic, 若消息量突然增大, 则这个spout不能满足需求, 需要动态xin



> Written with [StackEdit](https://stackedit.io/).
<!--stackedit_data:
eyJoaXN0b3J5IjpbLTE0OTQ2NzE2NzMsLTEwNDYzNDAzOTRdfQ
==
-->