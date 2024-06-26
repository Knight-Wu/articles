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
* 为什么例如 ArrayList, 单线程在for 循环remove 的时候会报ConcurrentModifcationEx
因为 remove 的时候会把remove 节点之后的元素都全部arrayCopy 前移一位, 导致下次for 循环会少遍历一个节点, 传统的remove(index) 和remove(obj) 只用于remove 单个节点的情况, 所以需要iterator 去remove, itr 有两个指针, a 一个指向上一次返回的节点index, b 一个指向下一个返回的index, 所以remove arraycopy 之后, 将b 往前移一位就可以了. 

* ArrayList与LinkedList区别
前者更适合随机下标遍历, O(1)的时间复杂度; 后者更适合增删,因为前者需要数组的拷贝,效率比较低,可以当作堆栈、队列和双向队列使用。
* ArrayList与Vector的区别
Vector是线程安全的, 进而效率比较低, 在方法上加了对象锁
* Stack
继承了Vector, 添加了Pop和Push等几个方法, 是线程安全的

* LinkedHashSet 可以说是 LinkedHashMap按照插入顺序排序的特例, 通过维护一个双端链表, 保证插入顺序和遍历顺序一致. 
* TreeSet 是 TreeMap value都相同的特例. 内部是一个二叉搜索树, 是排序的
* HashMap
只能有一个null的key, 可以有多个null的value; hashMap的不是有序的, 

* LinkedHashMap是有序的,可以按照元素插入和访问的顺序排序, LinkedHashMap 的访问序可以方便地用来实现一个 LRU(least and recently used) Cache。在访问序模式下，尾部节点是最近一次被访问的节点 (least-recently)，而头部节点则是最远访问 (most-recently) 的节点。因而在决定失效缓存的时候，将头部节点移除即可。
* TreeMap基于Compartor的排序. 红黑树








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
 * 要加volatile, 因为第一个判断是否为空在synchronize的外面, 若不加volatile,各自线程有自己的缓存,可能导致数据不一致,没初始化完成就被使用的情况.
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

* this
    
    * Suppose you’re inside a method and you’d like to get the reference to the current object. Since that reference is passed secretly by the compiler, there’s no identifier for it. However, for this purpose there’s a keyword: this. The this keyword—which can be used only inside a non-static method—produces the reference to the object that the method has been called for
    * The this keyword is used only for those special cases in which you need to explicitly use the reference to the current object. For example, it’s often used in return statements when you want to return the reference to the current object


### null pointer exception
* 当使用一个reference的时候,但是没有指向任何object,或者指向的object出错,实际为null的时候.
> The best way to avoid this type of exception is to always check for null when you did not create the object yourself." If the caller passes null, but null is not a valid argument for the method, then it's correct to throw the exception back at the caller because it's the caller's fault

### hashmap

