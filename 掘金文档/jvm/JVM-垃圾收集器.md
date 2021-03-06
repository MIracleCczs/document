![垃圾收集器](/Users/miracle/miracle/chrome/垃圾收集器.png)

如果说收集算法是内存回收的方法论，那么垃圾收集器就是内存回收的具体实现。虽然我们对各个收集器进行比较，但并非为了挑选出一个最好的收集器。因为直到现在为止还没有最好的垃圾收集器出现，更加没有万能的垃圾收集器，我们能做的就是根据具体应用场景选择适合自己的垃圾收集器。试想一下：如果有一种四海之内、任何场景下都适用的完美收集器存在，那么我们的Java虚拟机就不会实现那么多不同的垃圾收集器了。

### Serial收集器（-XX:+UseSerialGC-XX:+UseSerialOldGC）

Serial（串行）收集器是最基本、历史最悠久的垃圾收集器了。大家看名字就知道这个收集器是一个单线程收集器了。它的“单线程”的意义不仅仅意味着它只会使用一条垃圾收集线程去完成垃圾收集工作，更重要的是它在进行垃圾收集工作的时候必须暂停其他所有的工作线程（**Stop The World**），直到它收集结束。**新生代采用复制算法，老年代采用标记-整理算法。**

![Serial垃圾收集器](/Users/miracle/miracle/githubdoc/document/掘金文档/jvm/Serial垃圾收集器.png)

虚拟机的设计者们当然知道Stop The World带来的不良用户体验，所以在后续的垃圾收集器设计中停顿时间在不断缩短（仍然还有停顿，寻找最优秀的垃圾收集器的过程仍然在继续）。但是Serial收集器有没有优于其他垃圾收集器的地方呢？当然有，它简单而高效（与其他收集器的单线程相比）。Serial收集器由于没有线程交互的开销，自然可以获得很高的单线程收集效率。SerialOld收集器是Serial收集器的老年代版本，它同样是一个单线程收集器。它主要有两大用途：一种用途是在JDK1.5以及以前的版本中与ParallelScavenge收集器搭配使用，另一种用途是作为CMS收集器的后备方案。

### ParNew收集器(-XX:+UseParNewGC)

ParNew收集器其实就是Serial收集器的多线程版本，除了使用多线程进行垃圾收集外，其余行为（控制参数、收集算法、回收策略等等）和Serial收集器完全一样。默认的收集线程数跟cpu核数相同，当然也可以用参数(-XX:ParallelGCThreads)指定收集线程数，但是一般不推荐修改。**新生代采用复制算法，老年代采用标记-整理算法。**

![ParNew](/Users/miracle/miracle/githubdoc/document/掘金文档/jvm/ParNew.png)

它是许多运行在Server模式下的虚拟机的首要选择，除了Serial收集器外，只有它能与CMS收集器配合工作。

### ParallelScavenge收集器(-XX:+UseParallelGC,-XX:+UseParallelOldGC)

ParallelScavenge收集器类似于ParNew收集器，是Server模式下的默认收集。

**那么它有什么特别之处呢**？

ParallelScavenge收集器关注点是吞吐量。所谓**吞吐量**就是**CPU中用于运行用户代码的时间与CPU总消耗时间的比值**。ParallelScavenge收集器提供了很多参数供用户找到最合适的停顿时间或最大吞吐量，如果对于收集器运作不太了解的话，可以选择把内存管理优化交给虚拟机去完成也是一个不错的选择。**新生代采用复制算法，老年代采用标记-整理算法。**

ParallelOld收集器是ParallelScavenge收集器的老年代版本。使用多线程和“标记-整理”算法。在注重吞吐量以及CPU资源的场合，都可以优先考虑ParallelScavenge收集器和ParallelOld收集器。

![Parallel Old](/Users/miracle/miracle/githubdoc/document/掘金文档/jvm/Parallel Old.png)

### CMS收集器(-XX:+UseConcMarkSweepGC(old))

CMS（ConcurrentMarkSweep）收集器是一种以**获取最短回收停顿时间为目标的收集器**。它非常符合在注重用户体验的应用上使用，它是HotSpot虚拟机第一款真正意义上的并发收集器，它第一次实现了让垃圾收集线程与用户线程（基本上）同时工作。

从名字中的Mark Sweep这两个词可以看出，CMS收集器是一种“**标记-清除**”算法实现的，它的运作过程相比于前面几种垃圾收集器来说更加复杂一些。

