在生产中常见的问题如下：

- CPU飙升导致系统不可用或tps降低；
- Minor GC次数过多；
- Full GC次数过多；
- Full GC时间过长；
- 内存泄漏、内存溢出；
- 死锁等；

恰当地使用虚拟机故障处理、分析的工具可以提升我们分析数据、定位并解决问题的效率。

### jps

查看虚拟机进程。

![image-20210814104652012](/Users/miracle/miracle/githubdoc/document/掘金文档/jvm/jps.png)

### jmp

用于查看内存信息和生成堆存储快照（一般称为headdump或dump文件）。

#### 查看实例数和占用内存大小

使用`jmap -histo 1695 > ./jmp.txt`打印演示所用程序内存信息到jmp.txt文件中，部分截图如下：

![image-20210814105204821](/Users/miracle/miracle/githubdoc/document/掘金文档/jvm/jmp-histo.png)

- num：序号
- Instances：实例数量
- bytes：占用空间大小
- class name：类名称（**[C** is a char[]，**[S**  is a short[]，**[I** is a int[]，**[B** is a byte[]，**[[I** is a **`int[][]`**）

#### 查看堆信息

使用`jhsdb jmap --heap --pid 1695 > jmap_heap.txt`查看堆信息，打印到jmap_heap.txt文件中，部分截图如下所示：

![image-20210814120932382](/Users/miracle/miracle/githubdoc/document/掘金文档/jvm/堆内存信息.png)

首先是`Heap Configuration`，即堆配置信息：

- MinHeapFreeRatio：JVM堆最小空闲比率。对应JVM参数为`-XX:MinHeapFreeRatio`。
- MaxHeapFreeRatio：JVM堆最小空闲比率。对应JVM参数为`-XX:MaxHeapFreeRatio`。
- MaxHeapSize：堆的最大大小。对应JVM参数为`-XX:MaxHeapSize`。
- NewSize：堆的‘新生代’的默认大小。对应JVM参数为`-XX:NewSize`。
- MaxNewSize：堆的‘新生代’的最大大小。对应JVM参数为`-XX:MaxNewSize`。
- OldSize：堆的‘老生代’的大小。对应JVM参数为`-XX:OldSize`。
- NewRatio：‘新生代’和‘老生代’的大小比率。对应JVM参数为`-XX:NewRatio`。
- SurvivorRatio：年轻代中Eden区与Survivor区的大小比值。对应JVM参数为`-XX:SurvivorRatio`。
- SurvivorRatio：年轻代中Eden区与Survivor区的大小比值。对应JVM参数为`-XX:SurvivorRatio`。
- MetaspaceSize：元空间的默认大小。对应JVM参数为`-XX:MetaspaceSize`。
- CompressedClassSpaceSize：类指针压缩空间的默认大小。对应JVM参数为`-XX:CompressedClassSpaceSize`。
- MaxMetaspaceSize：元空间的最大大小。对应JVM参数为`-XX:MaxMetaspaceSize`。
- G1HeapRegionSize：使用G1垃圾收集器的时候，堆被分割的大小（即每个region的大小）。对应JVM参数为`-XX:G1HeapRegionSize`。

接着就是`Heap Usage`，即堆使用情况，包括G1收集器的内存划分，新生代区域、Eden区和Survivor区以及老年代区域分配使用情况等。

#### 导出dump文件

使用`jmap -dump:format=b,file=javagoing.hprof 1695`导出dump文件。

可以使用jvisualvm工具来分析这个dump文件。

![image-20210814123946301](/Users/miracle/miracle/githubdoc/document/掘金文档/jvm/jvisualvm分析dump.png)

### jstack

Java堆栈跟踪工具。用于生成虚拟机当前时刻的线程快照（一般称为threaddump或javacore）。线程快照就是当前虚拟机内每一条正在执行的方法堆栈的集合，生成线程快照的目的通常是定位线程出现长时间停顿的原因，如线程间死锁、死循环、请求外部资源导致的长时间挂起等。

#### 死锁检测

**一个简单的死锁例子**：

```java
public class DeadLock {

    private static Object lock1 = new Object();
    private static Object lock2 = new Object();

    public static void main(String[] args) {
        new Thread(() -> {
            synchronized (lock1) {
                System.out.println("thread1 begin");
                try {
                    Thread.sleep(5000);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                synchronized (lock2) {
                    System.out.println("thread1 end");
                }
            }

        }).start();


        new Thread(() -> {
            synchronized (lock2) {
                System.out.println("thread2 begin");
                try {
                    Thread.sleep(5000);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                synchronized (lock1) {
                    System.out.println("thread2 end");
                }
            }

        }).start();
    }
}
```

启动程序，发现进入死锁状态，使用jps命令查看进程id，然后使用jstack+进程id即可查看死锁信息。如`jstack 2862>deadlock.txt`，生成如下信息：

![image-20210814125659099](/Users/miracle/miracle/githubdoc/document/掘金文档/jvm/死锁jstack.png)

还可以使用`jvisualvm`自动检测死锁：

![image-20210814125930235](/Users/miracle/miracle/githubdoc/document/掘金文档/jvm/jvisualvm死锁检测.png)

#### 找出CPU最高的堆栈信息

1. 使用命令top-p<pid>，显示你的java进程的内存情况，pid是你的java进程号，比如2862。
2. 按H，获取每个线程的内存情况。
3. 找到内存和cpu占用最高的线程tid，比如2862。
4. 转为十六进制得到b2e,此为线程id的十六进制表示。
5. 执行jstack4977|grep-b2e，得到线程堆栈信息中1371这个线程所在行的后面10行。
6. 查看对应的堆栈信息找出可能存在问题的代码。

### jnfo

Java配置信息工具。查看正在运行的Java程序的扩展参数。

#### 查看jvm参数

使用`jinfo -flags pid`，如下所示：

![image-20210814130752831](/Users/miracle/miracle/githubdoc/document/掘金文档/jvm/jinfo查看jvm参数.png)

#### 查看Java系统参数

使用`jinfo -sysprops pid`命令。

### jstat

虚拟机信息监视工具。用于监视虚拟机各种运行状态信息，可以显示本地或者远程虚拟机进程中的类加载、内存、垃圾收集、即时编译等运行时数据。

#### 垃圾回收统计

使用`jstat -gc pid`命令可以查看gc信息，用于评估程序内存使用及GC压力整体情况。

![image-20210814131251380](/Users/miracle/miracle/githubdoc/document/掘金文档/jvm/jstat垃圾回收统计.png)

- S0C：第一个幸存区的大小
- S1C：第二个幸存区的大小
- S0U：第一个幸存区的使用大小
- S1U：第二个幸存区的使用大小
- EC：伊甸园区的大小
- EU：伊甸园区的使用大小
- OC：老年代大小
- OU：老年代使用大小
- MC：方法区大小(元空间)
- MU：方法区使用大小
- CCSC:压缩类空间大小
- CCSU:压缩类空间使用大小
- YGC：年轻代垃圾回收次数
- YGCT：年轻代垃圾回收消耗时间，单位s
- FGC：老年代垃圾回收次数
- FGCT：老年代垃圾回收消耗时间，单位s
- GCT：垃圾回收消耗总时间，单位s

#### 堆内存统计

使用`jstat -gccapacity pid`查看堆内存统计。

![image-20210814131612010](/Users/miracle/miracle/githubdoc/document/掘金文档/jvm/jstat堆内存统计.png)

- NGCMN：新生代最小容量
- NGCMX：新生代最大容量
- NGC：当前新生代容量
- S0C：第一个幸存区大小
- S1C：第二个幸存区的大小
- EC：伊甸园区的大小
- OGCMN：老年代最小容量
- OGCMX：老年代最大容量
- OGC：当前老年代大小
- OC:当前老年代大小
- MCMN:最小元数据容量
- MCMX：最大元数据容量
- MC：当前元数据空间大小
- CCSMN：最小压缩类空间大小
- CCSMX：最大压缩类空间大小
- CCSC：当前压缩类空间大小
- YGC：年轻代gc次数
- FGC：老年代
- GC次数

#### 新生代垃圾回收统计

`jstat -gcnew pid`

![image-20210814131827112](/Users/miracle/miracle/githubdoc/document/掘金文档/jvm/jstat-新生代垃圾回收统计.png)

- S0C：第一个幸存区的大小
- S1C：第二个幸存区的大小
- S0U：第一个幸存区的使用大小
- S1U：第二个幸存区的使用大小
- TT:对象在新生代存活的次数
- MTT:对象在新生代存活的最大次数
- DSS:期望的幸存区大小
- EC：伊甸园区的大小
- EU：伊甸园区的使用大小
- YGC：年轻代垃圾回收次数
- YGCT：年轻代垃圾回收消耗时间

#### 新生代内存统计

`jstat -gcnewcapacity pid`

![image-20210814132031274](/Users/miracle/miracle/githubdoc/document/掘金文档/jvm/jstat新生代内存统计.png)

- NGCMN：新生代最小容量
- NGCMX：新生代最大容量
- NGC：当前新生代容量
- S0CMX：最大幸存1区大小
- S0C：当前幸存1区大小
- S1CMX：最大幸存2区大小
- S1C：当前幸存2区大小
- ECMX：最大伊甸园区大小
- EC：当前伊甸园区大小
- YGC：年轻代垃圾回收次数
- FGC：老年代回收次数

还可以查看老年代垃圾回收、内存统计以及元空间内存统计等，就不再演示了。有了这些数据就可以采用之前介绍过的优化思路，先给自己的系统设置一些初始性的JVM参数，比如堆内存大小，年轻代大小，Eden和Survivor的比例，老年代的大小，大对象的阈值，大龄对象进入老年代的阈值等。

### 总结

以上介绍了JVM提供的几个重要的故障处理工具，希望能够给大家解决问题提供一些帮助。
