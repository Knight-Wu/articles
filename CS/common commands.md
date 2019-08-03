### kafka
* describe consumer, get lag and offset
1. version>=0.9, offset store in a topic named "__consumer_offsets" in broker


>./kafka-consumer-groups.sh --describe --group groupid  --bootstrap-server broker:9092


2. version <0.9, offset store in zookeeper

>./kafka-consumer-groups.sh --describe --zookeeper zk:2181/rootDir --group groupid

* consume msg from cli, and auto commit from background

> ./kafka-console-consumer.sh --bootstrap-server broker:9092 --topic topic  --consumer-property group.id=groupid --max-messages 1

> ./kafka-console-consumer.sh --bootstrap-server broker:9092 --topic topic  --consumer-property group.id=groupid


* list all cousumergroup, not use --topic
>./kafka-consumer-groups.sh  --list --bootstrap-server brokeraddr

* manual consume from topic="", but message is huge. 
>echo "exclude.internal.topics=false" > ./consumer.config; bin/kafka-console-consumer.sh --consumer.config ./consumer.config --bootstrap-server broker:9092 --topic __consumer_offsets  --consumer-property group.id=groupid  --formatter "kafka.coordinator.group.GroupMetadataManager\$OffsetsMessageFormatter" 

output msg: 
[**groupId**,**topicName**,**partitionNumber**]::[OffsetMetadata[**OffsetNumber**,NO_METADATA],CommitTime 1520613132835,ExpirationTime 1520699532835]

about msg format class between different version: 
[https://medium.com/@felipedutratine/kafka-consumer-offsets-topic-3d5483cda4a6](https://medium.com/@felipedutratine/kafka-consumer-offsets-topic-3d5483cda4a6)


### redis
* redis-cli 
>redis-cli -h hostname -p port
* slotscan script 
https://drive.google.com/open?id=1DO3anQgMfAUSsb9JPyLZDhIfNhtMrvMP
### elasticsearch

### ssh

> ssh -NfL localport:targetAddr:targetPort remoteIp

只要 remoteIp 通过 telnet targetAddr:targetPort , 并不需要只在targetAddr
<!--stackedit_data:
eyJoaXN0b3J5IjpbLTIwOTU1NTY1MDMsLTE5NjY5Mzg5ODEsLT
M1NDMzODExOF19
-->