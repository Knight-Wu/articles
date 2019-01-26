
  
##  运行时数据区域
### 程序计数器(programme counter register)
若执行的是非native方法, 则保存下条指令的地址; 若是native方法, 则为空;每个线程独有, 互不影响; 因为多个线程在一个 cpu 是交替执行的, 需要用PCR 来保存线程切换后线程执行的状态, 线程越多占得内存空间也越大

### 虚拟机栈(virtual machine stacks), 本地方法栈(native method stack)
每个线程独有, 每个方法创建的时候都会创建一个栈帧(stack frame),用于存储方法的局部变量, 操作数栈等.
 , 虚拟机栈和本地方法栈的不同是,前者执行java方法, 后者执行native方法
* StackOverFlowError
线程请求的栈深度大于虚拟机允许的栈深度, 抛出该异常
* OutOfMemoryError
如果虚拟机栈可以动态扩展,但是扩展时无法申请到固定的内存,会抛出OutOfMemoryError


### java 堆(heap)
存放对象的实例和数组, 所有线程所共有; 如果堆中没有内存完成实例的分配, 并且堆也无法再扩展时,抛出 OutOfMemoryError, 从java 7开始, constant pool 从永久代移到了堆, 包括string pool ,

String str = new String("hello");
程序中的字面量（literal）如直接书写的100、"hello"和常量都是放在常量池中，常量池是方法区的一部分，

### 方法区(Method Area)
线程间共享, 存储每个类的结构; 
java 8之后, 没有了**PermGen space**, 用方法区代替, 且方法区不属于heap size的一部分, 属于进程的内存, 光监控java heap size 已经不够了, 需要用top 监控整个 jvm 进程的内存, 因为还包括方法区和native memory.

### 本地内存(native memory, C heap)
1. 管理java heap的状态数据（用于GC）;
2. JNI调用，也就是Native Stack;
3. JIT（即使编译器）编译时使用Native Memory，并且JIT的输入（Java字节码）和输出（可执行代码）也都是保存在Native Memory；
4. NIO direct buffer。对于IBM JVM和Hotspot，都可以通过-XX:MaxDirectMemorySize来设置nio直接缓冲区的最大值。默认是64M。超过这个时，会按照32M自动增大。
    DirectBuffer访问更快，避免了数据从heap memory 拷贝到本地堆。DirectBuffer byte array 实际是保存在native heap中，但是操作该byte array的对象保存在java heap中。 
GC时不会直接回收native memory, 通过释放heap memory中的对象来释放native memory, 但是通常java heap没达到gc 的条件. 
5. 对于IBM的JVM某些版本实现，类加载器和类信息都是保存在Native Memory中的。

* tips
    * 分配内存的时候优先给heap memory 分配, 再到native memory


### 虚拟机对象
* 创建
每个线程分配一块独立的内存,本地线程分配缓冲(Thread local allocation buffer),来控制给每个对象分配内存时是线程安全的

* 对象的内存布局
对象头, 实例数据, 对齐填充
* 对象的访问
 sun hotspot通过直接指针的方式, reference存储了对象的地址,存储在栈区(应该指的是虚拟机栈),直接访问到堆中的对象的数据,对象的数据中包含类的信息.

### 内存泄漏原因
* StackOverFlowError
>有可能是栈的深度超过最大深度, 也有可能是栈区的内存大小不足, 实质应该是一样的, 可以通过增加栈区的内存大小(-Xss).
操作系统分配个每个进程的内存是固定的 , windows下32位每个进程最大内存是2GB, 减去堆(-Xmx 最大堆容量) ,方法区(-MaxPermSize  最大方法区容量),程序计数器所占内存太小忽略不计, 剩下的就是栈区(包括虚拟机栈和本地方法栈)

