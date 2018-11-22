很奇怪cdh 测试环境出现的这个问题, 死活查不到, spark-2.2 , hive-1.1, 使用spark-shell, spark.sql("show databases"), spark.sql("show tables") 均查不到

### 测试方法
* 看spark 的配置 是否依赖了hive 服务, 并且把spark shell的日志级别调到info, 跟踪

![enter image description here](https://drive.google.com/uc?id=1EMvPfK4EHC1TQDEScJWgOzV6vRM__vkq)

* 


> Written with [StackEdit](https://stackedit.io/).
<!--stackedit_data:
eyJoaXN0b3J5IjpbLTU5NTI5NDY0OV19
-->