**整个过程分为四个步骤**：

1. **初始标记**：暂停所有的其他线程，并记录下GC Roots直接能引用的对象，速度很快；
2. **并发标记**：同时开启GC和用户线程，用一个闭包结构去记录可达对象。但在这个阶段结束，这个闭包结构并不能保证包含当前所有的可达对象。因为用户线程可能会不断的更新引用域，所以GC线程无法保证可达性分析的实时性。所以这个算法里会跟踪记录这些发生引用更新的地方。
3. **重新标记**：重新标记阶段就是为了修正并发标记期间因为用户程序继续运行而导致标记产生变动的那一部分对象的标记记录，这个阶段的停顿时间一般会比初始标记阶段的时间稍长，远远比并发标记阶段时间短。
4. **并发清理**：开启用户线程，同时GC线程开始对未标记的区域做清扫。

![CMS](/Users/miracle/miracle/githubdoc/document/掘金文档/jvm/CMS.png)

**有如下几个缺点**：

- CMS收集器对处理器非常敏感（会和服务抢资源）。
- 无法处理浮动垃圾。
- 使用标记清除算法会产生大量内存碎片，可使用-XX: +UseCMS-CompactAtFullCollection开关参数开启内存碎片整理。

**CMS的相关参数**：

- -XX:+UseConcMarkSweepGC：启用cms 
- -XX:ConcGCThreads：并发的GC线程数 
- -XX:+UseCMSCompactAtFullCollection：FullGC之后做压缩整理（减少碎片）
- -XX:CMSFullGCsBeforeCompaction：多少次FullGC之后压缩一次，默认是0，代表每次 FullGC后都会压缩一次 
- -XX:CMSInitiatingOccupancyFraction: 当老年代使用达到该比例时会触发FullGC（默认 是92，这是百分比） 
-  -XX:+UseCMSInitiatingOccupancyOnly：只使用设定的回收阈值(XX:CMSInitiatingOccupancyFraction设定的值)，如果不指定，JVM仅在第一次使用设定 值，后续则会自动调整 
-  -XX:+CMSScavengeBeforeRemark：在CMS GC前启动一次minor gc，目的在于减少 老年代对年轻代的引用，降低CMS GC的标记阶段时的开销，一般CMS的GC耗时 80%都在 remark阶段

### G1(-XX:+UseG1GC)

G1收集器是一款面向服务器的垃圾收集器,主要针对配备多颗处理器及大容量内存的机器. 以极高概率满足GC停顿时间要求的同时,还具备高吞吐量性能特征。

G1将Java堆划分为多个大小相等的独立区域（Region），JVM最多可以有2048个Region。 G1保留了年轻代和老年代的概念，但不再是物理隔阂了，它们都是（可以不连续）Region的集合。一个Region可能之前是年轻代，如果Region进行了垃圾回收，之后可能又会变成老年代，也就是 说Region的区域功能可能会动态变化。

G1垃圾收集器对于对象什么时候会转移到老年代跟之前讲过的原则一样，唯一不同的是对大对象 的处理，G1有专门分配大对象的Region叫Humongous区，而不是让大对象直接进入老年代的 Region中。

![G1垃圾收集器（Garbage First）](/Users/miracle/miracle/githubdoc/document/掘金文档/jvm/G1垃圾收集器（Garbage First）.png)

**包含如下几个步骤**：

1. **初始标记**：标记一下GC Roots能直接关联到的对象。
2. **并发标记**：从GC Root开始对堆中对象进行可达性分析，递归扫描整个堆里的对象图。
3. **最终标记**：对用户线程做一个短暂的暂停，用于处理并发阶段结束后遗留的SATB记录。
4. **筛选回收**：更新Region统计数据，根据用户期望的停顿时间制定回收计划，回收算法主要使用复制算法。

![G1](/Users/miracle/miracle/githubdoc/document/掘金文档/jvm/G1.png)

**特点**：

- **并行与并发**：G1能充分利用CPU、多核环境下的硬件优势，使用多个CPU来缩短Stop-The-World停顿时间。
- **空间整合**：与CMS的“标记--清理”算法不同，G1从整体来看是基于“标记整理”算法 实现的收集器；从局部上来看是基于“复制”算法实现的。
- **可预测的停顿**：通过-XX:MaxGCPauseMillis参数指定在M毫秒内完成垃圾收集（G1收集器在后台维护了一个优先列表，每次根据允许的收集时间，优先选择回收价值最大的 Region）。

