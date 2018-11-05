### 架构
目前公司用的监控系统的架构是: 客户端收集日志吐到kafka, 然后jstorm 消费kafka的消息, 存储到opentsdb, 用grafana 可视化

> jstorm 架构

日志监控只有一个topology, 有系统需要新增对日志的监控, 则通过配置页面配置一条清洗过滤的规则, 满足规则的日志被cunc




> Written with [StackEdit](https://stackedit.io/).
<!--stackedit_data:
eyJoaXN0b3J5IjpbLTgzMjE2NjA5MV19
-->