因为用jvm-profiler 观察到spark executor 的cpuUsagePerProcess 比较低, 故找到这篇文章, 主要是关于这篇文章的总结: 
[https://db-blog.web.cern.ch/blog/luca-canali/2017-09-performance-analysis-cpu-intensive-workload-apache-spark](https://db-blog.web.cern.ch/blog/luca-canali/2017-09-performance-analysis-cpu-intensive-workload-apache-spark)

1. read a table frm parquet file is mostly cpu-intensive
2. can use [https://github.com/LucaCanali/sparkMeasure](https://github.com/LucaCanali/sparkMeasure), to measure more details metrics from executor internal, not from javaagent, which is used by uber jvm-profiler.
3. lab2 is faster than lab 1, because data is already in file


> Written with [StackEdit](https://stackedit.io/).
<!--stackedit_data:
eyJoaXN0b3J5IjpbOTMyMTgxNDI5XX0=
-->