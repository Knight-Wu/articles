### kafka
* describe consumer, get lag and offset
1. version>=0.9, offset store in a topic named "__consumer_offsets" in broker

```
./kafka-consumer-groups.sh --describe --group groupid  --bootstrap-server brokerAddr
```

2. version <0.9, offset store in zookeeper
```
./kafka-consumer-groups.sh --describe --zookeeper zk:2181/rootDir --group groupid
```





> Written with [StackEdit](https://stackedit.io/).
<!--stackedit_data:
eyJoaXN0b3J5IjpbNjkzNTg3MDMsLTM1NDMzODExOF19
-->