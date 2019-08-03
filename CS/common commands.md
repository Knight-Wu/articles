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
* consume msg from cli
```
./kafka-console-consumer.sh --bootstrap-server brokeraddr --topic topic  --consumer-property group.id=sz-mydata-orderetl --max-messages 1
```




> Written with [StackEdit](https://stackedit.io/).
<!--stackedit_data:
eyJoaXN0b3J5IjpbLTEzNzgyMzY4MzQsLTM1NDMzODExOF19
-->