

## 死锁条件
这四个条件是产生死锁的 必要条件 ，也就是说只要系统发生死锁，这些条件必然成立

互斥：资源必须处于非共享模式，即一次只有一个进程可以使用。如果另一进程申请该资源，那么必须等待直到该资源被释放为止。
占有并等待：一个进程至少应该占有一个资源，并等待另一资源，而该资源被其他进程所占有。
非抢占：资源不能被抢占。只能在持有资源的进程完成任务后，该资源才会被释放。
循环等待：有一组等待进程 {P0, P1,..., Pn}， P0 等待的资源被 P1 占有，P1 等待的资源被 P2 占有，......，Pn-1 等待的资源被 Pn 占有，Pn 等待的资源被 P0 占有。

## 思路
定义两个资源o1，o2
对象deadLock1占有资源o1，需要资源o2
对象deadLock2占有资源o2，需要资源o1
死锁产生

## 代码
```
public class DeadLock implements Runnable {

    // flag=1，占有对象o1，等待对象o2
    // flag=0，占有对象o2，等待对象o1
    public int flag = 1;

    // 定义两个Object对象，模拟两个线程占有的资源
    public static Object o1 = new Object();
    public static Object o2 = new Object();

    public static void main(String[] args) {

        DeadLock deadLock1 = new DeadLock();
        DeadLock deadLock2 = new DeadLock();

        deadLock1.flag = 0;
        deadLock2.flag = 1;

        Thread thread1 = new Thread(deadLock1);
        Thread thread2 = new Thread(deadLock2);

        thread1.start();
        thread2.start();

    }

    public void run() {

        System.out.println("flag: " + flag);

        // deadLock2占用资源o1，准备获取资源o2
        if (flag == 1) {
            synchronized (o1) {
                try {
                    Thread.sleep(1000);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                synchronized (o2) {
                    System.out.println("1");
                }
            }
        }

        // deadLock1占用资源o2，准备获取资源o1
        else if (flag == 0) {
            synchronized (o2) {
                try {
                    Thread.sleep(1000);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                synchronized (o1) {
                    System.out.println("0");
                }
            }
        }
    }
}

```

## 解决死锁的方法
解决死锁的方法可以从多个角度去分析，一般的情况下，有预防，避免，检测和解除四种。
预防 是采用某种策略，限制并发进程对资源的请求，从而使得死锁的必要条件在系统执行的任何时间上都不满足。

避免则是系统在分配资源时，根据资源的使用情况提前做出预测，从而避免死锁的发生检测是指系统设有专门的机构，当死锁发生时，该机构能够检测死锁的发生，并精确地确定与死锁有关的进程和资源。

解除 是与检测相配套的一种措施，用于将进程从死锁状态下解脱出来。

# 死锁的预防

死锁四大必要条件上面都已经列出来了，很显然，只要破坏四个必要条件中的任何一个就能够预防死锁的发生。

破坏第一个条件 互斥条件：使得资源是可以同时访问的，这是种简单的方法，磁盘就可以用这种方法管理，但是我们要知道，有很多资源 往往是不能同时访问的 ，所以这种做法在大多数的场合是行不通的。

破坏第三个条件 非抢占 ：也就是说可以采用 剥夺式调度算法，但剥夺式调度方法目前一般仅适用于 主存资源 和 处理器资源 的分配，并不适用于所有的资源，会导致 资源利用率下降。

所以一般比较实用的 预防死锁的方法，是通过考虑破坏第二个条件和第四个条件。

1、静态分配策略静态分配策略可以破坏死锁产生的第二个条件（占有并等待）。所谓静态分配策略，就是指一个进程必须在执行前就申请到它所需要的全部资源，并且知道它所要的资源都得到满足之后才开始执行。**进程要么占有所有的资源然后开始执行，要么不占有资源，不会出现占有一些资源等待一些资源的情况。** 静态分配策略逻辑简单，实现也很容易，但这种策略 严重地降低了资源利用率，因为在每个进程所占有的资源中，**有些资源是在比较靠后的执行时间里采用的，甚至有些资源是在额外的情况下才是用的，**这样就可能造成了一个进程占有了一些 几乎不用的资源而使其他需要该资源的进程产生等待 的情况。

2、层次分配策略层次分配策略破坏了产生死锁的第四个条件(循环等待)。在层次分配策略下，所有的资源被分成了多个层次，一个进程得到某一次的一个资源后，它只能再申请较高一层的资源；当一个进程要释放某层的一个资源时，必须先释放所占用的较高层的资源，按这种策略，是不可能出现循环等待链的，因为那样的话，就出现了已经申请了较高层的资源，反而去申请了较低层的资源，不符合层次分配策略，证明略。