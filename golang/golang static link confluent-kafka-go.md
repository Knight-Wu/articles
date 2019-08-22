背景: 
golang 的 之前的 kafka 库都不太好用, sarama-kafka 已经不维护了, 而且自动提交 offset 的时候有 bug, 会导致重复消费; seg-kafka 不支持消费 topic 的 pattern, 并且设置了 groupid 之后就不能设置 auto.offset.reset 了, 真是够了. 然后就选用 confluent-kafka, 但是这个是用 go 封装了 


> Written with [StackEdit](https://stackedit.io/).
<!--stackedit_data:
eyJoaXN0b3J5IjpbMjUzNzc3ODAxLC0xMzc1NTQzNjc1XX0=
-->