### debug binary file in local machine
1. download a small utility program named gops, available at [https://github.com/google/gops](https://github.com/google/gops). This program helps the IDE find Go process running on your machine. Then invoke the _Attach to Process…_ feature again.
2. If you are running with Go 1.10 or newer, you need to add `-gcflags="all=-N -l"` to the `go build` command. 然后运行. 
3. 然后选择goland Run 菜单下的 attach , 选择自己的进程, 就可以断点调试了

### debug binary file in remote machine
1. If you are running with Go 1.10 or newer, you need to add `-gcflags="all=-N -l"` to the `go build` command.  然后上传服务器.
2. 在远程服务器下载 delve. 
```
go get -u github.com/go-delve/delve/cmd/dlv
```
[https://github.com/go-delve/delve/blob/master/Documentation/installation/linux/install.md](https://github.com/go-delve/delve/blob/master/Documentation/installation/linux/install.md)

3. 然后运行应用. 
有两种方式: 
a. let the debugger run the process for you: if you choose this option, you need to run `dlv --listen=:2345 --headless=true --api-version=2 exec ./application`
b. attaching to the process: you need to run  `dlv --listen=:2345 --headless=true --api-version=2 attach <pid>` where  _<pid>_  is the process id of your application.
都需要暴露服务器上的应用的端口给 local machine
4. _Run | Edit Configurations… | + | Go Remote_ and configuring the host and port your remote debugger is listening on.


[jetbrains 的完整文档](https://blog.jetbrains.com/go/2019/02/06/debugging-with-goland-getting-started/#debugging-a-running-application-on-a-remote-machine)

### float 运算
若某个浮点数, 没有经过强制保留两位小数, 则有可能再经过运算后, 出现尾数, 
例如
> float a = 0.02, float b= 1-0.02= 0.9800001 , 

* 解决办法
1. 计算结果强制保留两位小数: 
>c, _ := strconv.ParseFloat(fmt.Sprintf("%.2f", 1-a), 32), 则c = 0.98

2. 浮点数计算框架: 
[https://github.com/shopspring/decimal](https://github.com/shopspring/decimal)

3. 将浮点数转化为整数, 乘 10 的 n 次方
> Written with [StackEdit](https://stackedit.io/).
<!--stackedit_data:
eyJoaXN0b3J5IjpbMTIwMjgxMTczMSwtMTQwOTgwMTk2MiwtMT
kwOTg1MzkxXX0=
-->