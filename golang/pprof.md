回归测试liveMonitor 的时候发现一会内存就飙升, 就被mesos kill 掉了, 用物理机部署也是一会就挂掉了, 怀疑是内存泄漏, 就找了pprof 来分析. 后面发现是goroutine 泄漏. orz

主要参考的是这篇文章: [https://segmentfault.com/a/1190000019222661](https://segmentfault.com/a/1190000019222661)
讲的很详细. 

总结一下: 
* web UI
> ip:port/debug/pprof/ 进入到总览页面

* cmd 
> go tool pprof http://ip:port/debug/pprof/goroutine?debug=1 需要加 debug=1, 不然有可能出现格式错误的问题. 

### 查看stacktrace duration
```
1. go tool pprof http://localhost:8825/debug/pprof/profile -seconds 60
若 parsing profile: unrecognized profile format, 则可能因为 golang 版本问题, 在 mac 本地执行试试
3. 然后会保存 pb.gz 文件到本地
4. 如果linux 能直接打开最好, 如果打不开则scp 下载到本地
5. 可以通过cmd 直接进入web UI
Type: cpu
Time: May 17, 2020 at 9:18pm (+08)
Duration: 30.12s, Total samples = 26.96s (89.50%)
Entering interactive mode (type "help" for commands, "o" for options)
(pprof) 

6. 但是需要先安装 Graphviz, linux : apt-get install
mac: brew install
7. 第四步也可以 go tool pprof -http=:8810 pprof.xxx.samples.cpu.006.pb.gz
8. 通过localhost:8810 webUI, 可以看到火焰图

```

> Written with [StackEdit](https://stackedit.io/).
<!--stackedit_data:
eyJoaXN0b3J5IjpbLTc3NjkzMjY4LDEzMDM1Mzc4NDgsODE0Nz
IzMTQwLC05Njk5MzY0MDZdfQ==
-->