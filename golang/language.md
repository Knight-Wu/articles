
# GMP model 
## 为什么需要 gmp 模型
* 切换开销: process > kernel thread > user thread, goroutine 就相当于一类user thread , 切换更轻量
* 一个kernel thread 内存占用 8 MB, 而一个 goroutine 只需要 2KB.
* goroutine scheduler 都发生在用户空间, 阻塞了 goroutine 也不会阻塞内核线程,

## gmp 疑问
* 阻塞了 goroutine 也不会阻塞内核线程, java 会吗

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


# value or pointer
### https://jordanorelli.com/post/32665860244/how-to-use-interfaces-in-go 总结
func (t T)MyMethod(s string) {
    // ...
}

is a function of type  `func(T, string)`; method receivers are passed into the function by value just like any other parameter. by value means copy.

* Any changes to the receiver made inside of a method defined on a value type (e.g., `func (d Dog) Speak() { ... }`) will not be seen by the caller because the caller is scoping a completely separate `Dog` value.

* This works because a pointer type can access the methods of its associated value type, but not vice versa.( a pointer type may call the methods of its associated value type, but not vice versa)

* Since everything is passed by value, it should be obvious why a `*Cat` method is not usable by a `Cat` value; any one `Cat` value may have any number of `*Cat` pointers that point to it. If we try to call a `*Cat` method by using a `Cat` value, we never had a `*Cat` pointer to begin with. Conversely, if we have a method on the `Dog` type, and we have a `*Dog` pointer, we know exactly which `Dog` value to use when calling this method, because the `*Dog` pointer points to exactly one `Dog` value
( 所以 method receiver 最好是 pointer ? )


* Remember: everything is pass-by-value in Go. That means that inside of the `UnmarshalJSON` method, the pointer `t` is not the same pointer as the pointer in its calling context; it is a copy. If you were to assign `t` to another value directly, you would just be reassigning a function-local pointer; the change would not be seen by the caller.
(包括指针也是, 所以在函数内把指针重新指向, 对函数外也不生效)

* as a rule of thumb you can just remember that it’s typically better to take in an `interface{}` value as a parameter than it is to return an `interface{}` value 
(interface 最好作为参数而不是返回值) 

-   an  `interface{}`  value is not of any type; it is of  `interface{}`  type
-   interfaces are two words wide; schematically they look like  `(type, value)`

