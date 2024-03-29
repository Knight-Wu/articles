### 除了加锁, 还有什么方式实现并发安全
1. 采用副本, copy on write, threadlocal
2. cas
3. merge, 类似bitcoin 的多个链的merge.

## copy on write

写时复制技术, 适用于读多写少的场景, 写的时候还是要加锁的, 写写互斥, 也可以只在copy 的时候加, 然后比较老对象的引用, CAS .
读的时候不需要加锁. 

下面这两个问题只适用于openjdk 19, copyOnWriteArrayList
* 为什么写的时候要copy, 而不是直接在原数组上操作, 虽然写的时候已经持有一个全局锁了

因为读的时候没有加锁, 要保证读写的原子性, 否则可能读到异常的写到一半的数据, 所以每次写都copy 一份出来, 然后在新的数据上操作, 最后再指向新的数据, 利用java 赋值语句的原子性, 使得读到的数据要么是旧的, 要么是新的. 

* 为什么遍历的时候不能add 也不能remove

add 和remove 方法都会抛出 UnsupportEx, 因为遍历的时候相当于读, 是不能再去写的, 这个类就不是做这种事的, 并没有实现读写的互斥, 因为你遍历
的时候是根据snapshot 去遍历, 如果你这时候写又有一份可能完全不一样的snapshot, 会造成数据混乱. 

* 那么如何多线程并发的遍历并修改, 并且此时有新的数据写进来. 

貌似只能


