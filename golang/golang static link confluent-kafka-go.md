背景: 
golang 的 之前的 kafka 库都不太好用, sarama-kafka 已经不维护了, 而且自动提交 offset 的时候有 bug, 会导致重复消费; segmentio-kafka 不支持消费 topic 的 pattern, 并且设置了 groupid 之后就不能设置 auto.offset.reset 了, 真是够了. 然后就选用 [confluent-kafka]([https://github.com/confluentinc/confluent-kafka-go](https://github.com/confluentinc/confluent-kafka-go)) , 但是这个是用 go 封装了 c++的代码, 核心代码是 C++, 所以需要 build 的时候依赖 [librdkafka](https://github.com/edenhill/librdkafka),  这个 c++的库, 本来在物理机上很简单的, 但是需要通过 Jenkins 在 docker 上面 build, 问题可能就多了.  

依赖的方法很多, 当机器上面有这个包了之后, 接下来分为dynamic link 和 static link. 
依赖的方法: 
1. 直接用各个系统的包管理器, 例如Ubuntu apt-get install, 但是有些源下载下来的版本过低, 所以采用 confluent 的 repository. 但是! apt-get update 又出现 some indices update failed, 没有更详细的报错, 故作罢, 本来这个是最简单的方式, 而且 docker container 重启后会新建一个, 不会影响其他的环境. 
```
"wget -qO - https://packages.confluent.io/deb/5.2/archive.key |apt-key add -",
"echo \"deb [arch=amd64] https://packages.confluent.io/deb/5.2 stable main\" >> /etc/apt/sources.list",
"apt-get update",
"apt-get install -y librdkafka-dev=1.1.0~1confluent5.3.0-1",
```

2. 在一台 Ubuntu 上面编译好放到代码路径中, 但是有可能因为编译的系统环境和运行的系统环境不同而有风险, 公司的环境基本一致, 故觉得这个方法最好, 不需要依赖外部的网络, 速度又快. 但是编译好之后需要将包含rdkafka.pc 的文件夹添加到这个环境变量 PKG_CONFIG, 但是始终无法让这个环境变量生效, 莫非是嵌套 shell 的问题? 
3.  最后在这个无法找到 PKG_CONFIG 的报错中, 发现confluent-kafka-go 有个 issue : [Static build failing with missing rdkafka-static.pc #316](https://github.com/confluentinc/confluent-kafka-go/issues/316) , 介绍了一个分支是预先将 build 之后的包放到了confluent-kafka 这个的包路径中, 然后终于成功了, 可以看下他是如何预先编译的, 因为还没有正式合到 release , 

> Written with [StackEdit](https://stackedit.io/).
<!--stackedit_data:
eyJoaXN0b3J5IjpbNzg2ODk4MzU3LDE3NDk1MjE0OTUsLTIzOT
E4ODkzMywtMTM3NTU0MzY3NV19
-->