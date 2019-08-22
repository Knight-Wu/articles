背景: 
golang 的 之前的 kafka 库都不太好用, sarama-kafka 已经不维护了, 而且自动提交 offset 的时候有 bug, 会导致重复消费; segmentio-kafka 不支持消费 topic 的 pattern, 并且设置了 groupid 之后就不能设置 auto.offset.reset 了, 真是够了. 然后就选用 confluent-kafka, 但是这个是用 go 封装了 c++的代码, 核心代码是 C++, 所以需要 build 的时候依赖 [librdkafka](https://github.com/edenhill/librdkafka),  这个 c++的库

依赖的方法很多, 当机器上面有这个包了之后, 接下来分为dynamic link 和 static link. 
依赖的方法: 
1. 直接用各个系统的包管理器, 例如Ubuntu apt-get install, 但是有些源下载下来的版本过低, 
```
"wget -qO - https://packages.confluent.io/deb/5.2/archive.key |apt-key add -",
      "echo \"deb [arch=amd64] https://packages.confluent.io/deb/5.2 stable main\" >> /etc/apt/sources.list",
"apt-get update",
"apt-get install -y librdkafka-dev=1.1.0~1confluent5.3.0-1",
```


> Written with [StackEdit](https://stackedit.io/).
<!--stackedit_data:
eyJoaXN0b3J5IjpbOTQ4OTIyMTExLC0yMzkxODg5MzMsLTEzNz
U1NDM2NzVdfQ==
-->