赞一个, 方法很多, 转自 [ "Axb的自我修养"](http://blog.2baxb.me/archives/918#comments)
2.现象
线上机器部署了两个java实例，在运行几天后java开始吃swap空间，java实例的内存占用接近7G，程序响应很慢，重启后又恢复正常。线上配置的堆内存为3600M，栈大小为512k。

3.排查
首先怀疑是java heap的问题，查看heap占用内存，没有什么特殊。
`jmap -heap pid`
然后又怀疑是directbuffer的问题，jdk1.7之后对directbuffer监控的支持变得简单了一些，使用如下脚本：

`
import java.io.File;
import java.util.*;
import java.lang.management.BufferPoolMXBean;
import java.lang.management.ManagementFactory;
import javax.management.MBeanServerConnection;
import javax.management.ObjectName;
import javax.management.remote.*;

import com.sun.tools.attach.VirtualMachine; // Attach API

/**
 * Simple tool to attach to running VM to report buffer pool usage.
 */

public class MonBuffers {
    static final String CONNECTOR_ADDRESS =
          "com.sun.management.jmxremote.localConnectorAddress";

    public static void main(String args[]) throws Exception {
        // attach to target VM to get connector address
        VirtualMachine vm = VirtualMachine.attach(args[0]);
        String connectorAddress = vm.getAgentProperties().getProperty(CONNECTOR_ADDRESS);
        if (connectorAddress == null) {
            // start management agent
            String agent = vm.getSystemProperties().getProperty("java.home") +
                    File.separator + "lib" + File.separator + "management-agent.jar";
            vm.loadAgent(agent);
            connectorAddress = vm.getAgentProperties().getProperty(CONNECTOR_ADDRESS);
            assert connectorAddress != null;
        }

        // connect to agent
        JMXServiceURL url = new JMXServiceURL(connectorAddress);
        JMXConnector c = JMXConnectorFactory.connect(url);
        MBeanServerConnection server = c.getMBeanServerConnection();

        // get the list of pools
        Set<ObjectName> mbeans = server.queryNames(
            new ObjectName("java.nio:type=BufferPool,*"), null);
        List<BufferPoolMXBean> pools = new ArrayList<BufferPoolMXBean>();
        for (ObjectName name: mbeans) {
            BufferPoolMXBean pool = ManagementFactory
                .newPlatformMXBeanProxy(server, name.toString(), BufferPoolMXBean.class);
            pools.add(pool);
        }

        // print headers
        for (BufferPoolMXBean pool: pools)
            System.out.format("         %8s             ", pool.getName());
        System.out.println();
        for (int i=0; i<pools.size(); i++)
            System.out.format("%6s %10s %10s  ",  "Count", "Capacity", "Memory");
        System.out.println();

        // poll and print usage
        for (;;) {
            for (BufferPoolMXBean pool: pools) {
                System.out.format("%6d %10d %10d  ",
                    pool.getCount(), pool.getTotalCapacity(), pool.getMemoryUsed());
            }
            System.out.println();
            Thread.sleep(2000);
        }
    }
}

`

查看java线程的情况，虽然线程数很多，但是内存增长时线程数基本没有什么变化。

$ jstack pid |grep 'java.lang.Thread.State' |wc -l

或者
$ cat /proc/pid/status |grep Thread

对java做了一次heap dump，使用eclipse的MAT查看堆内使用情况，没有发现明显有哪个对象数量有明显异常，heap的大小也只有几百兆。

$ jmap -dump:file=/tmp/heap.bin

发现stack dump里的global jni reference一直在增长，怀疑是jni调用存在内存溢出。

$ jstack pid |grep JNI

查找了jar包里的.so/.h等c文件，发现jruby、jthon等jar包里有jni相关的文件。

$ wtool jarfind *.so .

上网发现确实有不少jruby内存溢出的issue。把这些jar包直接删掉之后观察global jni reference数量还是在涨，内存增长情况也没有改善。

之后突然想到full gc的问题，对增长中的java进程做了一次full gc，global jni reference数量由几千个下降到几十个，但是占用内存还是没有变化，排除掉global reference的可能性。

用pmap查看进程内的内存情况，发现java的heap和stack大小都没啥变化，但是定期会多出来一个64M左右的内存块。
![image](https://user-images.githubusercontent.com/20329409/41631524-ae9f6c70-7467-11e8-9f08-ed5c5a36800a.png)

使用gdb观察内存块里的内容，发现里面有一些接口的返回值、mc的返回值、还有一些类名等等

gdb: dump memory /tmp/memory.bin 0x7f6b38000000 0x7f6b38000000+65535000

$ hexdump -C /tmp/memory.bin
或
$ strings /tmp/memory.bin |less
![image](https://user-images.githubusercontent.com/20329409/41631530-b9a5210a-7467-11e8-8461-62f84e14ffed.png)
上网搜索后发现有人遇到过这个问题，在这个网页里有ibm对64M问题的研究。依照网站上说的办法，把MALLOC_ARENA_MAX参数调成1，发现virtual memory正常了，res也小了1G左右。同时hadoop的issue里也有一些性能方面的测试，发现MALLOC_ARENA_MAX=4的时候性能会提升，但是他们也说不清楚为什么。
修改之后程序启动时的virtual memory明显降低，res也降低到了3.2g： 
![image](https://user-images.githubusercontent.com/20329409/41631546-c634a8b4-7467-11e8-9f1c-53138097a6a5.png)
本来以为到这里应该算是解决了，但是这个程序跑了几天之后内存依然在上涨,只是内存块由很多64M变成了一个2g+的普通native heap。

继续寻找线索，在一些关于MALLOC_ARENA_MAX这个参数的讨论里也发现一些关于glibc的其它参数。比如M_TRIM_THRESHOLD和M_MMAP_THRESHOLD或者MALLOC_MMAP_MAX_，试用之后发现依然没有效果。

试着从glibc的malloc实现上找问题，比如这里和这里，同样没有什么进展。

尝试用strace和ltrace查找malloc调用，发现定期有32k的内存申请，但是无法确定是从哪调用的。

尝试用valgrind查找内存泄露，但是jvm跑在valgrind上几分钟就crash了。

在网上查到了一个关于thread pool用法错误有可能导致内存溢出的问题，可以写一个小程序重现：
`
public static void main(String[] args) throws InterruptedException {
    Logger.getRootLogger().setLevel(org.apache.log4j.Level.ERROR);
    while(true) {
        long max = Runtime.getRuntime().maxMemory();
        long total = Runtime.getRuntime().totalMemory();
        long free= Runtime.getRuntime().freeMemory();
        System.out.println("max:"+max+",total:"+total+",free:"+free);
        for (int i = 0; i < 10000; i++) {
            ThreadPoolExecutor executor = new ThreadPoolExecutor();
        }
        Thread.sleep(1000);
    }
}

`
但是用btrace挂了一天也没有发现有错误的调用，源代码里也没找到类似的用法。

重新用MAT在heap dump里查找是否有native reference，发现finalizer队列里有很多java.util.zip.Deflater的实例，上网搜索发现这个类有可能导致native内存溢出，使用的jesery框架里有这个问题导致gzip异常的issue；用btrace监视发现有大量这个类的构造函数被调用，但是经过几次full gc的观察，每次full gc后finalizer队列里的Deflater数量都会减少到个位数，但是内存依然在上涨；同时排查了线上配置，发现没有开启gzip。

也发现了有人说SunPKCS11有可能导致内存泄露，但是也没发现有相关java对象。

尝试把Xss参数调到256k，运行几天后发现内存维持在5.7g左右，比较稳定，但是从各种角度都无法解释为什么xss调小会影响native heap的大小。

怀疑是JIT的问题，用-Xint或者-XX:-Inline方式启动之后发现内存依然增长。

本来排查到这里已经绝望了，但是最后想到是不是JDK本身有什么bug？

查看jdk的changelog，发现线上使用的1.7-b15的版本比较老，之后有一些对native memory leak的修复。尝试用新的jdk1.7-u71启动应用，内存竟然稳定下来了！

在升级jdk、限制directbuffer大小为256M、调整MALLOC_ARENA_MAX=1后，4倍流量的tcpcopy运行几天后内存占用稳定在5G；只升级了jdk，其它参数不变，运行一天后内存为5.4G，是否上涨还有待观察。对比之前占用6.8G左右，效果还是比较明显的

4. 其它参考资料
Linux Native Memory issues for WebSphere Application Server
vmoptions



> Written with [StackEdit](https://stackedit.io/).
<!--stackedit_data:
eyJoaXN0b3J5IjpbMzA2MjgyNTk3XX0=
-->