* OutOfMemoryError
> 可以参考自 
> 1. [Java内存溢出(OOM)异常排查指南](https://blog.csdn.net/m0_38110132/article/details/79848426)
> 2. [深入解析OutOfMemoryError](http://www.importnew.com/22173.html) 
1. Exception in thread "main" java.lang.OutOfMemoryError: Java heap space
> 堆内存溢出, 超过设置的最大堆的大小,则报该异常, 直接原因就是因为没有内存分配给新的对象了, 间接原因可以说是gc时应该清除的对象没有被清除, 某些地方仍然保留有该对象的引用.

1.  因为jvm会尽量保持堆是初始化的大小, 所以设置最大堆(-Xmx) 和堆的初始化值(-Xms)一样, 以减小gc 频率
2. heap dumps
> 反映对象的数量和类文件所占用的字节数, -XX:+HeapDumpOnOutOfMemoryError, 在快oom的时候打印日志.
1. 堆转储分析：live objects
>使用jmap并且加上-histo参数可以为你产生一个直方图，它显示了从程序运行到现在所有对象的数量和内存消耗，并且包含了已经被回收的对象和内存。如果使用-histo:live参数会显示当前还在堆中得对象数量及其内存消耗，不论这些对象是否要被垃圾搜集器进行回收。
2. 堆转储分析：跟踪引用链 
>浏览堆转储引用链具有两个步骤：首先需要使用-dump参数来使用jmap，然后需要用jhat来使用转储文件。查看对象的引用链
3. 堆转储分析：内存分配情况
> 可以找到对象使用的情况, 以及这些对象的引用被哪里的代码使用的, 但是有时候这种方式还是不够的, 例如string对象会很多, 

* Exception in thread "main" java.lang.OutOfMemoryError: PermGen space
> 存储类的内存区域出现内存泄漏, 可能出现在应用被频繁部署的时候, 某些类占的内存区域没有被释放, 解决永久代错误的第一个方法就是增大永久大的空间，你可以使用-XX:MaxPermSize命令行参数。默认是64M，但是web应用程序或者IDE一般都需要256M。

> 解决永久代的问题通常都是比较痛苦的。一般可以先考虑加上-XX:+TraceClassLoading和-XX:+TraceClassUnloading命令行选项以便找出那些被加载了但是没有被卸载的类。如果你加上了-XX:+TraceClassResolution命令行选项，你还可以看到哪些类访问了其他类，但是没有被正常卸载。

* Exception in thread "main" java.lang.OutOfMemoryError: Direct buffer memory
> native memory不够用了, 见"深入理解JVM" 83页: 明显的特征, heap dump文件中不会看见明显的异常, dump文件很小, 如果程序中使用了大量的NIO, 需要考虑是否是这个问题导致.

* Exception in thread "main" java.lang.OutOfMemoryError: unable to create new native thread
> 线程太多了, 实际应该不需要, 因为可能大多数线程在等待, 而且需要频繁切换, 可以换成aio 或者nio


#### java heap分析工具
```
jmap -dump:format=b,file-fileName pid
// 然后用 Eclipse memory analyzer 打开
```

*问题* 
1. linux内存分配策略, 每个进程如何分配内存, 和windows有何不同

## 三. 垃圾回收

### 可达性分析
>若不存在从对象到GC root的引用链, 则在下次gc时, 该对象会被回收, 下次gc可能是minor gc(日志中称作 gc, 发生在新生代 ), major gc(日志中称作 full gc, 发生在老年代)


* 回收方法区
主要针对类的回收,满足以下三个条件: 
1. 类的所有实例已被回收
2. 加载该类的classloader已被回收
3. 该类的java.lang.class对象没有任何地方被引用, 不存在反射访问该类的方法.


### 引用
* 强引用
只要强引用存在, 则永远不会被回收
```
 Obj a = new Obj();
```

* 软引用
SoftReference来实现, 除非内存准备溢出了, 不然不会被回收.

* 弱引用
WeakReference的对象, 若只被弱引用引用, 不被其他任何强引用引用时, 如果GC运行, 那么该对象就会被回收.例子 ThreadLocal中的Entry 持有对ThreadLocal对象的弱引用

* 虚引用
> DirectByteBuffer

### 垃圾收集策略
* 标记-清除算法

需要回收的对象进行一次标记,标记完成后统一回收, 不需要对对象进行移动
* 缺点
  * 标记和回收的效率都不高
  * 回收后产生大量内存碎片,以至于分配较大对象时,无法得到连续的足够内存而导致再次进行gc

* 标记-复制算法

将内存分配为一块eden和两块survivor, 比例是8:1, 每次使用新生代内存的90%, gc时将存活对象复制到空闲的survivor, 剩余对象一次性清理
缺点:  不适用对象存活率较高的情况,复制的对象太多, 例如老年代
* 标记整理算法
将不可用对象进行一次标记, 并清除, 然后将可用的对象向一端移动,然后清理掉边界以外的内存,有效避免了内存碎片(针对标记清除算法)和需要有一部分空间来作为留存空间(针对标记-复制算法).

* 分代收集算法
对新生代和老年代采用不同的垃圾收集算法, 例如HotSpot新生代采用标记-复制, 老年代采用标记整理

### java heap 分代(基于jdk1.8)
* 新生代(PSYoungGen)
 eden space , survivor(from space , to space) ,可设置比例,采用标记复制算法
* 老年代(ParOldGen)
采用标记清除算法或标记整理算法
* Metaspace(方法区)
存储类的信息


### 垃圾收集器
* 重要概念

并发和并行: 并发指的是gc 线程和用户线程在一个cpu 内进行交替执行; 而并行指的是这两个线程在不同的cpu 上同时执行, 不需要切换.

各个收集器的配套使用如下图: 
![enter image description here](https://drive.google.com/uc?id=1UkKi-2ipWHbbol68eC3-5wQtM2-cNAPY)

http://www.fasterj.com/articles/oraclecollectors1.shtml
https://stackoverflow.com/questions/33206313/default-garbage-collector-for-java-8

* Serial
 单线程收集器, 不只是说只会使用一个cpu或者一个线程去收集, 而是指GC时会暂停其他工作线程,但省去了线程切换的开销, 在client 模式下, heap是几百兆的情况, gc效率很高, 造成的停顿仅仅是几百毫秒, 是client端默认的GC收集器

* ParNew
是Serial 的多线程版本, 只是并发, 并不能做到并行执行.  复制算法, 只能与CMS联合使用, 

* Parallel Scavenage
新生代收集器, 复制算法, 与ParNew 的最大区别可以控制吞吐量, gc 最大暂停时间等. 
java1.8的默认垃圾收集器是 parallel collector
(https://docs.oracle.com/javase/8/docs/technotes/guides/vm/gctuning/parallel.html) , 即新生代是 Parallel Scavenage , 老年代是Parallel Old , 当使用 **-XX:+UseParallelGC**或 **-XX:+UseParallelOldGC** 均意味着两个配合使用. 
有以下特点: 
可以控制吞吐量(gc 时间和运行时间的比值), -XX:GCTimeRatio
最大GC 暂停时间,  -XX:MaxGCPauseMills
动态调整heap size的大小, 如果某个代的gc时间超过最大GC 停顿时间, 则会按比例减少这个代的大小, 如果某个代的吞吐量不满足, 则会增大某个代的大小.

这个收集器关于增减heap size 的配置是特殊的, 需要注意: 
> The following discussion regarding growing and shrinking of the heap and default heap sizes does not apply to the parallel collector. (See the section [Parallel Collector Ergonomics](https://docs.oracle.com/javase/8/docs/technotes/guides/vm/gctuning/parallel.html#parallel_collector_ergonomics) in [Sizing the Generations](https://docs.oracle.com/javase/8/docs/technotes/guides/vm/gctuning/sizing.html#sizing_generations) for details on heap resizing and default heap sizes with the parallel collector.) However, the parameters that control the total size of the heap and the sizes of the generations do apply to the parallel collector.

* Serial Old
是Serial 的老年代版本, 可以与Parallel Scavenage配合使用, 作为CMS的备案, 在发生concurrent mode failure使用

* Parallel Old
标记整理算法, 是Parallel Scavenage的老年代版本, 可与Parallel Scavenage 配合使用, 

* CMS(Concurrent Mark Sweep)
> designed for applications that prefer shorter garbage collection pauses and that can afford to share processor resources with the garbage collector while the application is running
基于标记-清除算法, 所以会产生内存碎片.

oracle 文章的截图: 
![enter image description here](https://drive.google.com/uc?id=12ASb1yk3McGByLQ0jEeNrAEcIvgfOCbK)
过程: 
1. 第一次STW 暂停,  initial mark , 标记老年代中被GC root (可能来自新生代和老年代)**直接可达**的对象, 通常耗时很短,  比minor gc 还要快. 
2.  Concurrent Marking, 这个阶段不会暂停用户线程, 并行的标记老年代的所有存活的对象. 
3.  Concurrent Preclean（并发预清理）此阶段同样是与应用线程并行执行的，不需要停止应用线程。因为前一阶段是与程序并发进行的，可能有一些引用已经改变。如果在并发标记过程中发生了引用关系变化，JVM 会通过 Card 将发生了改变的区域标记为「脏」区，这就是所谓的卡片标记（Card Marking）。本阶段也会执行一些必要的细节处理，并为 Final Remark 阶段做一些准备工作
4. Concurrent Abortable Preclean(并发可取消的预清理）,不会暂停用户线程, 3,4 阶段的作用在CMS 调优中再详细介绍.

5.  Remark 最终标记, 本阶段的目标是完成老年代中剩余存活对象的标记,  因为之前的concurrent mask 是和用户线程并发执行的, 可能中间会产生浮动垃圾, 所以需要进行最终标记, 会STW, 通过耗时较长, 而且还可能还随着年轻代存活对象的数量的数量增多而时间变长.


> The remark pause is often comparable in length to a minor collection. The remark pause is affected by certain application characteristics (for example, a high rate of object modification can increase this pause) and the time since the last minor collection (for example, more objects nation may increase this pause)
> [https://docs.oracle.com/javase/8/docs/technotes/guides/vm/gctuning/cms.html](https://docs.oracle.com/javase/8/docs/technotes/guides/vm/gctuning/cms.html)

7.  Concurrent Sweep（并发清除）此阶段与应用程序并发执行，不需要 STW。目的是删除未使用的对象，并收回他们占用的空间
8. Concurrent Reset（并发重置）此阶段与应用程序并发执行，重置 CMS 算法相关的内部数据，为下一次 GC 循环做准备

* 浮动垃圾(Floating Garbage)
由于应用线程和gc 线程并行执行, gc 线程标记的可达的对象在标记结束后又不可达了, 这部分剩余的对象叫做浮动垃圾, 这部分对象会在下次gc 被回收.


> concurrent mode failure

    因为gc时用户线程继续产生新对象, 如果CMS 此时不能完成清除工作, 导致老年代空间满了, 会触发这个失败, 会停止所有的用户线程去做gc, 停顿会很长.
 所以需要预留至少一部分空间用作gc时新对象的产生, 所以可以设置 -XX: CMSInitiatingOccupancyFraction 作为开始老年代收集的百分比, 超过这个占用即开始收集, 减小concurrent mode failure 的概率
       
> 采用标记清除算法, 内存碎片很多

使用 -XX:UseCMSCompactionAtFullCollection(默认开启, 开启内存碎片的整理工作), 但是会导致停顿时间变长,  也可采用 -XX:CMSFullGCsBeforeCompaction(表示执行了多少次不压缩的full GC后, 来一次压缩的full GC ,默认是0, 表示每次都压缩).
    
* Garbage First(G1)
  
https://www.oracle.com/webfolder/technetwork/tutorials/obe/java/G1GettingStarted/index.html#t4
  
https://docs.oracle.com/javase/8/docs/technotes/guides/vm/gctuning/g1_gc_tuning.html

https://tech.meituan.com/g1.html
 > 比起CMS 的优点

可以参考[http://openinx.github.io/ppt/hbaseconasia2017_paper_18.pdf](http://openinx.github.io/ppt/hbaseconasia2017_paper_18.pdf)
  G1是一个有整理内存过程的垃圾收集器，不会产生很多内存碎片。
  G1的Stop The World(STW)更可控，G1在停顿时间上添加了预测机制，用户可以指定期望停顿时间, 通过控制收集哪些region和region 的多少来控制停顿时间. 
  
停顿时间: (和吞吐量是此消彼长的)
> When you evaluatize orf tune any garbage collection, there is always a latency versus throughput trade-off. The G1 GC is an incremental garbage collector with uniform pauses, but also more overhead on the application threads. The throughput goal for the G1 GC is 90 percent application time and 10 percent garbage collection time. Compare this to the Java HotSpot VM parallel collector. The throughput goal of the parallel collector is 99 percent application time and 1 percent garbage collection time. Therefore, when you evaluate the G1 GC for throughput, relax your pause time target. Setting too aggressive a goal indicates that you are willing to bear an increase in garbage collection overhead, which has a direct effect on throughput. When you evaluate the G1 GC for latency, you set your desired (soft) real-time goal, and the G1 GC will try to meet it. As a side effect, throughput may suffer. See the section [Pause Time Goal](https://docs.oracle.com/javase/8/docs/technotes/guides/vm/gctuning/g1_gc.html#pause_time_goal) in [Garbage-First Garbage Collector](https://docs.oracle.com/javase/8/docs/technotes/guides/vm/gctuning/g1_gc.html#garbage_first_garbage_collection) for additional informationhe young generat

> Applications running today with either the CMS or the with parallel compaction would benefit from switching to G1 if the application has one or more of the following traits: 
a. More than 50% of the Java heap is occupied with live data.
 b. The rate of object allocation rate or promotion varies significantly.
 c. The application is experiencing undesired long garbage collection or compaction pauses (longer than 0.5 to 1 second).
 
> gc 的类型: young gc和mixed gc
具体流程可以参考: https://www.oracle.com/webfolder/technetwork/tutorials/obe/java/G1GettingStarted/index.html

Young GC：选定年轻代里的一些 Region。通过控制年轻代的region**总数**，即年轻代内存大小，来控制young GC的时间开销。
大致过程: 
当Eden区域无法申请新的对象时（满了），就会进行Young GC, Young GC将Eden和Survivor区域的Region(**称为Collection Set, CSet**)中的活对象Copy到一些新Region中(即新的Survivor)，当对象的GC年龄达到阈值后会Copy到Old Region中。由于采取的是Copying算法, 并且copy 途中会压缩，所以就避免了内存碎片的问题，

Mixed GC 选定所有年轻代里的Region，外加根据concurrent marking统计得出lowest "liveness" 老年代Region( 回收最快), 在用户指定的开销目标范围内.

Mixed GC 的触发条件: 
-   G1HeapWastePercent：在global concurrent marking结束之后，我们可以知道old gen regions中有多少空间要被回收，在每次YGC之后和再次发生Mixed GC之前，会检查垃圾占比是否达到此参数，只有达到了，下次才会发生Mixed GC。
-   G1MixedGCLiveThresholdPercent：old generation region中的存活对象的占比，只有在此参数之下，才会被选入CSet。
-   G1MixedGCCountTarget：一次global concurrent marking之后，最多执行Mixed GC的次数。
-   G1OldCSetRegionThresholdPercent：一次Mixed GC中能被选入CSet的最多old generation region数量。

Full GC:
和CMS一样，G1的一些收集过程是和应用程序并发执行的，所以可能还没有回收完成，是由于申请内存的速度比回收速度快，新的对象就占满了所有空间，在CMS中叫做Concurrent Mode Failure, 在G1中称为Evacuation Failure，
>  https://www.oracle.com/webfolder/technetwork/tutorials/obe/java/G1GettingStarted/index.html **How to avoid Evacuation Failure**


> 什么情况触发 concurrent marking 

A concurrent marking phase is started when the occupancy of the entire Java heap reaches the value of the parameter  `InitiatingHeapOccupancyPercent`. Set the value of this parameter with the command-line option  `-XX:InitiatingHeapOccupancyPercent=``<NN>`. The default value of  `InitiatingHeapOccupancyPercent`  is 45.

> global concurrent marking 大致步骤

-   初始标记（initial mark，STW）。它标记了从GC Root开始直接可达的对象。
-   并发标记（Concurrent Marking）。这个阶段从GC Root开始对heap中的对象标记，标记线程与应用程序线程并行执行，并且收集各个Region的存活对象信息。
-   最终标记（Remark，STW）。标记那些在并发标记阶段发生变化的对象，将被回收。
-   清除垃圾（Cleanup, STW）。清除空Region（没有存活对象的），加入到free list。


> region
![enter image description here](https://drive.google.com/uc?id=1Ts2G1JO3TdWeT-m7YOsN3o76y8kgf3PC)
新生代和老年代由region 构成, 存储地址不是连续的, H 表示这些Region存储的是巨大对象（humongous object，H-obj），即大小大于等于region一半的对象, 一个H 区域的剩余空间就不能分配其他对象了, H直接属于old gen. 为了减少连续H-objs分配对GC的影响，需要把大对象变为普通的对象，建议增大Region size, 否则会根据heap size 去分配region size. 




### GC回收过程
* 新生代gc过程
对象先在eden分配, 然后eden满了, 启动一次minor, 存活对象分配到from区, eden清空, 然后eden再次满了, 将eden和from中仍然存活的对象copy到to区, 然后eden和from清空, 之后to和from相对于就对换了, 随后的minor 会再次将eden和from区存活对象复制到to区, 若满足晋升条件则直接晋升到老年代, 或者 to区域满了就会直接晋升老年代. 

> 新生代标记的过程中, 如何避免像老年代的全堆扫描, 知道哪些新生代对象被老年代引用呢

经过统计信息显示，老年代持有新生代对象引用的情况不足1%，根据这一特性JVM引入了卡表（card table）, 卡表的具体策略是将老年代的空间分成大小为512B的若干张卡（card）, 当老年代引用新生代时会更新卡表,  新生代扫描卡表就避免了全堆扫描. 



* 新生代晋升条件

     * **动态年龄计算**：Hotspot遍历所有对象时，按照年龄从小到大对其所占用的大小进行累积，当累积的某个年龄大小超过了survivor区的一半时，取这个年龄和MaxTenuringThreshold中更小的一个值，作为新的晋升年龄阈值。

  * 某些对象大小超过指定的阈值, 则这种大对象直接分配到老年代


* major gc 
指的老年代进行的gc, 老年代对象比例超过某个阈值, 通常有参数设置, 不可能是老年代百分百了才触发, 例如 CMS gc 设置这个值 :  -XX:CMSInitiatingOccupancyFraction=75%,  意味着老年代超过75 才触发major gc. 


* full GC
https://www.zhihu.com/question/41922036/answer/93079526  
Full GC时，就不在分 “young gen使用young gen自己的收集器(一般是copy算法)；old gen使用old gen的收集器(一般是mark-sweep-compact算法)”，而是，整个heap以及perm gen，所有内存，全部的统一使用 old gen的收集器(一般是mark-sweep-compact算法) 一站式搞定




* GC root
 1. local variable
 2. active java thread
 3. static variable
 4. JNI reference


#### GC调优
**线上GC 调优经验, 看 dap hbase 的gc 日志**
* 定理
Maximum Pause Time Goal,  Throughput Goal,  Footprint Goal三者只能取其二, 特别是1和2是互相矛盾的, heap size 越大, 频率会降低, 但是当gc 的时候, 存活的对象很多的话, gc 的时间就很长

> 目标

gc 的频率和时间都降低

* gc pause 时间取决于什么
取决于存活的对象的数量, 而不是heap size, 所以只有进行gc 标记时大多数的对象都死了, 那么gc 暂停时间会比较短.

> 新生代

* 降低young gc 频率
直接原因: eden 区满了, 
直接方法: 增大eden区的大小, 间接方法: 增大young 区, 增大heap, 
* 降低 young gc 的时间
增大eden区的大小, 假设young gc 的间隔越久, 则存活的对象越少, 但是如果过大, 也会增加young gc 的时间, 因为需要复制的对象大小也变大了,  当然取决于新生代对象的存活时间的分布. 举例如下: 
扩容前：新生代容量为R ，假设对象A的存活时间为750ms，Minor GC间隔500ms，那么本次Minor GC时间= T1（扫描新生代R）+T2（复制对象A到S）。
    
扩容后：新生代容量为2R ，对象A的生命周期为750ms，那么Minor GC间隔增加为1000ms，此时Minor GC对象A已不再存活，不需要把它复制到Survivor区，那么本次GC时间 = 2 × T1（扫描新生代R），没有T2复制时间

> 老年代

* 降低old gc 频率
直接方法: 
1. 提高老年代的大小
2. 提高晋升老年代的门槛
>  可以增大新生代的大小, minor gc 频率越低, 晋升老年代的门槛会越高, 可能可以降低full GC 的频率. 虽然老年带就会越小(永久代 + 年轻代等于 heap size), 进而带来major gc 的频率升高, 反之如果新生代大小调小, 可以适当提高晋升的年龄大小,  还有一种情况是 具体的调优值取决于对象的生命周期的组成, 可以在同一个应用的几台服务器设置不同的newRadio 观察gc 的日志, 参数:`-XX:NewRatio=3`  means that the ratio between the young and old generation is 1:3

> 增大surivor 区域, SurvivorRatio 默认为8
SurvivorRatio 为2,   eden: surivor1: surivor2的比例为 2:1:1, 增大了surivor, 减小了eden, 防止surivor 区域太小而导致新生带对象过早进入老年代.

> 如果一次old gc之后, old gc 的剩余对象很小, 则证明很多都不是真正的老对象, 需要提升门槛;

* 降低old gc 的时间
目前还没有比较普遍适用的方法, 

> full gc

原因: 
1. 方法区空间不足
2.  CMS GC时出现promotion failed和concurrent mode failure；(日志中会有明细标志)
3.  统计得到的Young GC晋升到老年代的平均大小大于老年代的剩余空间；(可以查看老年代发送full gc时的剩余空间)
4.  主动触发Full GC（执行jmap -histo:live [pid]）来避免碎片问题, 可以通过参数禁止

解决方法:
* 通过把-XX:PermSize参数和-XX:MaxPermSize设置成一样，强制虚拟机在启动的时候就把方法区的容量固定下来，避免运行时自动扩容。
 * CMS默认情况下不会回收Perm区，通过参数CMSPermGenSweepingEnabled、CMSClassUnloadingEnabled ，可以让CMS在Perm区容量不足时对其回收

 **所以, 简而言之, 降低full gc的频率, 一个方向是增大老年代的大小, 二是减小新生代晋升的对象的大小, 提高晋升门槛**, 如果是一次fullgc后，剩余对象不多, 证明很多都不是真正的老对象。那么说明你eden区设置太小，导致短生命周期的对象进入了old区。如果一次fullgc后，old区回收率不大，那么说明old区太小。
 
* jvm heap 大小初始化如何设置
简而言之, 一开始可以根据默认值或者一个大概的估计值去配置, 并设置最大堆和最小堆的范围, 然后触发了 full gc 之后将老年代的大小作为基准值, 其他带都可以根据公式按照这个值去配置. 
https://www.dutycode.com/jvm_xmx_xmn_xms_shezhi.html
![enter image description here](https://drive.google.com/uc?id=1ma3MPNgckROY3F3KSl971__nSeHVpqf6)

> By default, the virtual machine grows or shrinks the heap at each collection to try to keep the proportion of free space to live objects at each collection within a specific range. This target range is set as a percentage by the parameters `-XX:MinHeapFreeRatio=``<minimum>` and `-XX:MaxHeapFreeRatio=``<maximum>`, and the total size is bounded below by `-Xms``<min>` and above by `-Xmx``<max>`.

>    Unless you have problems with pauses, try granting as much memory as possible to the virtual machine. The default size is often too small.( 越大的heap size , 会增加gc pause 时间)
    
> Setting  `-Xms`  and  `-Xmx`  to the same value increases predictability by removing the most important sizing decision from the virtual machine. However, the virtual machine is then unable to compensate if you make a poor choice.( 这两个值设置成一样的话, jvm 就无法调整size)
    
> In general, increase the memory as you increase the number of processors, since allocation can be parallelized.




* parallel gc 参数调优: 
https://docs.oracle.com/cd/E13209_01/wlcp/wlss30/configwlss/jvmgc.html
分为三个方面: Maximum Pause Time Goal,  Throughput Goal,  Footprint Goal(如果前两者都满足了, 则会降低heap 的大小, 目前这个goal 还没有满足)

 The garbage collector tries to meet any pause time goal before the throughput goal.(pause time goal有更高的优先级 )
 吞吐量不够, 需要加大heap size; gc pause 时间太长, 需要减小heap size, 可能会导致heap size 在震荡, 如何协调. 


* ParNew gc collector 调优, 和CMS 配套使用, 负责新生代
 gc collector threads 是多线程的, 仍然会STW


 
* CMS 调优
常用参数解释: 
![enter image description here](https://drive.google.com/uc?id=1dxbRc-uCa9v2HbMKFPf3EAWs9FDeFDlm)

推荐配置, 基本适合大多数场景: 
-XX:+UseConcMarkSweepGC -XX:+UseParNewGC  -XX:+CMSParallelRemarkEnabled  -XX:+UseCMSCompactAtFullCollection  -XX:+UseCMSInitiatingOccupancyOnly -XX:CMSInitiatingOccupancyFraction=75% -XX:-DisableExplicitGC

> 降低CMS gc 时remark阶段的暂停时间: 

因为remark 的时间会随着新生代存活对象的数量增多而上升, 这样如果Remark前执行一次Minor GC，大部分对象就会被回收。CMS就采用了这样的方式，在Remark前增加了一个可中断的并发预清理（CMS-concurrent-abortable-preclean），该阶段主要工作仍然是并发标记对象是否存活，只是这个过程可被中断。此阶段在Eden区使用超过2M时启动，当然2M是默认的阈值，可以通过CMSScheduleRemarkEdenSizeThreshold 参数修改, 并且当eden 空间使用率小于CMSScheduleRemarkEdenPenetration 这个比例时中断, 如果此阶段执行时等到了Minor GC，那么上述新生代不可达的对象将被回收，Reamark阶段需要扫描的对象就少了。
除此之外CMS为了避免这个阶段没有等到Minor GC而陷入无限等待，提供了参数CMSMaxAbortablePrecleanTime ，默认为5s，含义是如果可中断的预清理执行超过5s，不管发没发生Minor GC，都会中止等待minor gc，进入Remark. 对于这种情况，CMS提供CMSScavengeBeforeRemark参数，用来保证Remark前强制进行一次Minor GC。

* 参考文献
https://tech.meituan.com/jvm_optimize.html
[高级语言虚拟机论坛](https://hllvm-group.iteye.com/)





#### GC 常用命令
具体要参考oracle jvm option
* -verbose:class , -verbose:gc ,-verbose:jni 
 [https://dzone.com/articles/how-use-verbose-options-java](https://dzone.com/articles/how-use-verbose-options-java)
 -verbose:class is used to display the information about classes being loaded by JVM. This is useful when using class loaders for loading classes dynamically or for analysing what all classes are getting loaded in a particular scenario. 

-XX:+PrintGCDetails , -XX:+PrintGCTimeStamps

#### 开发中的GC优化
1. 尽量少使用临时对象, 局部变量尽量使用基本数据类型, 也可以避免装箱; 用StringBuffer, 不用string做累加.

> StringBuffer是线程安全的; StringBuilder 不是线程安全的(所以内部没有一个缓存的数组), 适合单线程快速使用后丢弃, 

2. 对象不用时显式置为null 
3. 尽量少用静态对象变量
> static变量被class 引用, class被classloader引用, 除非classloader is reloaded, 例如webapp reload, 否则static变量不会被垃圾回收.

```
// 如果只是想临时用一下static, 可以用static block, 在block结束之后, 就会被GC; 或者在static reference不使用之后, 显式赋为null
class MyUtils {
   static
   {
      MyObject myObject = new MyObject();
      doStuff(myObject, params);
   }

   static boolean doStuff(MyObject myObject, Params... params) {
       // do stuff with myObject and params...
   }
}
```

## 类加载机制

* 应用里类加载的详细过程

首先是触发: 调用到这个类的static 字段, 或者有static block 等触发条件, 然后根据调用者的classloader 去加载这个类, 如果这个类已经被该CL 加载过, 则直接从内存中拿, 否则会读取类的字节码去初始化这个类. 如果这个类(包名.类名 唯一确定一个类, 但是类的jar 包可能有多版本)是其他版本的, 且已经被加载过, 就会出现诸如方法找不到等类冲突异常, 

解决的核心思想: 不同的业务使用不同的线程池，线程池内部共享同一个 contextClassLoader，线程池之间使用不同的 contextClassLoader，就可以很好的起到隔离保护的作用，避免类版本冲突。

https://github.com/alipay/sofa-ark 解决类冲突问题的开源方案.
https://juejin.im/post/5c04892351882516e70dcc9b
https://docs.oracle.com/javase/specs/jls/se7/html/jls-12.html 



参考自 [link](http://www.importnew.com/28445.html)
* 类加载的步骤
1. 加载
 分为预加载和运行时加载, 

 预加载, 虚拟机启动的时候加载rt.jar的class, 像java.lang.*、java.util.*、java.io.*等等, 可以设置虚拟机参数 -XX+TraceClassLoading 来获取类加载信息

 运行时加载: 在用到一个class文件的时候, 如果内存中没有则按类的全限定名来加载.

 加载阶段: 
     1. 获取class文件的二进制流 , 例如从zip包中获取，这就是以后jar、ear、war格式的基础
从网络中获取，典型应用就是Applet
运行时计算生成，典型应用就是动态代理技术
由其他文件生成，典型应用就是JSP，即由JSP生成对应的.class文件
从数据库中读取，这种场景比较少见
     2. 将类信息, 静态变量, 字节码, 常量等内容放到方法区
     3. 内存中生成java.lang.Class的对象, 作为访问入口


2. 验证
 这个地方要说一点和开发者相关的。.class文件的第5~第8个字节表示的是该.class文件的主次版本号，验证的时候会对这4个字节做一个验证，高版本的JDK能向下兼容以前版本的.class文件，但不能运行以后的class文件(向后兼容)，即使文件格式未发生任何变化，虚拟机也必须拒绝执行超过其版本号的.class文件。举个具体的例子，如果一段.java代码是在JDK1.6下编译的，那么JDK1.6、JDK1.7的环境能运行这个.java代码生成的.class文件，但是JDK1.5、JDK1.4乃更低的JDK版本是无法运行这个.java代码生成的.class文件的。如果运行，会抛出java.lang.UnsupportedClassVersionError，这个小细节，务必注意。

3. 准备
 为类变量(static 变量, 不是实例变量)分配内存并设置其初始值, 均在方法区分配

这个阶段赋初始值的变量指的是那些不被final修饰的static变量，比如”public static int value = 123;”，value在准备阶段过后是0而不是123，给value赋值为123的动作将在初始化阶段才进行；比如”public static final int value = 123;”就不一样了，在准备阶段，虚拟机就会给value赋值为123。

4. 解析
将符号引用替换为直接引用的过程, 
符号引用, 包括: 类和接口的全限定名; 字段的名称和描述符; 方法的名称和描述符

例如下面这串代码: 
```
package com.xrq.test6;
 
public class TestMain
{
    private static int i;
    private double d;
     
    public static void print()
    {
         
    }
     
    private boolean trueOrFalse()
    {
        return false;
    }
}
```
用javap 进行反汇编
```
Constant pool:
   #1 = Class              #2             //  com/xrq/test6/TestMain
   #2 = Utf8               com/xrq/test6/TestMain
   #3 = Class              #4             //  java/lang/Object
   #4 = Utf8               java/lang/Object
   #5 = Utf8               i
   #6 = Utf8               I
   #7 = Utf8               d
   #8 = Utf8               D
   #9 = Utf8               <init>
  #10 = Utf8               ()V
  #11 = Utf8               Code
  #12 = Methodref          #3.#13         //  java/lang/Object."<init>":()V
  #13 = NameAndType        #9:#10         //  "<init>":()V
  #14 = Utf8               LineNumberTable
  #15 = Utf8               LocalVariableTable
  #16 = Utf8               this
  #17 = Utf8               Lcom/xrq/test6/TestMain;
  #18 = Utf8               print
  #19 = Utf8               trueOrFalse
  #20 = Utf8               ()Z
  #21 = Utf8               SourceFile
  #22 = Utf8               TestMain.java
```
> 看到Constant Pool也就是常量池中有22项内容，其中带”Utf8″的就是符号引用。比如#2，它的值是”com/xrq/test6/TestMain”，表示的是这个类的全限定名；又比如#5为i，#6为I，它们是一对的，表示变量时Integer（int）类型的，名字叫做i；#6为D、#7为d也是一样，表示一个Double（double）类型的变量，名字为d；#18、#19表示的都是方法的名字。
那其实总而言之，符号引用和我们上面讲的是一样的，是对于类、变量、方法的描述。符号引用和虚拟机的内存布局是没有关系的，引用的目标未必已经加载到内存中了。

> 直接引用: 直接引用可以是直接指向目标的指针、相对偏移量或是一个能间接定位到目标的句柄。直接引用是和虚拟机实现的内存布局相关的，同一个符号引用在不同的虚拟机示例上翻译出来的直接引用一般不会相同。如果有了直接引用，那引用的目标必定已经存在在内存中了。

5. 初始化
> 初始化过程是执行一个类的构造器<clinit>()方法的过程, 其实就是给static变量赋予用户指定的值以及执行静态代码块, 虚拟机会保证类在多线程环境下正确的被初始化并同步, 在同一个类加载器下, 一个类只会初始化一次. 

> 以下几种场景, 类会被正常初始化

 1、使用new关键字实例化对象、读取或者设置一个类的静态字段（被final修饰的静态字段除外）、调用一个类的静态方法的时候

 2、使用java.lang.reflect包中的方法对类进行反射调用的时候

 3、初始化一个类，发现其父类还没有初始化过的时候

 4、虚拟机启动的时候，虚拟机会先初始化用户指定的包含main()方法的那个类

---

* 除了上面4种场景外，所有引用类的方式都不会触发类的初始化，称为被动引用，接下来看下被动引用的几个例子：

1、子类引用父类静态字段，不会导致子类初始化。至于子类是否被加载、验证了，前者可以通过”-XX:+TraceClassLoading”来查看

```
public class SuperClass
{
    public static int value = 123;
     
    static
    {
        System.out.println("SuperClass init");
    }
}
 
public class SubClass extends SuperClass
{
    static
    {
        System.out.println("SubClass init");
    }
}
 
public class TestMain
{
    public static void main(String[] args)
    {
        System.out.println(SubClass.value);
    }
}
运行结果为

SuperClass init
```

2、通过数组定义引用类，不会触发此类的初始化
```
public class SuperClass
{
    public static int value = 123;
     
    static
    {
        System.out.println("SuperClass init");
    }
}
 
public class TestMain
{
    public static void main(String[] args)
    {
        SuperClass[] scs = new SuperClass[10];
    }
}
```
3、引用静态常量时，常量在编译阶段会存入类的常量池中，本质上并没有直接引用到定义常量的类
```
public class ConstClass
{
    public static final String HELLOWORLD =  "Hello World";
     
    static
    {
        System.out.println("ConstCLass init");
    }
}
 
public class TestMain
{
    public static void main(String[] args)
    {
        System.out.println(ConstClass.HELLOWORLD);
    }
}
运行结果为
Hello World
```
在编译阶段通过常量传播优化，常量HELLOWORLD的值”Hello World”实际上已经存储到了NotInitialization类的常量池中，以后NotInitialization对常量ConstClass.HELLOWORLD的引用实际上都被转化为NotInitialization类对自身常量池的引用了。也就是说，实际上的NotInitialization的Class文件中并没有ConstClass类的符号引用入口，这两个类在编译成Class之后就不存在任何联系了。

* 类与类的加载器
只要当两个类来自同一个class文件,被同一个虚拟机加载,类加载器相同, equals(), isAssignableFrom(), instanceof 才能返回两个类相等.
* 双亲委派模型(parents delegation model)
当一个类加载器收到了类加载的请求, 首先把请求委派给父类加载器执行, 所以所以的加载请求都会首先传递到顶层的启动类加载器, 当父类无法加载时,子加载器才会尝试自己加载
  * 加载器的层次关系
   Bootstrap ClassLoader -> Extension ClassLoader -> Application ClassLoader -> User ClassLoader

* classpath 
可以参考[honghailiang888](https://blog.csdn.net/honghailiang888/article/details/51878866), 非常齐全

* 如何手动编译并运行 java文件
    
    class文件发现规则：class文件所在目录 = classpath + '\' + 包名中的'.'全变成'\', 一般会把运行java, javac程序的当前目录(.)也加入到classpath中, 然后会遍历所有的classpath, 在每个classpath下面找 包名+类名 对应的class文件.

    * 编译java
    > 例如项目的目录结构如下: 
    
```
    
    D:/src/main/java/
                  packageA/A.java
                  packageB/B.java

```
```
import packageB.B;

public class A{
    
    // do something
}

```

当需要在任意目录编译 A.java时, 需要知道所引用的B.java的位置, 假设运行javac的目录为D: , 因为当前目录是D:, 在当前目录下用包名无法找到B.java, 故需要手动指定额外的classpath, 则会在packageA和packageB生成各自的class文件.


```
    javac -classpath src/main/java src/main/java/packageA/A.java
    
```

* 运行java
也需要通过classpath 找到对应的class文件, 并且需要指定 包名.类名 , 项目结构如之前所示, 在D: 下运行java, 
```
    java -classpath src/main/java packageA.A
    或者
    在linux 下面想跑一个class文件, 全限定类名为: packageA.packageB.A
    在需要将目录结构新建为: packageA/packageB/A.class
    在packageA的父目录执行 java packageA.packageB.A 即可, 若需要可以指定当前路径为 classpath, 命令如下: 
    java -classpath . packageA.packageB.A
```

* 实例化对象的过程

[https://yq.aliyun.com/articles/183181](https://yq.aliyun.com/articles/183181)
虚拟机遇到一条 new 指令时：

1.  检查这个指令的参数是否能在常量池中定位到一个类的符号引用，并且检查这个符号引用代表的类是否已被加载、解析和初始化过
2.  类加载检查通过
3.  虚拟机java堆为新主对象分配内存，对象所需内存的大小在类加载完成后便可完全确定

分配内存有两种方法: 指针碰撞和空闲列表, 前者
> 指针碰撞
用在垃圾回收算法回收后, 内存是连续的, 所以可以用一个指针来标示正在使用的内存和空闲内存的分界, 当有内存分配的时候, 直接将指针往后移所需要的内存大小即可

> 空闲列表

如果内存空间不是连续的, 就只能用一个列表维护哪些内存是规整的

4.  虚拟机将分配到的内存空间都初始化为零值（不包括对象头）。所以有时候某些字段不赋初始值就能直接使用
5.  设置对象头
6.  执行 init 方法（否则所有字段还为零值），把对象按照程序员的意愿进行初始化

* 对象的格式

分为对象头, 实例数据和对齐部分. 对象头包括: 哈希码、GC代年龄、锁状态、线程持有的锁、偏向线程ID、时间戳等，另外还包含类型指针; 
因为Hotspot JVM 要求对象的总大小必须为8 字节的整数倍, 所以最后可能需要对齐部分来填充.

* 访问对象
通过方法栈区的局部变量表, 里面包含了基本类型和引用类型, 而引用类型是一个指向对象的指针.
![enter image description here](https://drive.google.com/uc?id=1xqC9q9DkdY-HO8Z77kMvrgt-Fc4UnfvQ)

#### 零散问题
* 字节码的文件格式
https://www.jianshu.com/p/252f381a6bc4
https://www.zhihu.com/question/27339390
* java内部类的存储方式


<!--stackedit_data:
eyJoaXN0b3J5IjpbLTEyOTcwNDg3NTIsMTQ2MTg5OTY0MCw3Nj
Y5MDAwODMsMTk0MTY4MzU2OSwxNzU1MzQ5ODY3LDIxMDIyNDAz
OTMsMTM4MzYwMzg5NSwtMTc1Mzk1MjQxNywxNDM5MTU2OTkxLC
0zNzAwMzQ3NjcsLTE3OTA4MzkxMjEsMTcxMDg4MTIwNCwtMTky
Njk4Nzk5NywtMTM4NTMwNjc5NSwtODEwOTMwNTg3LDE1NjE2MD
kwOTAsMjA3MzI2MTA5NCwtNjM4MTUxNiwtMTA1Mzc4MzkyMCwx
MjcwNDA1MDUzXX0=
-->