*  `var u User` will automatically [zero](https://href.li/?http://golang.org/ref/spec#The_zero_value "http://golang.org/ref/spec#The_zero_value") the `User` struct. Go is not like some other languages in that declaration and initialization occur separately,

# protobuf
### 好处
1. 用Base 128 Varints 编码数字

例如300 , int, 本来需要四个字节, 用这个编码之后只需要两个字节. 
让比较小的int 用更少的字节, 但是较大的int 就需要五个的字节, 从统计学意义上是能节省空间的, 

具体原理: 
用每一个字节的第一位(是否是1 ) 来表示是否还有下一个字节, 所以每个字节的有效位数只有7 位. 

如果用不到 1 个字节，那么最高有效位设为 0 ，如下面这个例子，1 用一个字节就可以表示，0000 0001. 

#  Calling a method on a nil struct pointer doesn't panic.

because [Michael Jones explained this well](https://groups.google.com/d/msg/golang-nuts/wcrZ3P1zeAk/WI88iQgFMvwJ)
摘录如下
> I think the question is really "but how would you inspect the object pointer to find and dispatch the function, as in C++ vtable." We might answer this from that angle:

  

> Method dispatch, as is used in some other languages with something like objectpointer->methodName() involves inspection and indirection via the indicated pointer (or implicitly taking a pointer to the object) to determine what function to invoke using the named method. Some of these languages use pointer dereferencing syntax to access the named function. In any such cases, calling a member function or method on a zero pointer understandably has no valid meaning.
> 
>   
> 
> In Go, however, the function to be called by the Expression.Name() syntax is entirely determined by the type of Expression and not by the particular run-time value of that expression, including nil. In this manner, the invocation of a method on a nil pointer of a specific type has a clear and logical meaning. Those familiar with vtable[] implementations will find this odd at first, yet, when thinking of methods this way, it is even simpler and makes sense. Think of:
> 
>   
> 
> func (p *Sometype) Somemethod (firstArg int) {}
> 
>   
> 
> as having the literal meaning:
> 
>   
> 
> func SometypeSomemethod(p *Sometype, firstArg int) {}
> 
>   
> 
> and in this view, the body of SometypeSomemethod() is certainly free to test it's (actual) first argument (p *Sometype) for a value of nil. Note though that the calling site invoking on a nil value must have a context of the expected type. An effort to invoke an unadorned nil.Somemethod would not work in Go because there is be no implicit "Sometype" for the typeless value nil to expand the Somemethod() call into "SometypeSomemethod()"

大致意思就是如果 object 是 nil , 那是如何找到这个 function 的呢? 在 golang 中与其他语言不同,  [Expression.method](http://expression.name/)() 是由 expression 的 type 决定的, 而不是 expression 的 value 决定的. 但是有些情况却不一样, 因为没有隐式的 type 转换. 

**所以当出现 nil 报错的时候需要看清是哪一行的报错, nil.field 是肯定报错的, 但是 nil.method 却不一定报错 !**

# concurrent


## channel
* close
 send on a close channel will cause panic, receive on close channel 接收所有已经发送的 val, 直到 val 接收完, 若 val 已经接收完, 则后续的 receive 会立马返回, 并且返回 val 的 zero value

* unbufferer channel
>A send operation on an unbuffered channel blocks the sending goroutine until another goroutine executes a corresponding receive on the same channel, at which point the value is transmitted and both goroutines may continue. Conversely, if the receive operation was attempted first, the receiving goroutine is blocked until another goroutine performs a send on the same channel.

send 和 receive 在一个 unbuffered channel, 会互相阻塞, 例如若没有 goroutine 在 receive , send 会阻塞, 反之亦然


## sync map
sync map 实际是以空间换时间, 适合读多写少的场景, 有read 和dirty 两个map 结构. 读取不需要加锁. 

* 读取过程

先读read, 读到直接返回, 读不到且dirty 有更多的数据, 则从dirty 读, 并且报告一次miss ,因为后续是一个slow path. 

```
// Load returns the value stored in the map for a key, or nil if no
// value is present.
// The ok result indicates whether value was found in the map.
func (m *Map) Load(key any) (value any, ok bool) {
	read, _ := m.read.Load().(readOnly)
	e, ok := read.m[key]
	if !ok && read.amended {
		m.mu.Lock()
		// Avoid reporting a spurious miss if m.dirty got promoted while we were
		// blocked on m.mu. (If further loads of the same key will not miss, it's
		// not worth copying the dirty map for this key.)
		read, _ = m.read.Load().(readOnly)
		e, ok = read.m[key]
		if !ok && read.amended {
			e, ok = m.dirty[key]
			// Regardless of whether the entry was present, record a miss: this key
			// will take the slow path until the dirty map is promoted to the read
			// map.
			m.missLocked()
		}
		m.mu.Unlock()
	}
	if !ok {
		return nil, false
	}
	return e.load()
}
```

* 修改过程

## sync.waitgroup

*  waitgroup add 必须要在 go func() 的外面, 保证 happen before waitgroup wait().
* break in select means break out in one loop, 如果外面还有 for loop, 并不能跳出, 需要一个标志位. 
排查bug 的两点总结: 
1. waitgroup 要传指针
2. add 要在主线程做, 否则可能主线程都wait 了, 还没add , 就会出现和预期不符合的结果. 
>Typically this means the calls to Add should execute before the statement  creating the goroutine or other event to be waited for.

铭记一点: 越离奇, 越不符合之前认知的错误, 越复杂, 越要静下心来, 细细分析, 用最理性的思维, 判断, 很有可能需要推翻之前的很多认知, 这样才有可能越快找到问题. 


# type conversion
用 built-in 的 string(),int() 为什么有时候不行, 有时候 string([]byte) 又可以
### int to string	
[https://stackoverflow.com/questions/39442167/convert-int32-to-string-in-golang](https://stackoverflow.com/questions/39442167/convert-int32-to-string-in-golang)

### int array to string
[https://stackoverflow.com/questions/37532255/one-liner-to-transform-int-into-string](https://stackoverflow.com/questions/37532255/one-liner-to-transform-int-into-string)

### fmt 用法
[https://blog.csdn.net/chenbaoke/article/details/39932845](https://blog.csdn.net/chenbaoke/article/details/39932845)

# time format
[https://ichon.me/post/998.html](https://ichon.me/post/998.html)


而Go语言的方式则略显奇葩，采用的是`"2006-01-02 15:04:05"`这样的layout string:

> time.Now().Format("2006-01-02 15:04:05")

# 当go 中引用了cgo, cgo 不能在mac 编译linux , 所以需要在本地的linux docker 中编译, 本地编译方便, 

1. 启动docker container
 docker run --rm -it -v ~/.ssh/:/root/.ssh/ -v /usr/local/go/src/:/go/src -v /Users/xxx/go:/go/code  golang:1.12.5 /bin/bash
 
本地命令: 
docker run --rm -it -v ~/.ssh/:/root/.ssh/ -v /usr/local/go/src/:/go/src -v /Users/tongwu/go:/go/code  golang:1.12.5 /bin/bash

2. 在container 里面指定GOPATH, 否则找不到依赖. 
默认的依赖在 $GOPATH/src  目录下, 根据报错提示更改 $GOPATH 到指定的目录. 即可

本地命令: export GOPATH=/go/code


# static link confluent-kafka-go
背景: 
golang 的 之前的 kafka 库都不太好用, sarama-kafka 已经不维护了, 而且自动提交 offset 的时候有 bug, 会导致重复消费; segmentio-kafka 不支持消费 topic 的 pattern, 并且设置了 groupid 之后就不能设置 auto.offset.reset 了, 真是够了. 然后就选用 [confluent-kafka]([https://github.com/confluentinc/confluent-kafka-go](https://github.com/confluentinc/confluent-kafka-go)) , 但是这个是用 go 封装了 c++的代码, 核心代码是 C++, 所以需要 build 的时候依赖 [librdkafka](https://github.com/edenhill/librdkafka),  这个 c++的库, 本来在物理机上很简单的, 但是需要通过 Jenkins 在 docker 上面 build, 问题可能就多了.  

依赖的方法很多, 当机器上面有这个包了之后, 接下来分为dynamic link 和 static link. 
依赖的方法: 
1. 直接用各个系统的包管理器, 例如Ubuntu apt-get install, 但是有些源下载下来的版本过低, 所以采用 confluent 的 repository. 但是! apt-get update 又出现 some indices update failed, 没有更详细的报错, 故作罢, 本来这个是最简单的方式, 而且 docker container 重启后会新建一个, 不会影响其他的环境. 
```
"wget -qO - https://packages.confluent.io/deb/5.2/archive.key |apt-key add -",
"echo \"deb [arch=amd64] https://packages.confluent.io/deb/5.2 stable main\" >> /etc/apt/sources.list",
"apt-get update",
"apt-get install -y librdkafka-dev=1.1.0~1confluent5.3.0-1",
```

2. 在一台 Ubuntu 上面编译好放到代码路径中, 但是有可能因为编译的系统环境和运行的系统环境不同而有风险, 公司的环境基本一致, 故觉得这个方法最好, 不需要依赖外部的网络, 速度又快. 但是编译好之后需要将包含rdkafka.pc 的文件夹添加到这个环境变量 PKG_CONFIG, 但是始终无法让这个环境变量生效, 莫非是嵌套 shell 的问题? 
3.  最后在这个无法找到 PKG_CONFIG 的报错中, 发现confluent-kafka-go 有个 issue : [Static build failing with missing rdkafka-static.pc #316](https://github.com/confluentinc/confluent-kafka-go/issues/316) , 介绍了一个分支是预先将 build 之后的包放到了confluent-kafka 这个的包路径中, 然后终于成功了, 可以看下他是如何预先编译的, 因为还没有正式合到 release , 还有些担心用到生产. 


### 问题
1. methods 和function 的区别
2. 短赋值的场景
3. 为什么要有指针
* Choosing a value or pointer receiver

There are two reasons to use a pointer receiver.

The first is so that the method can modify the value that its receiver points to.

The second is to avoid copying the value on each method call. This can be more efficient if the receiver is a large struct, for example.
4. 为什么要用make
5. interface 在 go里面怎么理解
6.   evaluation of `f`, `x`, `y`, and `z` happens in the current goroutine and the execution of `f` happens in the new goroutine ？ two gorountine ？
7. 如何理解channel
8. sync.Mutex 是控制程序执行的顺序还是内存可见性
9. go 编译和运行？ 如何引用其他文件的代码
10. go get 如何在 ide get 的时候自动指定版本 
11. go build 为什么需要 sudo , 然后再服务器执行就 exec format error 了, go install 是可以的
12. ssh 端口转发
