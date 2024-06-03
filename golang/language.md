
# GMP model 
![image](https://github.com/Knight-Wu/articles/assets/20329409/1051e364-079a-4cda-8e9b-c2e384f0286f)

## 总体流程图
![image](https://github.com/Knight-Wu/articles/assets/20329409/42d24bd6-aaa6-45ad-8332-e7900b036879)

## 定义

### P 的定义
* 数量
The number of P’s is initialized at runtime startup and is by default equal to the number of logical cores of the cpu. It can be set at program startup with the environment variable GOMAXPROCS or the runtime.GOMAXPROCS() method, and the number of P’s is fixed for the duration of the program.

* IO 密集型

io 密集型系统中 P 的数量可以多于逻辑核心, 因为 M 会被 system call block, 此时 P 会被阻塞一会, 等待周期性 check (10MS) 去释放 P 和 G 到新的 M. 

In the IO-intensive scenario, the number of P can be adjusted appropriately, because M needs to be bound to P to run, and M will fall into the system call when executing G. At this time, P associated with M is in the waiting state, if the system call never returns, then the CPU resources are actually wasted during this time waiting for the system call, although there is a sysmon monitoring thread in runtime can seize G, here is to seize P associated with G, let P rebind a M to run G, but sysmon is periodic execution of seizure, after sysmon stable operation every 10ms to check whether to seize P, 

The open source database project https://github.com/dgraph-io/dgraph adjusts GOMAXPROCS to 128 to increase the IO processing power.

* p 状态

<img width="695" alt="image" src="https://user-images.githubusercontent.com/20329409/234467418-f25a923c-6b61-498d-9147-e7f317d48db7.png">

### M 的定义
M 每次创建就会创建一个操作系统线程, 所以 M 的数量是有上限的, 默认 10000, 创建太多 M 的内存开销很大, 每个 8 MB.
M is an object in runtime that represents a thread. Each M object created creates a thread bound to M. New threads are created by executing the clone() system call. runtime defines the maximum number of M to be 10000. The maximum number of M is defined in runtime as 10000, which can be adjusted by debug.SetMaxThreads(n).

* M 的创建

第一种是主线程 : M0, The Golang program creates the main thread when it starts, and the main thread is the first M i.e. M0.
另一种是当有 G 要创建或运行时, 并且有空闲的 P, 就会去找空闲的 M, 没有的话就创建.
When a new G is created or a G goes from _Gwaiting to _Grunning and there is a free P, startm() will be called, first getting an M from the global queue (sched.midle) and binding the free P to execute the G. If there is no free M, M will be created by newm().

如果 G (当做一个 function), 触发了系统调用, M 会释放 P; 如果结束了系统调用, M 会找空闲的 P,找不到就进入 sleep. 并且会记录 old P , 结束系统调用的时候倾向于找 old P, 因为之前的内存可以用, 减少拷贝. 
When the G associated with M enters the system call, M will actively unbind with the associated P. When the G associated with M executes the exitsyscall() function to exit the system call, M will find a free P to bind, if no free P is found then M will call stopm() to enter the sleep state.

* thread info


/proc/sys/kernel/threads-max: indicates the maximum number of threads supported by the system.
/proc/sys/kernel/pid_max: indicates the limit of the system global PID number value, every process or thread has an ID, the process or thread will fail to be created if the value of the ID exceeds this number.
/proc/sys/vm/max_map_count: indicates a limit on the number of VMAs (virtual memory areas) a process can have.

* M 中的 G0(g 零)

```
type m struct {
    g0      *g     // goroutine with scheduling stack
    ......
}
```
只用来记录线程的栈信息, 只用于调度的. 

Every time an M is started, the first goroutine created is g0. Each M will have its own g0. g0 is mainly used to record the stack information used by the worker thread, and is only used to be responsible for scheduling, which needs to be used when executing the scheduling code. When executing the user goroutine code, the stack of the user goroutine is used, and the stack switch occurs when scheduling.
### G 的定义

每次 go func 就会创建一个 G, 如果 G 里面工作很简单, 数量很多也没关系, 如果是需要网络连接和创建文件, 则太多的 G 会导致too many files open or Resource temporarily unavailable 

G 需要绑定 M 来跑, M 需要绑定 P 来跑. 
G is bound to M to run, and M needs to be bound to P to run, so theoretically the number of running G at the same time is equal to the number of P

### 调度过程中阻塞 
GMP模型的阻塞可能发生在下面几种情况：

1. I/O，select
2. block on syscall
3. channel
4. 等待锁
5. runtime.Gosched()
#### 用户态阻塞 
当goroutine因为channel操作或者network I/O而阻塞时（实际上golang已经用netpoller实现了goroutine网络I/O阻塞不会导致M被阻塞，仅阻塞G），对应的G会被放置到某个wait队列(如channel的waitq)，该G的状态由_Gruning变为_Gwaitting，而M会跳过该G尝试获取并执行下一个G，如果此时没有runnable的G供M运行，那么M将解绑P，并进入sleep状态；当阻塞的G被另一端的G2唤醒时（比如channel的可读/写通知），G被标记为runnable，尝试加入G2所在P的runnext，然后再是P的Local队列和Global队列。

#### 系统调用阻塞 
当G被阻塞在某个系统调用上时，此时G会阻塞在_Gsyscall状态，M也处于 block on syscall 状态，此时的M可被抢占调度：执行该G的M会与P解绑，而P则尝试与其它idle的M绑定，继续执行其它G。如果没有其它idle的M，但P的Local队列中仍然有G需要执行，则创建一个新的M；当系统调用完成后，G会重新尝试获取一个idle的P进入它的Local队列恢复执行，如果没有idle的P，G会被标记为runnable加入到Global队列。

## 为什么需要 gmp 模型
### 高效的并发模型

Goroutine 比系统线程更轻量，创建和销毁的开销更小。使用 GMP 模型，可以在单个进程内创建数十万甚至数百万个 Goroutine，而不会像线程那样引起过多的资源消耗和调度开销。
### 调度灵活性

GMP 模型允许在 P 上运行多个 Goroutine，并在不同的 M 线程之间切换。这样，Go 运行时可以更好地利用多核处理器的性能，最大限度地提高并发执行效率。
### 阻塞操作处理

当 Goroutine 执行阻塞操作（如 I/O 操作）时，Go 运行时会将其从当前的 M 线程中剥离出来，并分配另一个 Goroutine 到该 M 线程上继续执行。这种机制确保了阻塞操作不会阻塞整个线程，提高了程序的响应性和吞吐量。
### 自动扩展和收：

GMP 模型可以根据负载自动调整 M 线程的数量，以适应当前的工作量。它可以在需要时创建更多的 M 线程，或者在空闲时销毁不必要的 M 线程，从而高效地管理系统资源。


### g , m , p 的关系

![image](https://user-images.githubusercontent.com/20329409/234453413-4062b977-dc1e-4b5e-8952-4f4237818d57.png)

* g 的 global queue 
如果 p 的 groutine local queue 满了, 会把一部分 goroutine 放到 global queue

* p local queue

创建的 goroutine 会先放到 p local queue.

* P list

所有 p 都会按照 GOMAXPROCS 参数创建好, All P’s are created at program startup and stored in an array of up to GOMAXPROCS

* M

要跑的时候, 会先去拿 P, 再去 P 的 local queue 拿 G, 如果 local queue 空的, 会去 global 或者其他 p 的 local queue 去拿放到自己的 local queue .

### 为什么需要 P
Before Golang 1.1, there was no P component in the scheduler. The performance of the scheduler was still poor at this time. Dmitry Vyukov of the community summarized the problems in the current scheduler and designed to introduce the P component to solve the current problems ([Scalable Go Scheduler Design Doc](https://docs.google.com/document/d/1TTj4T2JO42uD5ID9e89oa0sLKhJYD0Y_kqxDv3I3XMw/edit#heading=h.mmq8lm48qfcw)), and introduced the P component in Go 1.1. The introduction of the P component not only solves several problems listed in the documentation, but also introduces some good mechanisms.

* global queue lock

之前没有p 的时候, 需要 global queue lock, 因为所有的 g 都在全局队列里, 引入了 p, 就可以大多数情况无锁访问 p 的 local G queue.</br>
为什么不直接把本地队列挂在M 上呢 ? 
一般来讲，M 的数量都会多于 P。像在 Go 中，M 的数量默认是 10000，P 的默认数量的 CPU 核数。另外由于 M 的属性，也就是如果存在系统阻塞调用，阻塞了 M，又不够用的情况下，M 会不断增加。
M 不断增加的话，如果本地队列挂载在 M 上，那就意味着本地队列也会随之增加。这显然是不合理的
* G 切换问题

切换G 带来的开销, 如果没有P, 一个goroutine 里面创建的g 会先放到全局g 的队列, 而不是直接被执行, 现在有了P 就直接放到P 的本地队列直接被执行

* M’s memory cache (M.mcache) problem

mcache 是一个 M object local cache 存放 G 的对象, 但是 M 有可能被 block by 系统调用, 所以 cache 就浪费了. 而且 cache 绑定到 M 的一个好处是G 如果再次被调度到 M 就可以重用, 但实际几率很少, 所以引入 P 之后 mcache 搬到了 P, 只有在运行的时候才会被占用, 不会造成空间浪费, 也避免了锁, 因为是没有其他线程去竞争的. 

* Frequent thread blocking and wake-up problems ?

In the original scheduler, the number of system threads is limited by runtime.GOMAXPROCS(). Only one system thread is opened by default. And since M performs operations such as system calls, when M blocks, it does not create a new M to perform other tasks, but waits for M to wake up, and M switches between blocking and waking frequently, which causes additional overhead. In the new scheduler, when M is in the system scheduling state, it will be disassociated from the bound P and will wake up the existing or create a new M to run other G bound to P.


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
