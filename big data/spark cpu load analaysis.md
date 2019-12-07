因为用jvm-profiler 观察到spark executor 的cpuUsagePerProcess 比较低, 故找到这篇文章, 主要是关于这篇文章的总结: 
[https://db-blog.web.cern.ch/blog/luca-canali/2017-09-performance-analysis-cpu-intensive-workload-apache-spark](https://db-blog.web.cern.ch/blog/luca-canali/2017-09-performance-analysis-cpu-intensive-workload-apache-spark)

1. read a table frm parquet file is mostly cpu-intensive
2. can use [https://github.com/LucaCanali/sparkMeasure](https://github.com/LucaCanali/sparkMeasure), to measure more details metrics from executor internal, not from javaagent, which is used by uber jvm-profiler.
3. lab2 is faster than lab 1, because data is already in file system cache, as long as memory is enough.
4. lab3 , in os level , the cpu time is more than metrics reported by sparkMeasure, because there are other threads working , not only spark task thread.
5. parquet file using snappy compression is cost more cpu time to read and execute compared with non-compression parquet file, but cost less storage. 
6. when using less heap memory, means cost more time in gc, and more cpu time in gc
7. >Note: each core of the CPU model used in this test ([Intel Xeon CPU E5-2630 v4](https://ark.intel.com/products/92981/Intel-Xeon-Processor-E5-2630-v4-25M-Cache-2_20-GHz)) can [retire](https://software.intel.com/en-us/forums/intel-vtune-amplifier-xe/topic/311170) up to 4 instructions per cycle. You can expect to see IPC values greater than 1 (up to 4) for compute-intensive workloads, while IPC values are expected to drop to values lower than 1 for memory-bound processes that stall frequently trying to read from memory.

IPC (Instructions and cycles are two key metrics, often reported as a ratio instructions over cycles,) , IPC less than 1, means than cpu stalls frequently to read from memory. otherwise, it proves that 一直在计算.


> Written with [StackEdit](https://stackedit.io/).
<!--stackedit_data:
eyJoaXN0b3J5IjpbLTE2ODkxNjA2MTAsMjYzNTM3MTgzLDE4OD
M4Mzc5NzgsNTc4Mjc1MTMyLDgwMTA2ODQxXX0=
-->