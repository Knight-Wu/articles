### 手动计算consume 和 produce 的速度
date;./kafka-consumer-groups.sh --describe --group groupid  --broker:9092| sort > lag.msg
执行这个命令两次, 记录时间, 然后将两个 lag.msg 导入到 google sheet, Data 菜单下


> Written with [StackEdit](https://stackedit.io/).
<!--stackedit_data:
eyJoaXN0b3J5IjpbLTI4ODEwMjE0OV19
-->