### ZGC（JDK11）

目标是对吞吐量影响不大的前提下，实现在任意堆内存大小下都可以把垃圾收集的停顿时间限制在10ms以内的低延迟。有如下特点：

- 基于**动态Region内存布局**。ZGC的Region具有动态性--动态的创建和销毁。在x64下，可以分为大中小三类：

  1. 小型：容量固定为2MB，用于放置小于256KB的小对象；
  2. 中型：容量固定为32MB，用于放置大于等于256KB但小于4MB的对象；
  3. 大型：容量不固定，可以动态变化，但是必须是2MB的整数倍，用于放置4MB或以上的大对象。每个大型Region只会存放一个大对象。

- 不设分代。

- 使用**染色指针**、**读屏障**和**内存多重映射**技术实现可并发的标记-整理算法。

  染色指针是ZGC的标志性设计，它是一种直接将少量额外的信息存储在指针上的技术（高4位提取出来存储4个标志信息）。通过这些标志位，虚拟机可以直接从指针中看到其引用对象的三色标记状态、是否进入了重分配集（被移动过）、是否只能通过finalize()方法才能被访问到。

  **染色指针的三大优势**：

  1. 染色指针可以使得一旦某个Region的存活对象被移走之后，这个Region立即就能被释放和重用掉，而不必等待整个堆中所有指向该Region的引用都被修正后才能清理；
  2. 染色指针可以大幅减少在垃圾收集过程中内存屏障的使用数量，设置内存屏障，尤其是写屏障的目的是为了记录对象引用的变动情况，如果将这些信息直接维护在指针中，显然可以省去一些专门的记录操作；
  3. 染色指针可以作为一种可扩展的存储结构用来记录更多与对象标记、重定位过程相关的数据，以便日后进一步提高性能。

**执行步骤如下**：

1. **并发标记**：与G1、Shenandoah一样，遍历对象图做可达性分析，前后也要经过类似于G1、Shenandoah的初始标记和最终标记的短暂停顿。
2. **并发预备重分配**：根据特定的查询条件统计得出本次收集过程要清理哪些Region，将这些Region组成重分配集。
3. **并发重分配**：将重分配集中的存活对象复制到新的Region上，并为重分配集中的每个Region维护一个转发表，记录从旧对象转到新对象的转向关系。
4. **并发重映射**：修正整个堆中指向重分配集中旧对象的所有引用。


### Shenandoah (JDK 12)

目标是实现一种能在任何堆内存大小下都可以把垃圾收集停顿时间限制在10ms以内的垃圾收集器。

Shenandoah很像G1，两者有相似的内存布局，在初始标记、并发标记等许多阶段的处理思路上都高度一致，甚至共享了一部分实现代码，**相比于G1收集器做了如下改进**：

- 支持并发的整理算法。
- **默认不使用分代收集**（并不是说分代对于Shenandoah没有价值，更多的是处于性价比的权衡）。
- 摒弃G1中耗费大量内存和计算资源维护的记忆集，改用名为“**连接矩阵**”的全局数据结构来记录跨Region的引用关系（连接矩阵可以简单理解为一张二维表格，如果Region N有对象指向Region M,就在表格的N行M列打上标记。）。

**执行步骤如下**：

1. **初始标记**：与G1一样，首先标记与GC Roots直接关联的对象。
2. **并发标记**：与G1一样，遍历对象图，标记出全部可达的对象。
3. **最终标记**：与G1一样，处理剩余的SATB扫描。
4. **并发清理**：这个阶段用于清理那些整个区域内连一个存活对象都没有找到的Region。
5. **并发回收**：与之前HotSpot其他收集器的核心差异，通过读屏障和被称为“Brooks Pointers”的转发指针来解决复制过程中对象移动的问题。
6. **初始引用更新**：建立线程集合点，确保所有并发回收阶段中进行的收集器线程都完成对象移动任务。
7. **最终引用更新**：修正存在于GC Roots中的引用。
8. **并发清理**：回收内存空间。经过并发回收和引用更新之后，整个回收集中所有Region再无存活对象，这些Region都变成Immediate Carbage Regions了，最后再调用一次并发清理过程来回收这些Region的内存空间

### 如何选用

垃圾收集器搭配关系如下，官方推荐使用G1，因为性能较高。

![JVM垃圾收集器](/Users/miracle/miracle/githubdoc/document/掘金文档/jvm/JVM垃圾收集器.png)

