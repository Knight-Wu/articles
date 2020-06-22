### 前言
距离sinker 上线已经快一个月了, 相当可靠没有一个error( 我把error 日志接入了企业微信, 前几天是有些提心吊胆的) , 于是终于不犯懒去写一篇回顾, 趁着还记得, 这是非常有必要的
### 背景
flume 接tracking 的数据写到hdfs 的hive 外表, 会丢消息, 集中在凌晨跑批 hdfs 繁忙的时候, 没有找到相关异常, 也没有在flume 找到issue, 一直没解决, 严重的时候有百分之十以上的丢消息, 过滤完之后的tracking kafka 平均 12w qps, 峰值 25w qps, 活动峰值 35w qps, 大约每天有百分之十的丢消息, 以12w qps 算, 一天消息有


> Written with [StackEdit](https://stackedit.io/).
<!--stackedit_data:
eyJoaXN0b3J5IjpbLTE0MzQ3MDIwMzEsMTkyODE2MjQ0MV19
-->