# java多线程体系图
[link](https://yq.aliyun.com/articles/61960?utm_campaign=wenzhang&utm_medium=article&utm_source=QQ-qun&utm_content=m_10571)

## 基本概念
### 什么叫线程安全
  多线程调用一个线程安全的对象时, 不需要考虑线程的交替执行,也不需要额外的同步,调用这个对象都能获得正确的结果; 可以简单分为两个方面, 
    1. 多线程环境下代码的调用顺序
    可以依据happen-before原则保证代码执行的顺序如你所愿, 不被重排序 
    2. 存储的可见性
    因为cpu有多核心和对应的多级缓存, 不加额外的同步会导致A核心对应的缓存内容对 B核心的线程不可见.
### 线程安全的实现方法 
* 互斥同步
synchronized, wait notify,  reentrantLock, 阻塞队列
* 非阻塞同步
线程自旋, 使用CAS保证原子
* 无同步方案
对象不共享(局部变量); 线程本地变量(threadLocal); 不可变对象(final)  
    
#### 线程本地变量(threadLocal)
 

每个thread里面持有一个私有的threadLocalMap, map里有个entry数组, entry持有一个ThreadLocal的弱引用,引用关系如下图所示, 虚线表示弱引用 
![image](https://user-images.githubusercontent.com/20329409/41815699-469bcc18-77a5-11e8-9336-53dea76da868.png)
若不存在对threadLocal 的强引用, 则entry会被回收, 变成null, 但是entry中的value未被回收, 若当前线程不结束, 则保持有一条这样的引用链: current thread ref ->current  thread -> threadLocalMap -> entry -> val, 


* 使用方法
1. 使用者需要手动调用remove函数，删除不再使用的ThreadLocal.若后续线程没有结束, 但是却没有使用get, set或remove , 则已经为null的entry对应的value没有释放, 会造成内存泄漏
2. 还有尽量将ThreadLocal设置成private static的，这样ThreadLocal会尽量和线程本身一起消亡。
```
        private void set(ThreadLocal<?> key, Object value) {

            // We don't use a fast path as with get() because it is at
            // least as common to use set() to create new entries as
            // it is to replace existing ones, in which case, a fast
            // path would fail more often than not.

            Entry[] tab = table;
            int len = tab.length;
            int i = key.threadLocalHashCode & (len-1);

            for (Entry e = tab[i];
                 e != null;
                 e = tab[i = nextIndex(i, len)]) {
                ThreadLocal<?> k = e.get();

                if (k == key) {
                    e.value = value;
                    return;
                }

                if (k == null) {// 这段表明key已经被GC, 因为是弱引用
                    replaceStaleEntry(key, value, i);
                    return;
                }
            }

            tab[i] = new Entry(key, value);
            int sz = ++size;
            if (!cleanSomeSlots(i, sz) && sz >= threshold)
                rehash();
        }
        
```
> 问题 
* 发现entry的key, 即threadLocal, 被GC后, 会对无用的entry进行资源释放(对value和entry进行置空), 并再次rehash, 这过程没看懂, 感觉都是为了释放内存, 防止内存溢出
    
    
        


#### 不可变对象(final)  
只能初始化一次,不能对原始变量进行赋值或修改对象的引用(可以修改引用的属性).对有可能需要线程安全的变量var,若需要声明为static,则需要为private,且对外部调用该var的方法实现同步,加volatile或synchronize;
否则若是public,则需要声明为final,防止外部修改,这样就不能进行变量的修改操作了





## 锁理论
参见 [link](https://segmentfault.com/a/1190000004935026)
#### 自旋锁
* 简单自旋锁
线程a占用锁的时候, 线程b此时不能获取, 线程b 不会阻塞 blocked, 会一直占用cpu, 跟互斥锁相反

* 自适应自旋锁
若历史自旋均成功获取锁, 则下一次可能自旋更长时间, 反之不然

```
public class TASLock {
    AtomicBoolean state = new AtomicBoolean(false);

    public void lock() {
        while (state.getAndSet(true)) {
            ;
        }
    }

    public void unlock() {
        state.set(false);
    }
}

// 更高性能的写法
public class TTASLock {
    AtomicBoolean state = new AtomicBoolean(false);

    public void lock() {
        while (true) {
            while (state.get()) {
                ;
            }
            if (! state.getAndSet(true)) {
                return;
            }
        }
    }

    public void unlock() {
        state.set(false);
    }
}
// 简而言之, 相比第一种写法避免了其他等待线程一直在写, 导致缓存和主存间不停的数据交换, 也占用了总线, 直接表现就是释放锁和获取锁都变慢, 
但是第二种写法 持有锁的线程在释放锁的时候, 会引起其他线程竞争, 造成总线流量暴增, 难以获取到锁.
```
* 自旋锁和互斥锁的区别
 自旋锁在等待时会占用cpu时间片, 适用于线程切换开销大于等待的情况; 互斥锁在等待时会被阻塞, 不占用cpu时间片,但是cpu切换上下文会消耗额外资源, 适用于线程等待时间大于线程切换的情况
* 指数后退锁

```
public class BackoffLock {
    private AtomicBoolean state = new AtomicBoolean(false);
    private static final int MIN_DELAY = 10;
    private static final int MAX_DELAY = 100;

    public void lock() {
        Backoff backoff = new Backoff(MIN_DELAY, MAX_DELAY);
        while (true) {
            while (state.get()) {
                ;
            }
            if (! state.getAndSet(true)) {
                return;
            } else {
                backoff.backoff();
            }
        }
    }

    public void unlock() {
        state.set(false);
    }
}

class Backoff {
    private final int minDelay, maxDelay;
    int limit;
    final Random random;

    public Backoff(int min, int max) {
        minDelay = min;
        maxDelay = max;
        limit = minDelay;
        random = new Random();
    }

    public void backoff() {
        int delay = random.nextInt(limit);
        limit = Math.min(maxDelay, 2 * limit);
        try {
            Thread.sleep(delay);
        } catch (InterruptedException e) {
            ;
        }
    }
}
```
* 排队自旋锁(基于数组的锁)
每个线程按照到来的先后顺序进行排队, 每个时刻只能由一个线程获取锁, 其他线程在非阻塞等待, 当持有锁的线程释放时, 排下个位置的线程才能获取到锁.

```
public class ALock {
    ThreadLocal<Integer> mySlotIndex = new ThreadLocal<Integer>();
    AtomicInteger tail;
    volatile boolean [] flag;
    int size;

    public ALock() {
        size = 100;
        tail = new AtomicInteger(0);
        flag = new boolean[size];
        flag[0] = true;
    }

    public void lock() {
        int slot = tail.getAndIncrement() % size;
        mySlotIndex.set(slot);
        while (! flag[slot]) {
            ;
        }
    }

    public void unlock() {
        int slot = mySlotIndex.get();
        flag[slot] = false;
        flag[(slot + 1) % size] = true;
    }
}
```
*问题*
CLH队列锁, MCS锁, 遗留



#### CAS(Compare and set)操作
如果当前值和期望值相等, 则才赋值, 等于说是一种非阻塞的多线程赋值操作.但是会带来cache一致性流量问题, 导致多组cache在同步内存时,导致总线流量增加, cpu使用率增加, 线程竞争的问题.



### 缓存一致性协议
[博文](http://www.infoq.com/cn/articles/cache-coherency-primer/)
cpu具有多级缓存, cpu只能逐层找数据, 例如从一级缓存找二级, 最后找到主存
> 基本定律：在任意时刻，任意级别缓存中的缓存段的内容，等同于它对应的内存中的内容。

> 回写定律：当所有的脏段被回写后，任意级别缓存中的缓存段的内容，等同于它对应的内存中的内容。

* 缓存段
每个缓存段对应一段物理内存

> 当我提到“缓存段”的时候，我就是指一段和缓存大小对齐的内存，不关心里面的内容是否真正被缓存进去（就是说保存在任何级别的缓存中）了。


> 缓存一致性协议有多种，但是你日常处理的大多数计算机设备使用的都属于“窥探（snooping）”协议

* 窥探（snooping）
因为所有的内存传输都在总线(bus)上,所有的处理器都能看到总线; 同一个指令周期只有一个缓存能够读写内存; 
当一个缓存去读写内存的时候, 其他处理器都会得到通知, 使得自己缓存所对应的段失效.

* MESI以及衍生协议
    * 失效（Invalid）缓存段，要么已经不在缓存中，要么它的内容已经过时。为了达到缓存的目的，这种状态的段将会被忽略。一旦缓存段被标记为失效，那效果就等同于它从来没被加载到缓存中。
    * 共享（Shared）缓存段，它是和主内存内容保持一致的一份拷贝，在这种状态下的缓存段只能被读取，不能被写入。多组缓存可以同时拥有针对同一内存地址的共享缓存段，这就是名称的由来。
    * 独占（Exclusive）缓存段，和S状态一样，也是和主内存内容保持一致的一份拷贝。区别在于，如果一个处理器持有了某个E状态的缓存段，那其他处理器就不能同时持有它，所以叫“独占”。这意味着，如果其他处理器原本也持有同一缓存段，那么它会马上变成“失效”状态。
    * 已修改（Modified）缓存段，属于脏段，它们已经被所属的处理器修改了。如果一个段处于已修改状态，那么它在其他处理器缓存中的拷贝马上会变成失效状态，这个规律和E状态一样。此外，已修改缓存段如果被丢弃或标记为失效，那么先要把它的内容回写到内存中——这和回写模式下常规的脏段处理方式一样。
  
  
> 第一，在多核系统中，读取某个缓存段，实际上会牵涉到和其他处理器的通讯，并且可能导致它们发生内存传输。写某个缓存段需要多个步骤：在你写任何东西之前，你首先要获得独占权，以及所请求的缓存段的当前内容的拷贝（所谓的“带权限获取的读（Read For Ownership）”请求）。

> 第二，尽管我们为了一致性问题做了额外的工作，但是最终结果还是非常有保证的。即它遵守以下定理，我称之为：
MESI定律：在所有的脏缓存段（M状态）被回写后，任意缓存级别的所有缓存段中的内容，和它们对应的内存中的内容一致。此外，在任意时刻，当某个位置的内存被一个处理器加载入独占缓存段时（E状态），那它就不会再出现在其他任何处理器的缓存中。

## 原生同步
#### 同步代码块

* synchronized
   保证互斥访问; 保证可见性; 禁止指令重排序, 保证顺序性;
   用对象的monitor 锁来实现, 
    * 三种用法, 推荐使用第三种, 因为前两种等于直接暴露了锁对象, 锁对象是实例, 任何其他可以获得该实例的外部代码, 都有可能恶意获取该锁, 并锁住; 锁方法和锁实例等于是直接锁住了该对象, 其他线程想要获取其他不需要同步的方法也会阻塞.
    ```
    // 第一种：直接锁方法
    public synchronized void print(){
        ...//逻辑代码
    }
    // 第二种：锁方法中的代码块
    public void print(){
        synchronized(this){
                ...//需要同步的代码
        }  
             ...//非同步代码
    }
    第三种方法：单独创建锁对象
    private final SynObj=new Object();
    public void print(){
        synchronized(SynObj){
                ...//需要同步的代码
        }  
             ...//非同步代码
    }
    
    
    ```
 * synchronized 与 reentrantlock 的区别
        * 后者更灵活, 可以跨方法解锁, 可以实现公平锁和非公平, 和中断获取锁
        * 原理不同, 前者基于java原生的互斥锁, 线程阻塞; 后者基于AQS
      


    
* wait, notify, notifyAll
    * wait
    见代码注释" this method causees the current thread to place itself in the wait set for this object , thread become disabled for thread scheduling , and 直到被other threads notify, notifyAll this object or interrupt this thread.  ", 
    
    调用之前需要啊占用对象锁, 否则会抛出 IllegalMonitorStateException , 调用后wait 之后会释放lock, 线程变为waited 状态, 直到其他线程调用了notify , notifyAll, 在 wait set的线程会变为blocked 状态, 并且与其他的blocked 的线程竞争lock, 竞争到lock 之后才会变为runnable 状态

> wait 例子, 见"concurrent programming in java second edition doug lea pdf, 3.2.3 Guraded Waits"
```
class GuradedClass{
	protected boolean cond = false;
	// PRE:lock held
	protected void awaitCond() throw InterruptedException{
	while(!cond) wait(); 
	// 使用while 循环, 是为了当wait 被唤醒的时候, 再次检查一遍cond 是否满足, 因为有spurious wakeup的可能; 
	// wait() 执行完之后, 就接着释放了 monitor lock, 等待其他线程唤醒.
}

public synchronized void guardedAction(){ // 要用synchronized ,是因为防止其他对象误调用notify 方法, 因为notify是public 的,
 try{
   awaitCond();
   //如果要执行到这一步, 线程必须竞争到monitor lock, 
 }
 catch(InterruptedException ie){
 // fail
 }
 // actions
}
}

```
> wait的使用场景

https://docs.oracle.com/javase/tutorial/essential/concurrency/guardmeth.html
 
* notify 
    唤醒任意一个正在等待object monitor的线程, 当前线程必须持有monitor , 否则会抛出 IllegalMonitorStateException, 其他在等待monitor的线程会发生竞争
    
  * notifyAll
    与notify的区别是, 会唤醒所有等待线程, 但是只有一个线程可以竞争到monitor, 其他线程在等待释放monitor
    
* wait 与LockSupport, sleep 的区别
    LockSupport不需要获取到 monitor lock , sleep 是Thread.java 的静态方法, 默认是执行方法的线程 sleep, 不需要获取到thread object 的monitor 锁, 而且sleep 之后不需要唤醒, 但是wait 是 Object.java 的实例方法, 执行前需要获取到 monitor lock, 
* 基于blockingQueue 的生产者和消费者, 能够阻塞
```
private static final int size = 10;  
private static final BlockingQueue<String> queue = new LinkedBlockingQueue(size);  
  
class Producer {  
    public void produce(String msg) {  
        queue.add(msg);  
    }  
}  
  
class Consumer {  
    public String consume() {  
        String msg = null;  
        try {  
            msg = queue.take();  
        } catch (InterruptedException e) {  
            e.printStackTrace();  
        }  
        return msg;  
    }  
}
```
 * 基于wait, notify 的生产者和消费者
```
ArrayDeque<String> queue = new ArrayDeque<String>(16);  
private Object notFull = new Object();  
private Object notEmpty = new Object();  
  
class Producer {  
    public void produce(String msg) {  
        synchronized (queue) {  
            while (queue.size() == 16) {  
                try {  
                    notFull.wait(); // queue is full   
					 } catch (InterruptedException e) {  
                    e.printStackTrace();  
                }  
            }  
            queue.add(msg);  
            notEmpty.notifyAll();  
        }  
    }  
  
    class Consumer {  
        public String consume() {  
            String msg = null;  
            synchronized (queue) {  
                while (queue.isEmpty()) {  
                    try {  
                        notEmpty.wait();  // queue is empty  
					 } catch (InterruptedException ex) {  
                    }  
                    msg = queue.poll();  
                    assert msg != null;  
                    notFull.notifyAll();  
  
                }  
            }  
            return msg;  
        }  
    }
```
#### 其他线程方法
join, yield, sleep
线程状态: https://docs.oracle.com/javase/8/docs/api/java/lang/Thread.State.html#TIMED_WAITING
* sleep
    不释放正在占用的对象的monitor lock，线程处于TIME_WAITED 的状态
* join
    在主线程中调用threadA.join(), 主线程需要等待threadA 死亡才会继续执行
* yield
    把当前cpu的使用交给调度器进行分配, 调度器可以忽略该信息. 细节见
[link](https://stackoverflow.com/questions/6979796/what-are-the-main-uses-of-yield-and-how-does-it-differ-from-join-and-interr)


#### 阻塞和可中断方法
当一个方法能够抛出InterruptedException，侧面说明该方法是可以被阻塞的。如果该方法被中断，会提前结束阻塞状态。

```

// 第一个方法是static，针对当前线程；第二个方法是针对调用线程的；当参数是true的情况，会清除interrupted标志位（意味着isInterrupted 返回false），interrupted仅仅是个标志位，不影响线程的状态。



 public static boolean interrupted() {
        return currentThread().isInterrupted(true);
    }

public boolean isInterrupted() {
        return isInterrupted(false);
    }
    
 public void interrupt() {
    if (this != Thread.currentThread())
        checkAccess();

    synchronized (blockerLock) {
        Interruptible b = blocker;
        if (b != null) {
            interrupt0();           // Just to set the interrupt flag
            b.interrupt(this);
            return;
        }
    }
    interrupt0();
}

// 第三个方法会给当前线程发送interruptEx， 并不会让 interrupted 标志位变为true，所以需要中断线程的话需要如下写法： 

   public static void main(String a[]) {
        try {
            Thread thread = new Thread(new Runnable() {
                public void run() {
                    while (!Thread.currentThread().isInterrupted()) {
                        System.out.println("running");
                        try {
                            sleep(2000);
                        } catch (InterruptedException e) {
                            e.printStackTrace();
                            Thread.currentThread().interrupt();
                        }
                    }
                    System.out.println("thread is going to die.");

                }
            });
            thread.start();
            sleep(5000);
            thread.interrupt();
            thread.join();// 等待thread 线程结束
            System.out.println("thread terminated");
            sleep(1000);
            System.out.println("main thread terminated");

        } catch (Exception e) {
            e.printStackTrace();
        }
    }
    
``` 

#### volatile 
每个线程都有各自的高速缓存, 为了协调cpu 和主存的速度不一致而诞生的, 根据缓存一致性协议, 当用volatile 修饰的时候, 都会把值直接写回主存, 并使其他线程的对应缓存段失效, 并读到最新的值.
[https://www.cnblogs.com/xrq730/p/7048693.html](https://www.cnblogs.com/xrq730/p/7048693.html)
  * The value of this variable will never be cached thread-locally: all reads and writes will go straight to "main memory",and read and write is atomic,means that when write is begined,then read must wait write end,so **volatile i++** is not ensured 一致性.(volatile 的变量如果A 线程发生了写入, 则会保证写入主存, 并且让其他线程的缓存失效, 根据缓存一致性原理, 其他线程的缓存失效后会将主存里更新后的值写到缓存, 从而保证volatile的可见性)
  * Access to the variable acts as though it is enclosed in a synchronized block, synchronized on itself.
  *  This can be useful for some actions where it is simply required that visibility of the variable be correct and order of accesses is not important. Using volatile also changes treatment of long and double to require accesses to them to be atomic.
  *  Happens-Before Guarantee:
 在写volatile变量之前的所有写nonVolatile变量的操作会被写入到main memory,在读volatile变量之后的所有读nonVolitale的操作都会从main memory中读.

            Thread A:
            sharedObject.nonVolatile = 123;
            sharedObject.counter     = sharedObject.counter + 1;
        
            Thread B:
            int counter     = sharedObject.counter;
            int nonVolatile = sharedObject.nonVolatile;
        so the nonVolatile is also be ensured read and write compatibility.this can also used for ensuring instructions order 一致性.


* atomic problem : getter, setter and change reference


>JLS 17.7 
> * Non-Atomic Treatment of double and long

>For the purposes of the Java programming language memory model, a single write
to a non-volatile long or double value is treated as two separate writes: one to each
32-bit half. This can result in a situation where a thread sees the first 32 bits of a
64-bit value from one write, and the second 32 bits from another write.
Writes and reads of volatile long and double values are always atomic.
**Writes to and reads of references are always atomic, regardless of whether they are
implemented as 32-bit or 64-bit values.** but to ensure 一致性, declare variable volatile.

#### 原生锁优化(jdk1.6 之后)
* 参见
[https://my.oschina.net/hosee/blog/615865](https://my.oschina.net/hosee/blog/615865)

JDK 6之前synchronized 依赖于操作系统Mutex Lock所实现的锁我们称之为“重量级锁”，JDK 6中为了减少获得锁和释放锁带来的性能消耗，引入了“偏向锁”和“轻量级锁”。

所以目前锁一共有4种状态，级别从低到高依次是：无锁、偏向锁、轻量级锁和重量级锁。锁状态只能升级不能降级。



* 锁粗化

    在锁释放的开销大于等待锁的开销时, 减少释放和获取锁的次数, 减少同步块的个数.
```
public void demoMethod(){  
		synchronized(lock){   
			//do sth.  
		}  
		//做其他不需要的同步的工作，但能很快执行完毕  
		synchronized(lock){   
			//do sth.  
		} 
	}
这种情况，根据锁粗化的思想，应该合并
public void demoMethod(){  
		//整合成一次锁请求 
		synchronized(lock){   
			//do sth.   
			//做其他不需要的同步的工作，但能很快执行完毕  
		}
	}
	
```
* 锁清除(Lock Elimination)

    锁清除是编译器级别的配置. 因为有些时候在线程安全的场景下使用某些内部做了额外同步的类(例如局部变量中使用到了内部加锁的对象, StringBuffer, Vector), 就可以启用JVM配置: 开启逃逸分析和锁清除, 
    ```
    -server -XX:+DoEscapeAnalysis -XX:+EliminateLocks
    ```
    * 开启逃逸分析, 例如sb这个对象有可能被在程序的其他地方被修改而可能导致线程安全问题, 若编译器分析该对象没有出现逃逸情况, 就可以把多余的锁去掉以提高性能
    ```
    public static StringBuffer craeteStringBuffer(String s1, String s2) {
		StringBuffer sb = new StringBuffer();
		sb.append(s1);
		sb.append(s2);
		return sb;
	}
	
	当JVM参数为：
    -server -XX:+DoEscapeAnalysis -XX:+EliminateLocks
    输出：
    craeteStringBuffer: 302 ms
    JVM参数为：
    -server -XX:+DoEscapeAnalysis -XX:-EliminateLocks
    输出：
    craeteStringBuffer: 660 ms
    ```
* 无锁
即线程在一段循环中一直尝试修改直至成功, CAS 是无锁的实现

* 偏向锁(Biased Locking)

    是指之前已经获取到锁的线程再次请求锁时, 不需再次申请锁, 可以直接进入同步块, 适用于无竞争的情况, 对象头的 Mark Word里存储锁偏向的线程ID, 当有其他线程竞争偏向锁h会升级为轻量级锁
```

-XX:+UseBiasedLocking -XX:BiasedLockingStartupDelay=0

```
* 轻量级锁(Lightweight Locking)
    竞争的线程会通过自旋的方式等待锁的释放, 若当前只有一个线程等待, 则为轻量级锁, 自旋等待, 若自旋超过一定的时间或者来了第三个线程竞争, 则会升级为重量级锁. 

* 重量级锁
竞争不到锁的线程会放弃cpu , 阻塞挂起, 
## JUC多线程支持体系

#### Executor
* 大致流程

    每个新任务到来都会创建一个worker执行任务（不管是否有其他核心线程空闲），如果已经超过核心线程的数量，会把任务放进队列排队, worker继承自AQS 实现了Runnable, 继承自AQS表示worker本身就是一个锁, 且该锁是独占锁, 不可重入, 若被占用表示当前worker正在执行task, 若不被占用表示当前worker空闲.
* 核心线程数
   
* 队列
    * ArrayBlockingQueue
        
        基于数组的有界阻塞队列. 
    * 大致流程
    先获取到ReentrantLock  lock, 若满足await的条件, 然后await, 进入条件队列(condition queue), 释放之前的lock, 并阻塞, 等待其他线程调用signal 唤醒,唤醒了之后再次进入阻塞队列(sync queue)等待获取互斥锁.
    * 读写互斥,读读互斥,写写互斥.

```
    // ArrayBlockingQueue 的功能与下列功能类似, 实际生产中可以直接使用 ArrayBlockingQueue
    
    static class BoundedBuffer {
		final Lock lock = new ReentrantLock();
		final Condition notFull = lock.newCondition();
		final Condition notEmpty = lock.newCondition();

		final Object[] items = new Object[100];
		int putptr, takeptr, count;

		public void put(Object x) throws InterruptedException {
			System .out.println("put wait lock");
			lock.lock();
			System.out.println("put get lock");
			try {
				while (count == items.length) {
					System.out.println("buffer full, please wait");
					notFull.await();
				}
					
				items[putptr] = x;
				if (++putptr == items.length)
					putptr = 0;
				++count;
				notEmpty.signal();
			} finally {
				lock.unlock();
			}
		}




		public Object take() throws InterruptedException {
			System.out.println("take wait lock");
			lock.lock();
			System.out.println("take get lock");
			try {
				while (count == 0) {
					System.out.println("no elements, please wait");
					notEmpty.await();
				}
				Object x = items[takeptr];
				if (++takeptr == items.length)
					takeptr = 0;
				--count;
				notFull.signal();
				return x;
			} finally {
				lock.unlock();
			}
		}
	}
	
	
    
```
* LinkedBlockingQueue

    基于链表的无界阻塞队列,吞吐量高于 ArrayBlockingQueue , Executors.newFixedThreadPool()使用了这个队列
* SynchronousQueue

    不存储元素的阻塞队列, 新任务必须等待有线程空闲,否则一直阻塞. 吞吐量高于 LinkedBlockingQueue, Executors.newCachedThreadPool默认使用这个队列
* PriorityBlockingQueue
> 一个具有优先级的无限阻塞队列
    * 问题总结
> 可能出现以下异常 futuretaskcastex
![futuretaskcastex](https://user-images.githubusercontent.com/20329409/41819566-fa008a06-77f4-11e8-95a9-8426fb60ad8a.png)

> 解决办法如下, 新写一个futureTask, 并重载threadPool的newTaskFor() 方法.
![overridefuturetask](https://user-images.githubusercontent.com/20329409/41819568-fef8703c-77f4-11e8-8953-7d0bb5023f20.png)

    
* 最大线程数

    当队列满了, 才会创建多的线程.
* 线程保持活跃时间
    
    当worker数量大于核心线程的数量, 并且空闲时间大于keepAliveTime之后, 线程会被终止
* 饱和策略(RejectedExecutionHandler)

    当队列和线程池都满的时候, 需要用策略去处理到来的新的任务.
    * AbortPolicy：直接抛出异常。
    * CallerRunsPolicy：只用调用者所在线程(可以说是调用线程池的主线程)来运行任务。
    * DiscardOldestPolicy：丢弃队列里最老的一个任务，并执行当前任务。
    * DiscardPolicy：不处理，丢弃掉。
当然也可以根据应用场景需要来实现RejectedExecutionHandler接口自定义策略。如记录日志或持久化不能处理的任务。
* 线程工厂方法

    用于设置创建线程的工厂，统一管理线程池初始化线程.
* FutureTask
    * Runnable 
    不可返回结果和抛出checked exception
    * Callable 
        可返回结果和抛出checked exception
    * Future
    是一个接口，Future就是对于具体的Runnable或者Callable任务的执行结果进行
    取消、查询是否完成、获取结果、设置结果操作
    * FutureTask
    
        FutureTask implement RunnableFuture

```
public interface RunnableFuture<V> extends Runnable, Future<V> {  
    /** 
     * Sets this Future to the result of its computation 
     * unless it has been cancelled. 
     */  
    void run();  
}  
```

#### Collections
* queue(BlockingQueue)

    大致流程: 读写线程先抢占进入阻塞队列(sync queue), 获取到互斥锁后, 读线程和写线程分别进入一个条件队列, 并写队列里元素为空,则读阻塞,直到新元素写入成功, 则通知读队列的第一个线程,该线程进入阻塞队列排队, 直到获取到锁. 写阻塞同理;
    
    * LinkedBlockingQueue
    * ArrayBlockingQueue
    * ConcurrentLinkedQueue
    
        线程安全的非阻塞队列, 
    
* CopyOnWriteArrayList, CopyOnWriteArraySet
每次写数组的时候，都会用ReentrantLock 锁住，然后生成一个新的数组进行拷贝, 只保证写是线程安全的，不保证写立马被读可见, 因为读的时候没有加锁. 适用于读多写少的场景.

* ConcurrenceHashMap

如果在创建HashMap实例时没有给定capacity、loadFactor则默认值分别是16和0.75。 当好多Node 被映射到同一个桶时，如果这个桶中 node 的数量小于TREEIFY_THRESHOLD(8 )当然不会转化成树形结构存储；如果这个桶中Node 的数量大于了 TREEIFY_THRESHOLD(8 ) ，但是capacity小于MIN_TREEIFY_CAPACITY(64 ) 则依然使用链表结构进行存储，此时会对HashMap进行扩绒, 链表分半到不同的bucket 上, 如果capacity大于MIN_TREEIFY_CAPACITY(64 ) ，则会进行树化


由于get操作不是阻塞的, 所以只能看到更新(包括put和remove )完成的值, 也就是:  update operation happend-before  retrievals .  引用下面一段jdk-1.8的注释来表达这个意思;  
   > retrievals reflect the results of the most recently completed update operations holding upon their onset.  
对于一些集合操作, putAll, clear, 并发的retrievals (检索) 可能只反映部分的结果. 
所以size, isEmpty, containValue 只是反映瞬时的结果, 只是个大概的估算. 



    * put 操作
    val和key 都不能为null, 根据hash, hash & (tab.length-1), 算出数组的下标, 如果下标没有元素, 则进行一次CAS, 失败则进行下次循环, 如果该下标有元素, 则synchronize(头结点), 继续判断如果是链表, 则放在链表的末尾, 如果是红黑树, 则调用红黑树的插值方法, 如果链表长度超过了默认值8 , 则会优先进行tab 数组的扩容, 如果数组的长度大于64, 则会把链表转化为红黑树. 
    
    * initTable
问题: 
中间用了一个 Thread.yield() 具体用处是啥, 注释是当前线程 "lost initialization race ; just spin(旋转) ", 意思是让当前线程自旋一会, 等待下次竞争到锁之后即可走这个线程后续的逻辑? 
    * iterator
  iterator is designed to be used only one thread at a time.多个线程共用一个iterator实例, 可能会抛出 java.lang.IllegalStateException. 所以需要在每个线程用个单独的iterator实例.
 一个线程创建iterator之后对另一个线程对map元素的增删改,是可见的.

以下来自ConcurrentHashMap 的overview
> 用每个bucket 的首节点做锁, 可以减少多持有一个额外的锁的空间消耗, 但是有个缺点是会影响这个list 其他节点的更新,   

> 当多个线程都发现map在扩容的时候, 会协助迁移 node, helpTransfer 这个方法;
>  在transfer 的时候, 碰到 forwarding node (hash 值是 "MOVED"), access and update 操作会重新扫描新的table. 
        
 * 如何对链表加锁    


    jdk1.8之前， 采用分段锁机制, 默认分16个Segment , Segment继承自ReentrantLock, 对每个段的table进行线程间的同步.
从1.8开始，采用CAS 机制，多个线程同时更新只有一个线程能成功。


#### Lock

* ReentrantLock, Condition, ReadWriteLock, CountDownLatch, CyclicBarrier, Semaphore 基本都是基于AQS实现的, 可以说都是AQS的子类.
* AbstractQueuedSynchronizer(AQS)
当多个线程对一个锁的发起竞争，用一个标志位state表示锁是否被占用，要么获取到该锁，否则线程进入队列排队，阻塞。
    * 数据结构
     队列是一个双向链表，节点持有指向前后节点的引用，该节点的线程，该节点类型（是独占还是共享），头结点的线程表示持有锁的线程，后续排队的线程在等待获取锁。
    * 作为父类只关心资源的访问形式（互斥还是共享），资源访问不了了线程如何等待（阻塞），线程等待不了了如何返回，并不关心资源什么时候被释放，这由子类去定义

    * ConditionObject (条件队列)
    用于实现blockQueue, 读线程和写线程分别进入一个条件队列,若元素为空,则读阻塞,直到写入成功, 通知读队列的第一个线程, 写阻塞同理; 用于替换生产者和消费者问题中的 wait, notify, notifyall, Condition的await等同于wait, signal等同于notify.
* ReentrantLock
    * 资源释放标志
    state初始化为0，判断 state==0，且只能由一个线程获取该资源
    * FairSync
    判断资源是否被占用，若被占用进入队列等待(阻塞)，每次cpu释放，只获取队列头节点的后继节点的线程，其他线程仍然等待;
阻塞线程可以用 LockSupport park()方法。
    * NonFairSync
    多个线程直接采用 CAS的方式抢占资源，抢占不到则与 FairSync处理逻辑一样。
    *问题*
    * 阻塞的线程如何取消等待? 通过中断
    
* CountDownLatch
设定一个计数器，使所有线程都等待(头节点自旋，其余节点阻塞），直到计数器变成0，
因为是共享模式的锁，所以头结点得到释放后，会依次释放后续排队的线程，都可以获取到资源。

    * 资源释放标志

        state初始化为count，判断 state==0，则队列所有线程都可以获取该资源。
  
* Semaphore(信号量)
   * 帮助记忆：厕所的钥匙，拿到钥匙（permit）才可以上厕所
    * 资源释放的标志

        permit的数量大于零

> Conceptually, a semaphore maintains a set of permits. Each acquire() blocks if necessary until a permit is available, and then takes it. Each release() adds a permit, potentially releasing a blocking acquirer.
    可由任何人获取 permits ，任何人释放 permits，并不需要permits的持有者才能释放 permit；binary Semaphore（permit=1）这点与Mutex有别，Mutex只能由锁的获取者释放。可以根据资源的数量控制同时访问的线程数量，
    
 
    
* CyclicBarrier(栅栏)
   * 帮助记忆：等待一定数量的线程在一定条件之后，一起开始运行某个任务。类似于运动员等待着发令枪一样。
    * 与CountDownLatch的区别
    
    CountDownLatch强调主线程（可能多个）等待任务线程完成任务后执行，任务线程执行完需要countDown(), countDown可以在多个线程中调用, 也可以在一个线程中调用多次, 且 CountDownLatch不能重用；CyclicBarrier 强调多个线程完成各自任务后，一起执行某个后续的任务，例如MapReduce. await必须在各个线程中调用, 若某个线程在await中超时, 中断或者结束, 其他在等待await的线程会收到BrokenBarrierException , 且可以通过调用reset恢复初始状态, 重用.

> A synchronization aid that allows a set of threads to all wait for each other to reach a common barrier point. The barrier is called cyclic because it can be re-used after the waiting threads are released.

 

#### 非阻塞算法
> 一个线程的失败和挂起不会引起其他些线程的失败和挂起，这样的算法称为非阻塞算法。非阻塞算法通过使用底层机器级别的原子指令来取代锁，从而保证数据在并发访问下的一致性。在更细粒度(机器指令)的级别上保证操作是原子的, 并发性提高, 且大大减少线程切换的开销. 同时，由于几乎所有的同步原语都只能对单个变量进行操作，这个限制导致非阻塞算法的设计和实现非常复杂。

* 例如CAS ，原子变量类

* 重量级锁的劣势
    
    JVM可以对非竞争锁的获取和释放进行优化，但是多个线程同时请求锁，JVM就需要求助操作系统，导致一些线程被挂起再恢复；挂起和恢复的过程会带来很大开销，而且当占有锁的线程被休眠时会导致其他线程也暂停所以当频繁发生锁的竞争时，会带来性能的降低。

* 轻量级锁CAS
    * 优点
    基于操作系统层级的调用，不经过JVM ？ 更快，非阻塞算法，不会引起线程的挂起和恢复，只保证操作是原子的，但是要保障数据的一致性（内存的可见性），仍然需要对数据加上volatile。
    * 缺点
    线程可能需要处理竞争的结果；会出现ABA问题，可以用AtomicStampedReference，记录每次更新的版本号。
* AtomicReference
可以判断一个实例是否被更新过，比较当前对象是否和之前的对象相等（==），等于再更新

* 原子类和加锁（ReentrantLock)的性能比较
在较低的竞争下，原子类（不会有线程切换的开销）的吞吐量更大； 高竞争下，加锁的吞吐量更大；可以类比于堵车时信号灯更好，拥堵较低时环岛吞吐量更好。

*问题* 

* 线程转换的状态，waiting和block的区别
    * waiting 指等待另一个线程的某个动作, 例如等待notify, 等待其他线程结束.
    * block 指等待获取monitor lock.都不会消耗cpu时间片

####  java memory model
https://en.wikipedia.org/wiki/Java_memory_model

The  **Java memory model**  describes how  [threads](https://en.wikipedia.org/wiki/Thread_(computer_science) "Thread (computer science)")  in the  [Java programming language](https://en.wikipedia.org/wiki/Java_(programming_language) "Java (programming language)")  interact through memory. Together with the description of single-threaded execution of code, the memory model provides the  [semantics](https://en.wikipedia.org/wiki/Formal_semantics_of_programming_languages "Formal semantics of programming languages")  of the Java programming language.

The original Java memory model, developed in 1995, was widely perceived as broken, preventing many runtime optimizations and not providing strong enough guarantees for code safety. It was updated through the  [Java Community Process](https://en.wikipedia.org/wiki/Java_Community_Process "Java Community Process"), as Java Specification Request 133 (JSR-133), which took effect in 2004, for  [Tiger (Java 5.0)](https://en.wikipedia.org/wiki/Java_version_history#J2SE_5.0_(September_30,_2004) "Java version history").

* 在多处理器的体系下, 处理器会牺牲一定程度的存储一致性(主存和处理器缓存的数据一致),来换取性能的提升,导致代码执行的顺序跟预想的不一样.
* happens-before
    * 定义
    
        想要保证动作A的结果被B所看到, 则A,B必须满足happens-before原则
    * 原则  
        > 见 concurrence in practice 16.1.1 P225 
* final 
final 修饰的field(例如final 数组, final map), 可以保证在多线程环境下被正确初始化之后才被调用,防止指令的重排序,注意若用final 修饰的object , 只有final修饰的域才能保证初始化是线程安全的.

* 指令重排序
`Object obj = new Object();`这样的操作在多线程的情况下会拿到一个未初始化的对象这点可能有疑惑，这里也做个简单的说明。以上java语句分为4个步骤：

1.  在栈中分配一片空间给obj引用
2.  在jvm堆中创建一个Object对象，注意这里仅仅是分配空间，没有调用构造方法
3.  初始化第2步创建的对象，也就是调用其构造方法
4.  栈中的obj指向堆中的对象

以上步骤看起来也是没有问题的，毕竟创建的对象要调用完构造方法后才会被引用。

但问题是jvm是会对指令进行重排序的，重排之后可能是第4步先于第3步执行，那这时候另外一个线程读到的就是没有还执行构造方法的对象，导致未知问题。jvm重排只保证重排前和重排后在**单线程**中的结果一致性。

注意java中引用的赋值操作一定是原子的，比如说a和b均是对象的情况下不管是32位还是64位jvm，`a=b`操作均是原子的。但如果a和b是long或者double原子型数据，那在32位jvm上`a=b`不一定是原子的（看jvm具体实现），有可能是分成了两个32位操作。 但是对于voliate的long,double 变量来说，其赋值是原子的。具体可以看这里[https://docs.oracle.com/javase/specs/jls/se7/html/jls-17.html#jls-17.7](https://docs.oracle.com/javase/specs/jls/se7/html/jls-17.html#jls-17.7)

#### publilsh safely in concurrence
[https://en.wikipedia.org/wiki/Double-checked_locking#Usage_in_Java](https://en.wikipedia.org/wiki/Double-checked_locking#Usage_in_Java)
* 
```

// Works with acquire/release semantics for volatile in Java 1.5 and later
// Broken under Java 1.4 and earlier semantics for volatile
class Foo {
    private volatile Helper helper;
    public Helper getHelper() {
        Helper localRef = helper;
        if (localRef == null) {
            synchronized(this) {
                localRef = helper;
                if (localRef == null) {
                    helper = localRef = new Helper();
                }
            }
        }
        return localRef;
    }
    
/*
Note the local variable "localRef", which seems unnecessary. The effect of this is that in cases where helper is
//already initialized (i.e., most of the time), the volatile field is only accessed once (due to "return localRef;"
instead of "return helper;"), which can improve the method's overall performance by as much as 25 percent.
*/

}


// another lazy init, more simple
// Correct lazy initialization in Java
class Foo {
    private static class HelperHolder {
       public static final Helper helper = new Helper();
    }

    public static Helper getHelper() {
        return HelperHolder.helper;
    }
}

```

#### 工作上出现的并发bug
* 并发下,全局变量的导致的线程不安全问题, 通过改为局部变量, 在每个线程的栈区, 则解决问题
* 线程池使用优先级队列, 出现futureTask cant cast to comparable ex.
<!--stackedit_data:
eyJoaXN0b3J5IjpbLTc3MDAzNDc1MCwtMTIzOTA0NzQwLDEwNT
EzMjY5MiwxNjMxMDI3NTA0LDEzMTk3MDY4NDYsMTY5NjUwODc1
MSwtMzg2ODg5NDUwLC0yODE3MDgzNTAsNTI0NDgzMzk4LDIwNz
Q0MDU1MDEsLTU0MzM0MjE2Nl19
-->