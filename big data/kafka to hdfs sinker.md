### 前言
距离sinker 上线已经快一个月了, 相当可靠没有一个error( 我把error 日志接入了企业微信, 前几天是有些提心吊胆的) , 于是终于不犯懒去写一篇回顾, 趁着还记得, 这是非常有必要的
### 背景
flume 接tracking 的数据写到hdfs 的hive 外表, 一是会丢消息, 集中在凌晨跑批 hdfs 繁忙的时候, 没有找到相关异常, 也没有在flume 找到issue, 一直没解决, 严重的时候有百分之十以上的丢消息, 过滤完之后的tracking kafka 平均 12w qps, 峰值 25w qps, 活动峰值 35w qps, 大约每天有百分之十的丢消息, 以12w qps 算, 一天消息有100 亿条, 百分之十有10 亿条, 这个量级已经无法忍受了; 二是hdfs 挂掉的时候, flume 会有大量没有正常close 的文件, hdfs 恢复之后, 在lease hard limit 到期之前, 这些文件不会被recover lease, 这些文件会导致hive 无法正常读取, 需要手动执行命令修复: 
```
hdfs debug recoverLease -path pathA
```vvvvvvvvvvvvvvv不这这
当文件数很多的时候, 过程会很慢, 长达几个小时, 
 故提出了两个思路, 一是解决flume 的异常(只是丢数的问题) , 二是新开发一套系统替换flume . 

### 目标
经后续与关注flume 异常的同学以及领导讨论, 判断新开发一套系统的时间会比较短, 以及可预期, 于是开始设计新系统. 新系统需要满足以下目标: 
1. 外部系统kafka, hdfs 挂掉, 不会丢消息, 系统block 住, 直到故障恢复, 不需手动干预
2. 在kafka exactly once 的情况下, 不丢不重复消息. 
3. 满足现有活动的max qps

### 设计
一开始认为hdfs 上传是一个原子操作, 要不就上传失败无法被其他客户端看到, 要不就上传成功, 但是经过实验: 上传一个很大的文件上去, 中途 ctrl+c, 文件只上传了部分, 并经过一个小时(hard recover limit), 文件大小又增加了一些( close 触发了flush) , 证明hdfs 上传不是一个原子性质的, 故在文件名上加上了上传前的文件长度, 以判断文件的完整性. 
> Written with [StackEdit](https://stackedit.io/).
<!--stackedit_data:
eyJoaXN0b3J5IjpbMTU4MTE1MDE4NiwtMTg5MjQ2MzU2OCwtMT
k5NjQ2NDI0OSwxODc5OTMxNzEzLC04MDQ0NjQyODUsLTE4Njk5
NTQ5MTcsMTkyODE2MjQ0MV19
-->