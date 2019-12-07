因为用jvm-profiler 观察到spark executor 的cpuUsagePerProcess 比较低, 故找到这篇文章, 主要是关于这篇文章的总结: 
[https://db-blog.web.cern.ch/blog/luca-canali/2017-09-performance-analysis-cpu-intensive-workload-apache-spark](https://db-blog.web.cern.ch/blog/luca-canali/2017-09-performance-analysis-cpu-intensive-workload-apache-spark)

1. read a table frm parquet file is mostly cpu-intensive
2. can use [https://github.com/LucaCanali/sparkMeasure](https://github.com/LucaCanali/sparkMeasure), to measure more details metrics from executor internal, not from javaagent, which is used by uber jvm-profiler.
3. lab2 is faster than lab 1, because data is already in file system cache, as long as memory is enough.
4. lab3 , in os level , the cpu time is more than metrics reported by sparkMeasure, because there are other threads working , not only spark task thread.
5. parquet file using snappy compression is cost more cpu time to read and execute compared with non-compression parquet file, but cost less storage. 
6. when using less heap memory, means cost more time in gc, and more extra cpu time in gc(not included in process cpu time maybe)
7. >Note: each core of the CPU model used in this test ([Intel Xeon CPU E5-2630 v4](https://ark.intel.com/products/92981/Intel-Xeon-Processor-E5-2630-v4-25M-Cache-2_20-GHz)) can [retire](https://software.intel.com/en-us/forums/intel-vtune-amplifier-xe/topic/311170) up to 4 instructions per cycle. You can expect to see IPC values greater than 1 (up to 4) for compute-intensive workloads, while IPC values are expected to drop to values lower than 1 for memory-bound processes that stall frequently trying to read from memory.

IPC (Instructions and cycles are two key metrics, often reported as a ratio instructions over cycles,) , IPC less than 1, means than cpu stalls frequently to read from memory. otherwise, it proves that
Instructions intensive( 一直在计算).
8.
>CPU utilization metrics can be misleading, drill down by measuring instruction per cycle (IPC) and memory throughput.

因为有可能IPC 低于1, means that cpu stalls because of waiting for data from memory. 

9. **hyper-threading** 技术 能够让物理cpu 扩展成逻辑cpu, 以对应更多的线程, 但是在cpu-intensive 的情况下, 总时间并没有缩短, 但cpu 工作时间是增加了, 但是若线程数大于逻辑cpu , 则cpu 工作时间也不能再增加. 

> Written with [StackEdit](https://stackedit.io/).
<!--stackedit_data:
eyJoaXN0b3J5IjpbNTM4OTIxNjcsLTQ3ODkxNjA1OCwtMjA2Mz
UzMDc3OCwtMTY4OTE2MDYxMCwyNjM1MzcxODMsMTg4MzgzNzk3
OCw1NzgyNzUxMzIsODAxMDY4NDFdfQ==
-->