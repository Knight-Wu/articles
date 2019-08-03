### kafka
* describe consumer, get lag and offset
1. version>=0.9, offset store in a topic named "__consumer_offsets" in broker


./kafka-consumer-groups.sh --describe --group groupid  --bootstrap-server brokerAddr


2. version <0.9, offset store in zookeeper

./kafka-consumer-groups.sh --describe --zookeeper zk:2181/rootDir --group groupid

* consume msg from cli, and auto commit from background

./kafka-console-consumer.sh --bootstrap-server brokeraddr --topic topic  --consumer-property group.id=groupid --max-messages 1

./kafka-console-consumer.sh --bootstrap-server brokeraddr --topic topic  --consumer-property group.id=groupid


* list all cousumergroup, not use --topic
./kafka-consumer-groups.sh  --list --bootstrap-server brokeraddr

* manual consume from topic="", but message is huge. not 

> Written with [StackEdit](https://stackedit.io/).
<!--stackedit_data:
eyJoaXN0b3J5IjpbNDMzODU1MDMsLTE5NjY5Mzg5ODEsLTM1ND
MzODExOF19
-->