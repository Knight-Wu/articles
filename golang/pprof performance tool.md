回归测试liveMonitor 的时候发现一会内存就飙升, 就被mesos kill 掉了, 用物理机部署也是一会就挂掉了, 怀疑是内存泄漏, 就找了pprof 来分析. 




> Written with [StackEdit](https://stackedit.io/).
<!--stackedit_data:
eyJoaXN0b3J5IjpbMTE4NjkyNDM3MF19
-->