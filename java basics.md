### 阿里巴巴 java开发手册注意事项
* 如果使用 tab 缩进，必须设置 缩进，必须设置 缩进，必须设置 缩进，必须设置 缩进，必须设置 缩进，必须设置 1个 tab 为 4个空格。 IDEA 设置 tab 为 4个空格时， 请勿勾选 Use tab character ；而在 eclipse 中，必须勾选 insert spaces for tabs 。
* 【强制】POJO类中布尔类型的变量，都不要加is，否则部分框架解析会引起序列化错误。
* 【强制】所有的覆写方法，必须加@Override注解。
* 【强制】Object的equals方法容易抛空指针异常，应使用常量或确定有值的对象来调用equals。推荐使用java.util.Objects#equals （JDK7引入的工具类）
* 【强制】所有的相同类型的包装类对象之间值的比较，全部使用equals方法比较。
* 【强制】关于基本数据类型与包装数据类型的使用标准如下： 1） 所有的POJO类属性必须使用包装数据类型。 2） RPC方法的返回值和参数必须使用包装数据类型。 3） 所有的局部变量【推荐】使用基本数据类型。
* 【强制】POJO类必须写toString方法。使用IDE的中工具：source> generate toString时，如果继承了另一个POJO类，注意在前面加一下super.toString。 说明：在方法执行抛出异常时，可以直接调用POJO的toString()方法打印其属性值，便于排查问题。
* 【推荐】 类内方法定义顺序依次是：公有方法或保护方法 > 私有方法 > getter/setter方法。
* 【强制】使用工具类Arrays.asList()把数组转换成集合时，不能使用其修改集合相关的方法，它的add/remove/clear方法会抛出UnsupportedOperationException异常。可以使用 List list = new ArrayList(Arrays.asList(array));
* 【强制】不要在foreach循环里进行元素的remove/add操作。remove元素请使用Iterator方式，如果并发操作，需要对Iterator对象加锁。
* 【推荐】使用entrySet遍历Map类集合KV，而不是keySet方式进行遍历。
* 【强制】线程池不允许使用 【强制】线程池不允许使用 Executors ,而是通过ThreadPoolExecutor
* 【推荐】使用CountDownLatch进行异步转同步操作，每个线程退出前必须调用countDown方法，线程执行代码注意catch异常，确保countDown方法可以执行，避免主线程无法执行至countDown方法，直到超时才返回结果。 说明：注意，子线程抛出异常堆栈，不能在主线程try-catch到。
* 【强制】在表查询中，一律不要使用 * 作为查询的字段列表，需要哪些字段必须明确写明。 说明：1）增加查询分析器解析成本。2）增减字段容易与resultMap配置不一致。
* 【强制】使用ISNULL()来判断是否为NULL值。注意：NULL与任何值的直接比较都为NULL。 说明： 1） NULL<>NULL的返回结果是NULL，而不是false。 2） NULL=NULL的返回结果是NULL，而不是true。 3） NULL<>1的返回结果是NULL，而不是true。
* 【推荐】所有pom文件中的依赖声明放在<dependencies>语句块中，所有版本仲裁放在<dependencyManagement>语句块中。意思是父pom在dependencyManagement中声明依赖和对应的版本, 子pom只需要引入各自实际需要的dependency而不需要再写版本了. 说明：<dependencyManagement>里只是声明版本，并不实现引入，因此子项目需要显式的声明依赖，version和scope都读取自父pom。而<dependencies>所有声明在主pom的<dependencies>里的依赖都会自动引入，并默认被所有的子项目继承。


### java集合
#### List
* ArrayList与LinkedList区别
前者更适合随机下标遍历, O(1)的时间复杂度; 后者更适合增删,因为前者需要数组的拷贝,效率比较低,可以当作堆栈、队列和双向队列使用。
* ArrayList与Vector的区别
Vector是线程安全的, 进而效率比较低, 在方法上加了对象锁
* Stack
继承了Vector, 添加了Pop和Push等几个方法, 是线程安全的

* LinkedHashSet 可以说是 LinkedHashMap按照插入顺序排序的特例
* TreeSet 是 TreeMap value都相同的特例.
* HashMap
只能有一个null的key, 可以有多个null的value; hashMap的不是有序的, 

