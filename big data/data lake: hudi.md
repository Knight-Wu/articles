https://hudi.apache.org/docs/use_cases.html

### use case
1. 更方便的 update and insert to hdfs; 而且减少小文件问题
2. 能够提供 fresh data 以便更好的进行 near realtime 查询, spark sql 和 presto
3. 增量的处理管道, 例如上游数据延迟, 下游多个表需要重跑, hudi 能够等数据到来之后自动触发下游多个表重跑


> Written with [StackEdit](https://stackedit.io/).
<!--stackedit_data:
eyJoaXN0b3J5IjpbLTMyNzQzNDc4OV19
-->