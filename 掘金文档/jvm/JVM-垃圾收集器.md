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