* LinkedHashMap是有序的,可以按照元素插入和访问的顺序排序, LinkedHashMap 的访问序可以方便地用来实现一个 LRU(least and recently used) Cache。在访问序模式下，尾部节点是最近一次被访问的节点 (least-recently)，而头部节点则是最远访问 (most-recently) 的节点。因而在决定失效缓存的时候，将头部节点移除即可。
* TreeMap基于Compartor的排序.



* HashTable与HashMap区别
hashTable是线程安全的, hashTable不允许null的key




### 方法重载与重写

* 重载(overload)

  * 指的是方法同名,但形参列表不同
* 重写(override)

  * 指的父子类间,子类重写父类的方法,并覆盖.(除非显示声明super.methodName(),否则调用子类的方法)
  * 子类的访问修饰权限不能小于父类
  * 要使父类的方法不被override,声明为final或者private

## concurrency
* 方法

  * extends Thread
  * implements Runnable
  * Executors
     
    Executors allow you to manage the execution of asynchronous tasks without having to explicitly manage the lifecycle of threads. Executors are the preferred method for starting tasks in Java SE5/6.
* 从task返回值
    
    If you want the task to produce a value when it’s done, you can implement the **Callable** interface rather than the Runnable interface. Callable, introduced in Java SE5, is a generic with a type parameter representing the return value from the method call( ) (instead of run( )), and must be invoked using an ExecutorService submit( ) method

* daemon
    
    A "daemon" thread is intended to provide a general service in the background as long as the program is running, but is not part of the essence of the program. Thus, when all of the non-daemon threads complete, the program is terminated, killing all daemon threads in the process. Conversely(相反的), if there are any non-daemon threads still running, the program doesn’t terminate. There is, for instance, a non-daemon thread that runs main( ).

    A daemon thread is a thread that does not prevent the JVM from exiting when the program finishes but the thread is still running. An example for a daemon thread is the garbage collection.
* join

    One thread may call join( ) on another thread to wait for the second thread to complete before proceeding. If a thread calls t.join( ) on another thread t, then the calling thread is suspended until the target thread t finishes (when t.isAlive( ) is false).
    
### memory
* Difference between “on-heap” and “off-heap”
    
    The on-heap store refers to objects that will be present in the Java heap (and also subject to GC). On the other hand, the off-heap store refers to (serialized) objects that are managed by EHCache, but stored outside the heap (and also not subject to GC). As the off-heap store continues to be managed in memory, it is slightly slower than the on-heap store, but still faster than the disk store

* heap and stack
    * stack is a data structure. LIFO.
    * stack is used to store primitive data type and function call.but heap is to store object.
    * If there is no memory left in the stack for storing function call or local variable, JVM will throw java.lang.StackOverFlowError, while if there is no more heap space for creating an object, JVM will throw java.lang.OutOfMemoryError: Java Heap Space
    *　Variables stored in stacks are only visible to the owner Thread while objects created in the heap are visible to all thread. In other words, stack memory is kind of private memory of Java Threads while heap memory is shared among all thread

### initialize and cleanup
*  constructor

    一旦自定义了构造函数,则默认的构造函数不会被隐式调用.
    ```
     new ClassName()
    ```
    compiler会不懂调用的是哪一个构造函数,必须显示调用.
    

* constructor is atomic or not
    
    构造函数是不具有synchronized的性质,在构造函数执行过程中,对象是对其他线程可见的,而且因为
```
objectA = new ClassA();
```
在不使用volatile的情况下,会被reorder,所以构造函数应该需要手动同步.

* static construct
    * static修饰的field属于某个类(class),而不是类的某个实例(instance),类的所有实例共用一个static 变量,在类初始化之前被初始化(类被调用的时候),只被初始化一次,如果没有明显的赋值,则会被赋予默认值.
    * The JVM won't execute a class's static initializer until you actually touch something in the class
    * Static variables are initialized only once , at the start of the execution 
    * 顺序: static field -> non static field -> class constructor,若有父类则父类优先

**To summarize the process of creating an object, consider a class called Dog:**

* 代码分析一


```
class Foo {
  private volatile Helper helper = null;
  public Helper getHelper() {
    if (helper == null) {
      synchronized(this) {
        if (helper == null) {
          helper = new Helper();
        }
      }
    }
  return helper;
}
```
 * 若不加volatile,各自线程有自己的缓存,可能导致数据不一致,没初始化完成就被使用的情况.
 * 第一个判断是否为空,是为了提升性能,如果不加,每次多线程调用的时候都会有可能产生线程间的等待,降低性能.
 * 第二个判断是否为空,是因为一开始instance为空,有多个线程会进入synchronize块,但是只有一个线程会完成instance的实例化,当这个线程完成实例化后,其他线程不能在对instance进行初始化,所以需要加第二个判断非空.
 * 较好的方法(lazy initialization)
