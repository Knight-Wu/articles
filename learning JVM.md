
  
##  运行时数据区域
### 程序计数器(programme counter register)
若执行的是非native方法, 则保存下条指令的地址; 若是native方法, 则为空;每个线程独有, 互不影响

### 虚拟机栈(virtual machine stacks), 本地方法栈(native method stack)
每个线程独有, 每个方法创建的时候都会创建一个栈帧(stack frame),用于存储方法的局部变量, 操作数栈等.
 , 虚拟机栈和本地方法栈的不同是,前者执行java方法, 后者执行native方法
* StackOverFlowError
线程请求的栈深度大于虚拟机允许的栈深度, 抛出该异常
* OutOfMemoryError
如果虚拟机栈可以动态扩展,但是扩展时无法申请到固定的内存,会抛出OutOfMemoryError


### java 堆(heap)
存放对象的实例和数组, 所有线程所共有; 如果堆中没有内存完成实例的分配, 并且堆也无法再扩展时,抛出 OutOfMemoryError

### 方法区(Method Area)
线程间共享, 存储每个类的结构,包括运行时常量 *(包括string pool)* ,静态变量,即时编译器编译后的代码等数据

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


* 回收方法区(永久代)
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
WeakReference的对象, 若只被弱引用引用, 不被其他任何强引用引用时, 如果GC运行, 那么该对象就会被回收.例子 ThreadLocal中的Entry 持有对ThreadLocal对象的弱引用, 若所有使用该ThreadLocal的线程均退出, 

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
* 缺点
  * 不适用对象存活率较高的情况,复制的对象太多, 例如老年代
* 标记整理算法
将不可用对象进行一次标记, 并清除, 然后将可用的对象向一端移动,然后清理掉边界以外的内存,有效避免了内存碎片(针对标记清除算法)和需要有一部分空间来作为留存空间(针对标记-复制算法).

* 分代收集算法
对新生代和老年代采用不同的垃圾收集算法, 例如HotSpot新生代采用标记-复制, 老年代采用标记整理
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
基于标记-清除算法, 

