### 手动计算consume 和 produce 的速度
date;./kafka-consumer-groups.sh --describe --group groupid  --broker:9092| sort > lag.msg
执行这个命令两次, 记录时间, 然后将两个 lag.msg 导入到 google sheet, 但是初始数据都在一列, 此时 选择 Data 菜单下split text to columns, 就可以分成多列, 通过两个时间点内的 current-offset 的相差计算消费速度, log end offset(是最新的一条消息进入


> Written with [StackEdit](https://stackedit.io/).
<!--stackedit_data:
eyJoaXN0b3J5IjpbOTY5NjE3MDU2XX0=
-->