```
public class Something {
    private Something() {}

    private static class LazyHolder {
        private static final Something INSTANCE = new Something();
    }

    public static Something getInstance() {
        return LazyHolder.INSTANCE;
    }
}

```
  Since the class initialization phase is guaranteed by the JLS to be serial, i.e., non-concurrent, so no further synchronization is required in the static getInstance method during loading and initialization
* 代码分析二
```
class MyClass {
  private static MyClass myClass = new MyClass();
  private static MyClass myClass2 = new MyClass();
  public MyClass() {
    System.out.println(myClass);
    System.out.println(myClass2);
  }
}
```
That will print:
```
null
null
myClassObject
null
```
    
  because it divide into two steps,first is initializaion,second is assignment.when the first initialzation is ended,the myClass is not null,but not until the  assignment to myClass2 is done,the myClass2 is  instantiated.
* GC
    * reference count
    
    被引用加一;程序走出引用作用域或者引用赋为null,减一;但是无法针对相互引用的情况,故JVM不采用该实现方法实现GC.
    * 根搜索算法
        * GC root

    1. Local variables are kept alive by the stack of a thread. This is not a real object virtual reference and thus is not visible. For all intents and purposes, local variables are GC roots.
    2. Active Java threads are always considered live objects and are therefore GC roots. This is especially important for thread local variables.
    3. Static variables are referenced by their classes. This fact makes them de facto GC roots. Classes themselves can be garbage-collected, which would remove all referenced static variables. This is of special importance when we use application servers, OSGi containers or class loaders in general. We will discuss the related problems in the Problem Patterns section.
    4. JNI References are Java objects that the native code has created as part of a JNI call. Objects thus created are treated specially because the JVM does not know if it is being referenced by the native code or not. Such objects represent a very special form of GC root, which we will examine in more detail in the Problem Patterns section below.
        
        * mark and swipe 
        
        1.The algorithm traverses all object references, starting with the GC roots, and marks every object found as alive.
        
        2.All of the heap memory that is not occupied by marked objects is reclaimed. It is simply marked as free, essentially swept free of unused objects.
        
        
* this
    
    * Suppose you’re inside a method and you’d like to get the reference to the current object. Since that reference is passed secretly by the compiler, there’s no identifier for it. However, for this purpose there’s a keyword: this. The this keyword—which can be used only inside a non-static method—produces the reference to the object that the method has been called for
    * The this keyword is used only for those special cases in which you need to explicitly use the reference to the current object. For example, it’s often used in return statements when you want to return the reference to the current object


### null pointer exception
* 当使用一个reference的时候,但是没有指向任何object,或者指向的object出错,实际为null的时候.
> The best way to avoid this type of exception is to always check for null when you did not create the object yourself." If the caller passes null, but null is not a valid argument for the method, then it's correct to throw the exception back at the caller because it's the caller's fault

#### hashmap
初始容量 和 负载因子，这两个参数是影响HashMap性能的重要参数。其中，容量表示哈希表中桶的数量 (table 数组的大小)，初始容量是创建哈希表时桶的数量；负载因子是哈希表在其容量自动增加之前可以达到多满的一种尺度，它衡量的是一个散列表的空间的使用程度，负载因子越大表示散列表的装填程度越高，整个hashmap 空间需要的更少, 但是查找时间会增加, 反之愈小。默认的resize的大小=初始容量

HashMap 的底层数组长度总是2的n次方的原因有两个，即当 length=2^n 时：
不同的hash值发生碰撞的概率比较小，这样就会使得数据在table数组中分布较均匀，空间利用率较高，查询速度也较快；
h&(length - 1) 就相当于对length取模，而且在速度、效率上比直接取模要快得多，即二者是等价不等效的，这是HashMap在速度和效率上的一个优化。

hashmap key和value都可以为null, 因为key 为null, 则hash值为0, hash& table.length -1 均为0, 所以key 为null的object 放在table[0], val覆盖.