过程: 
1. 第一次STW 暂停: 标记被 gc root 的直接引用, 和年轻代对象所引用的对象, 叫做 _initial mark pause_,
2.  Concurrent Mark, 这个阶段不会暂停用户线程, 第一步找到的老年代的root 的直接引用, 标记所有可达的对象, 并非所有老年代中存活的对象都在此阶段被标记，因为在标记过程中对象的引用关系还在发生变化 
3.  Concurrent Preclean（并发预清理）此阶段同样是与应用线程并行执行的，不需要停止应用线程。因为前一阶段是与程序并发进行的，可能有一些引用已经改变。如果在并发标记过程中发生了引用关系变化，JVM 会通过 Card 将发生了改变的区域标记为「脏」区，这就是所谓的卡片标记（Card Marking）。本阶段也会执行一些必要的细节处理，并为 Final Remark 阶段做一些准备工作
4. Concurrent Abortable Preclean(并发可取消的预清理）,不会暂停用户线程
5. 

* 浮动垃圾(Floating Garbage)
由于应用线程和gc 线程并行执行, gc 线程标记的可达的对象在标记结束后又不可达了, 这部分剩余的对象叫做浮动垃圾, 这部分对象会在下次gc 被回收.


* concurrent mode failure
    因为gc时用户线程继续产生新对象, 如果CMS 此时不能完成清除工作, 导致老年代空间满了, 会触发这个失败, 会停止所有的用户线程去做gc, 停顿会很长.
 所以需要预留至少一部分空间用作gc时新对象的产生(-XX: CMSInitiatingOccupancyFraction 设置百分比), 
       * 采用标记清除算法, 内存碎片很多,-XX:UseCMSCompactionAtFullCollection(默认开启, 开启内存碎片的整理工作), 但是会导致停顿时间变长,  也可采用 -XX:CMSFullGCsBeforeCompaction(表示执行了多少次不压缩的full GC后, 来一次压缩的full GC ,默认是0, 表示每次都压缩).
    
* Garbage First(G1)
  * 并行(多线程处理GC, 此时用户线程仍然等待)和并发(减少停顿时间, 和用户线程并行, 垃圾收集器在另一个CPU)
  * 分代收集
  * 空间整合
  整体使用标记-整理, 局部采用标记-复制,故不会有内存碎片.
  * 可预测的停顿

#### GC调优
jvm heap 大小初始化如何设置: 
https://www.dutycode.com/jvm_xmx_xmn_xms_shezhi.html
总结说来, 将full gc 之后老年代的大小作为基准值, 其他带都可以根据公式按照这个值去配置. 

* heap size 相关配置
> By default, the virtual machine grows or shrinks the heap at each collection to try to keep the proportion of free space to live objects at each collection within a specific range. This target range is set as a percentage by the parameters `-XX:MinHeapFreeRatio=``<minimum>` and `-XX:MaxHeapFreeRatio=``<maximum>`, and the total size is bounded below by `-Xms``<min>` and above by `-Xmx``<max>`.

>    Unless you have problems with pauses, try granting as much memory as possible to the virtual machine. The default size is often too small.( 越大的heap size , 会增加gc pause 时间)
    
> Setting  `-Xms`  and  `-Xmx`  to the same value increases predictability by removing the most important sizing decision from the virtual machine. However, the virtual machine is then unable to compensate if you make a poor choice.( 这两个值设置成一样的话, jvm 就无法调整size)
    
> In general, increase the memory as you increase the number of processors, since allocation can be parallelized.

* The Young Generation
Young Generation size 越大, minor gc 频率越低, 永久代就会越小(永久代 + 年轻代等于 heap size), major 会越频繁, major 通常会引起minor , 就是一次full gc.
> By default, the young generation size is controlled by the parameter  `NewRatio`. For example, setting  `-XX:NewRatio=3`  means that the ratio between the young and tenured generation is 1:3. In other words, the combined size of the eden and survivor spaces will be one-fourth of the total heap size.
The parameters  `NewSize`  and  `MaxNewSize`  bound the young generation size from below and above. Setting these to the same value fixes the young generation, just as setting  `-Xms`  and  `-Xmx`  to the same value fixes the total heap size.

jvm 参数调优: 
https://docs.oracle.com/cd/E13209_01/wlcp/wlss30/configwlss/jvmgc.html
分为三个方面: Maximum Pause Time Goal,  Throughput Goal,  Footprint Goal(如果前两者都满足了, 则会降低heap 的大小, 目前这个goal 还没有满足)

 The garbage collector tries to meet any pause time goal before the throughput goal.(pause time goal有更高的优先级 )
 吞吐量不够, 需要加大heap size; gc pause 时间太长, 需要减小heap size, 可能会导致heap size 在震荡, 如何协调. 




* 将SurvivorRatio由默认的8改为2
使surivor的比例增大, eden: surivor1: surivor2的比例为 2:1:1, 增大了surivor, 减小了eden, 影响是: minorGC的频率增大, 因为eden小了; 增大了新生代对象复制的开销, 因为有更多的对象会留在 surivor区域, 但是提高了晋升老年代的门槛, 让新生代对象能进行充分的淘汰才能进入老年代,  使真正的长寿的对象才能进入老年代, 使fullGC的时间变短了. 

* NewParSize调优
 NewParSize表示新生代大小, 增大新生代大小, 则单次minorGC时间变长, 频率下降, 业务读写操作时间抖动较大; 减小新生代大小, 就会使minorGC频率加快, 加快晋升到老年代的速度(因为每minorGC一次, 对象年龄加一), 增加fullGC的机会

### java heap 分代(基于jdk1.8)
* 新生代(PSYoungGen)
 eden space , survivor(from space , to space) ,可设置比例,采用标记复制算法
* 老年代(ParOldGen)
采用标记清除算法或标记整理算法
* Metaspace(方法区)
存储类的信息

### GC回收过程
*  对象优先在eden分配, 当eden没有足够空间时, 进行一次minorGC, 将存在GC root引用链的对象复制到survivor区域, 使用标记复制算法, 然后存活对象的年龄加一. 
* 大对象直接分配到老年代
当对象大小大于参数设置的 -XX:PretenureSizeThreshold (默认是 0 , 即无论多大不会直接进入老年代)
* 长期存活对象进入老年代
当对象出生在eden, 经过一次minorGC进入survivor, 则对象年龄设为1, 当对象年龄超过该参数 -XX:MaxTenuringThreshold 默认15, 则进入老年代
* 动态对象年龄判定
当某个年龄的对象的大小总和超过survivor的一半, 则大于或等于该年龄的所有对象都会进入老年代.
* 空间分配担保
minorGC之前会检查老年代最大的连续内存空间是否大于新生代所有对象的大小,  或者检查最大的老年代的连续内存是否大于历史平均进入老年代的对象大小, 满足一个就进行minorGC, 否则fullGC
> 新生代gc过程
1. 对象先在eden分配, 然后eden满了, 启动一次minor, 存活对象分配到from区, eden清空, 然后eden再次满了, 将eden和from中仍然存活的对象copy到to区, 然后eden和from清空, 之后to和from相对于就对换了, 随后的minor 会再次将eden和from区存活对象复制到to区, 若满足晋升条件则直接晋升到老年代.

* full GC
https://www.zhihu.com/question/41922036/answer/93079526  
> full GC：当准备要触发一次young GC时，如果发现统计数据说之前young GC的平均晋升大小比目前old gen剩余的空间大，则不会触发young GC而是转为触发full GC（因为HotSpot VM的GC里，除了CMS的concurrent collection之外，其它能收集old gen的GC都会同时收集整个GC堆，包括young gen，所以不需要事先触发一次单独的young GC）；或者，如果有perm gen的话，要在perm gen分配空间但已经没有足够空间时，也要触发一次full GC；或者System.gc()、heap dump带GC，默认也是触发full GC。

> Full GC时，就不在分 “young gen使用young gen自己的收集器(一般是copy算法)；old gen使用old gen的收集器(一般是mark-sweep-compact算法)”，而是，整个heap以及perm gen，所有内存，全部的统一使用 old gen的收集器(一般是mark-sweep-compact算法) 一站式搞定


#### GC 常用命令
* -verbose:class , -verbose:gc ,-verbose:jni 
 [https://dzone.com/articles/how-use-verbose-options-java](https://dzone.com/articles/how-use-verbose-options-java)
 -verbose:class is used to display the information about classes being loaded by JVM. This is useful when using class loaders for loading classes dynamically or for analysing what all classes are getting loaded in a particular scenario. 

-XX:+PrintGCDetails , -XX:+PrintGCTimeStamps







###  hotSpot的算法实现
* 什么时候开始GC
 当eden和一个survivor的空间容不下新的对象时,产生minorGC,将长期存活对象移到老年代, 若老年代的空间不够, 则进行fullGC
* GC root
 1. local variable
 2. active java thread
 3. static variable
 4. JNI reference
* 寻找GC Root的引用链
  *  逐个检测(包括方法区), 花费很多时间
  *  GC 产生停顿, 保证引用关系不变
  * 解决办法: 减小时间, 类设置OOP map, 记录引用位置 
  
* 但是导致引用变化的指令可能非常多, 可能导致 OOP Map的所占空间巨大
  *  解决办法:在程序需要长时间执行的地方设置safepoint, 并生成OOP map, 程序跑到safepoint才GC, 并在safepoint设置标志, 跑到这个safepoint的时候判断有没有gc, 若线程处在block等状态, 在一段引用关系不变的代码段设置safeRegion, 随时可以GC

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
用javap把这段代码的.class反编译一下：
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



#### 零散问题
* 字节码的文件格式
https://www.jianshu.com/p/252f381a6bc4

https://www.zhihu.com/question/27339390
* java内部类
<!--stackedit_data:
eyJoaXN0b3J5IjpbNTgyMTMyNzY0LDEzODI2NDA0NDgsLTIwMj
IxMzgxNTIsLTE0MDc1NDU3OTAsLTk0NzY4MzY4NCwtNjY4MTIx
NTgwLC0xODgxMDM3MzY0LDEzNjU2NDAwNTEsOTQ0MDU1NDM2LC
00NTI3NjYzNTYsLTE2MzY0MzkwNzgsLTE3OTQ4NDA3MzksLTIx
NDExNzEzOTIsLTEyMzY2ODA2NjMsLTQwODA3NjMwOSwxMjMzMz
M0Mjc4LDE3OTk0MzM0MCwtNTIzNzM4MTc2LC0xMjk2OTU5MjM4
LC0xNjk5NzEzMzI2XX0=
-->