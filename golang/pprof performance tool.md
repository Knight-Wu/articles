回归测试liveMonitor 的时候发现一会内存就飙升, 就被mesos kill 掉了, 用物理机部署也是一会就挂掉了, 怀疑是内存泄漏, 就找了pprof 来分析. 

主要参考的是这篇文章: [https://segmentfault.com/a/1190000019222661](https://segmentfault.com/a/1190000019222661)
讲的很详细. 

总结一下: 
* web 
> ip:port/debug/pprof/ 进入到总览页面

* cmd
> go tool pprof http://ip:port/debug/pprof/goroutine?debug=1 需要加 debug=1, 不然有可能c

> Written with [StackEdit](https://stackedit.io/).
<!--stackedit_data:
eyJoaXN0b3J5IjpbOTMyMTAxNDYxXX0=
-->