* 解决hash冲突的方法
    * 开放地址法(例如 ThreadLocalMap), 冲突时,往下遍历, 直到找到null的位置; 缺点: 容易产生元素的堆聚, 因为较长的链总是更容易产生冲突, 从而元素会落到链的末尾.
    * 拉链法(HashMap)
    *问题* 
    * 两个方法的优劣, 为什么ThreadLocalMap 采用开放地址法
    因为threadLocal的应用场景决定了数据量并不大, 采用开放地址法, 并采用Fibonacci hashing, 使hash 分布均匀, 在小数据量的时候存取会很快.
  
* hashcode 
作为一个native 的方法, 有以下三点规范, 来自代码注释
1. 当对一个java application 的一次调用中, 必须返回同一个整型, 提供的信息也用于对象的相等比较，且不会被修改,但是在两次调用中, 不必返回相同的值
2. 如果两个对象使用equal 方法区判断是否相等, 则调用hashCode 必须返回相同的结果
3. 如果两个对象使用equal 方法比较是不相等的, 则调用hashcode 方法必须返回不同的结果. 开发者需要意识到对unequal 的对象产生不同的hashcode 有利于提高hashtable 的性能. 

* java 7 和java 8的hashmap 的主要区别在于
定位到数组的位置之后, 在链表往后进行一个个查找的时候, 当链表长度大于8 时, java 8会转化为红黑树, 将查找的时间复杂度O(n), 降低为O(logn)


* put 大致过程
根据hash, 找到需要存放的entry 数组的index, 如果该位置没元素, 就直接放入, 如果有, 则判断该位置是不是红黑树的节点, 是的话就调用红黑树的插入方法(默认链表长度大于8, 则会变为红黑树), 否则遍历链表, 放到链表末尾, 如果插入后, 链表长度大于等于9, 则将该链表转化为红黑树, 最后如果插入后, hashMap size大于threshold, 则进行resize. 

* resize 的大致过程
resize 成原来容量的两倍, 如果原先数组的位置只有一个元素, 就直接取模迁移到新数组的位置, 如果是红黑树则是调用红黑树的分裂方法, 如果是链表, 则分成两个子链表, 一个放在原位置, 另一个放在 (原位置+oldCapacity).

1. 返回bucket的index
 ``` 
int indexFor(int hash,int length)
  return hash & (length-1)
```
比取余数(hash % length)的方法更加的快,

* hashmap线程不安全的表现
> 多线程put的情况下, 同时进入了rehash, 出现死循环; 同时在遍历map的时候, 如果有其他线程改变了map的结构, 会抛出ConcurrentModificationException, 是fail-fast的策略, 目的是为了提醒线程安全的问题. 

* 为什么String, Interger这样的wrapper类适合作为键
> 保证key是不可变的, 具体说来就是hashCode不能变, 作为键的对象, 需要重写hashCode和equal方法, 保证能正常获取到之前的对象. 


## 泛型
1.使用原生态类型
> 为了兼容老的jdk1.5之前的Collection的代码,仅有这个不是类型安全的,编译期不能保证类型安全,可能在运行期出现,例如可能将java.util.Date插入到java.sql.Date中,很难发现

## inner class
* inner class cannot be inited outside the parent class scope, except inner class is static


### difference between static class and singleton
* singleton can 实现多态,继承或者实现接口,但是static不能
* 在spring框架中, singleton可以被DI, 但是static不能autowire
```
// 实在要用, 也可以这样注入static field
private static Class a;
@Autowired
public void setClass(A a){
    TheClass.a = a;
}
```
反正最好用singleton

### compression 压缩
* lz4
[lz4 stream example](https://stackoverflow.com/questions/36012183/java-lz4-compression-using-input-output-streams)

### 代理
分为静态代理和动态, 静态指代理类在运行前就生成的, 动态则指运行时才生成, 由中介类生成, 中介类需要实现 InvacationHandler 接口, 等于说中介类生成代理类, 代理类去真正代理, 所以说动态代理类的类名也都是由程序生成的. 
> Written with [StackEdit](https://stackedit.io/).
<!--stackedit_data:
eyJoaXN0b3J5IjpbLTg3NTg5MDYxMiwtMTE4OTk1OTU0MiwtMT
g2NTY5NzE3LDE1MjAxOTc3NzEsMjg1NTkwMDQ3LDExNTY3OTU3
LDEzMjkyODczMzksMTc4OTQyMDM1NF19
-->