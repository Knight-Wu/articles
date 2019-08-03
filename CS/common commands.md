### kafka
* describe consumer, get lag and offset
1. version>=0.9, offset store in a topic named "__consumer_offsets" in broker

```
./kafka-consumer-groups.sh --describe --group groupid  --bootstrap-server brokerA
```

2. version <0.9, offset store in zookeeper




> Written with [StackEdit](https://stackedit.io/).
<!--stackedit_data:
eyJoaXN0b3J5IjpbLTEwNDA3Njk3MiwtMzU0MzM4MTE4XX0=
-->