![enter image description here](https://drive.google.com/uc?id=1CliBbv1YMdfPtT7NwlZwoxL6MUf_MRmj)
* 一些基础概念
有一个table 数组, 数组的每个槽作为一个bucket, 然后数组的元素可能是一个链表的node ,也可能是一个 红黑树的node. 取决于jdk 的版本和一个bucket 里面entry的数量; 每个key 和val 组成一个entry. 

初始容量 和 负载因子，这两个参数是影响HashMap性能的重要参数。其中，容量表示哈希表中桶的数量 (table 数组的大小)，初始容量是创建哈希表时桶的数量；负载因子是哈希表在其容量自动增加之前可以达到多满的一种尺度，它衡量的是一个散列表的空间的使用程度，负载因子越大表示散列表的装填程度越高，整个hashmap 空间需要的更少, 但是查找时间会增加, 反之愈小。默认的, 当初始容量 capacity(默认16 ) * load factor (0.75 )>  enrty的数量的时候, 会认为需要进行table 数组的扩容了. 当初始容量不是2的n 次方的时候, 会选择比它大的, 但是最小的2的n 次方作为数组的初始容量. 所以当entry 的最大数量小于容量 * 负责因子的时候, 就永远不会进行rehash 

* 负载因子是怎么算出0.75 的
根据这个问题的第三个答案 https://stackoverflow.com/questions/10901752/what-is-the-significance-of-load-factor-in-hashmap, 
公式的原理应该是:  size 为s , 已经有了n 个entry, 当下一个entry 进来的时候, 如何能最小化碰撞的概率, 并且此时空间利用率最大, 然后那个公式给出是n/s 小于 log2 的时候, 碰撞的概率都很小, 然后近似取了个比较容易算的值: 0.75

* 为什么当链表长度大于等于8 的时候, 链表转化为红黑树
这种情况用于hashcode 分布性很差的时候, 出现了很多个hash 冲突的极端情况, 因为当负载因子等于0.75 的时候, list 中元素的个数符合泊松分布: 
https://www.zhihu.com/question/26441147
参见代码注释, 所以出现list size 为8 的情况是极为少见的, 就算出现这个情况, 将链表转化为红黑树, 查找的平均时间复杂度由O(n) 变为O(logn) , 但是需要近似两倍的空间. 

* HashMap 的底层数组长度总是2的n次方的原因有两个

一是当 length=2^n 时：h&(length - 1)  在随机hash 的情况下, 能均匀分布到各个index , 
https://blog.csdn.net/claram/article/details/77750899

假定 length = 50 （非 2 的整数次幂），二进制值为 0011 0010，这里我们使用 8 位二进制数来进行计算。length - 1 = 49，二进制值为 0011 0001。我们计算任何整数与 49 进行与运算的可能的结果如下：
```
0000  0000  //0  
0000  0001  //1  
0001  0000  //16  
0001  0001  //17 
0010  0000  //32  
0010  0001  //33  
0011  0000  //48  
0011  0001  //49
相当于为1的各个位置的排列组合, 
```
可能的结果值为：0、1、16、17、32、33、48、49，对于一个长度为 50 的数组，我们只命中了其中的 8 个index. 

假定 length = 16，length - 1 = 15，二进制值为 0000 1111。我们计算任何整数与 31 进行与运算的可能的结果如下：

```
0000  0000  //0  
0000  0001  //1  
0000  0010  //2  
0000  0011  //3  
0000  0100  //4  
0000  0101  //5  
0000  0110  //6  
0000  0111  //7  
0000  1000  //8  
0000  1001  //9  
0000  1010  //10  
0000  1011  //11  
0000  1100  //12  
0000  1101  //13  
0000  1110  //14  
0000  1111  //15
```
每个 index 都用到了, 均匀分布. 


* null 的key 放在数组的第一位
hashmap key和value都可以为null, 因为key 为null, 则hash值为0, hash& table.length -1 均为0, 所以key 为null的object 放在table[0], val覆盖.



* 解决hash冲突的方法
    * 开放地址法(例如 ThreadLocalMap), 冲突时,往下遍历, 直到找到null的位置; 缺点: 容易产生元素的堆聚, 因为较长的链总是更容易产生冲突, 从而元素会落到链的末尾.
    * 拉链法(HashMap)
    
    
    * 两个方法的优劣, 为什么ThreadLocalMap 采用开放地址法

    因为threadLocal的应用场景决定了数据量并不大, 采用开放地址法, 并采用Fibonacci hashing, 使hash 分布均匀, 在小数据量的时候存取会很快.
  
* hashcode 
作为一个native 的方法, 有以下三点规范, 来自代码注释
1. 当对一个java application 的一次调用中, 必须返回同一个整型, 提供的信息也用于对象的相等比较，且不会被修改,但是在两次调用中, 不必返回相同的值
2. 如果两个对象使用equal 方法区判断是否相等, 则调用hashCode 必须返回相同的结果
3. 如果两个对象使用equal 方法比较是不相等的, 则调用hashcode 方法必须返回不同的结果. 开发者需要意识到对unequal 的对象产生不同的hashcode 有利于提高hashtable 的性能. 

* jdk1.8 hash 方法
```
static final int hash(Object key) {  
    int h;  
    return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);  
}
```
讲原 int32 的 hashcode 无符号右移 16 位在与原 hashcode 异或, 等于说高 16 位不变, 低 16 位是低十六位与高十六位的异或的结果, 因为某些 size 比较小的 map, 例如 size=8, size-1=7, 二进制: 0000 0111, hashcode & (size-1), 只用了低三位, 大大增加了冲突的概率, 

* java 7 和java 8的hashmap 的主要优化在于
https://juejin.im/post/5aa5d8d26fb9a028d2079264#heading-20

1. 定位到数组的位置之后, 在链表往后进行一个个查找的时候, 当链表长度大于8 时, java 8会转化为红黑树, 将查找的时间复杂度O(n), 降低为O(logn)
2. hashcode 函数不同, java 7 中进行了多次与和异或运算, 8 均降低为1次, 边际递减. 
3. resize 的时候, 8 不会重新计算hash 值, 根据hash & oldCapacity == 0 ? 放在元数组位置: 否则放在oldIndex+oldCapacity 的位置. 


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

* HashMap, ConcurrentHashMap,HashTable 的结构，在JDK 1.7 和1.8 中有什么不同

* put时，是加到链表头还是链表尾
在jdk1.8之前是插入头部的，在jdk1.8中是插入尾部的


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

#### String 注意事项

![enter image description here](https://drive.google.com/uc?id=1Vj5wcAnfdZUdF-p9lH-BScBprXBJM0_7)
https://juejin.im/entry/5a4ed02a51882573541c29d5

>  String 为什么是Immutable 不可变的
1. 为了安全性, String 作为网络连接和数据库连接的参数, 防止被修改
2. 用作hasheSet 等的key, 保证了key 的唯一性

* 什么是Immutable  类
简而言之对象的状态一旦初始化之后就是不可变的, 由以下几个直接的现象: 一是final 不能被继承, 不能被子类所修改; 二是每次都返回一个新的对象, 三是无需要多线程的同步 
> Written with [StackEdit](https://stackedit.io/).
<!--stackedit_data:
eyJoaXN0b3J5IjpbNjI5MjkxNDQ0LDIwNjgzNTY1NCwxODQ2Mj
IxMzUwLDM0NzA5Nzg0NywtMTQ1OTMzOTIwNCwyMTMyNzI1NSwt
MTg2NTkxMTc4MywxMzk5Mzc1NzgsMTI1NTY4MTMxMSwtNTc1ND
kxNjQ5LC05NzU5NjQzOTksLTExNzkzMTIwODQsLTIwMTM5Mjc0
MzldfQ==
-->
