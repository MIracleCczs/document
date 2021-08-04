### 无锁

就人的性格而言，我们可以分为乐天派和悲观派。对于乐观派来说，总是会把事情往好的方面想。他们认为所有所有事情总是不太容易发生问题，出错是小概率的，所以我们可以肆无忌惮地做事。如果真的不幸遇到了问题，则是有则改之无则加勉。对于悲观的人群来书，他们总是担惊受怕，认为出错是一种常态，所以无论巨细，都考虑得面面俱到，滴水不漏，确保万无一失。

对于并发控制而言，锁是一种悲观的策略。它总是假设每一次的临界区操作会产生冲突，因此，必须对每次操作都小心翼翼。如果有多个线程同时需要访问临界区资源，就宁可牺牲性能让线程进行等待，所以说锁会阻塞线程执行。而无锁是一种乐观的策略，它会假设对资源的访问时没有冲突的。既然没有冲突，自然不需要等待，所以所有的线程都可以在不停顿的状态下执行。无锁的策略使用CAS比较交换技术来鉴别线程冲突，一旦检测到冲突产生，就重试当前操作直到没有冲突为止。

### 与众不同的并发策略：比较交换（CAS）

CAS算法的过程：它包含三个参数CAS(V,E,N)。V表示要更新的变量，E表示预期值，N表示新值。仅当V值等于E值时，才会将V的值设置为N，如果V值和E值不同，则说明已经有其他线程做了更新，则当前线程什么都不做。

![CAS](F:\miracle\java going\并发\CAS与Atomic包\CAS.png)



### Atomic

Java的`java.util.concurrent`包除了提供底层锁、并发集合外，还提供了一组原子操作的封装类，它们位于`java.util.concurrent.atomic`包。

以`AtomicInteger`为例，它提供的主要操作有：

- 增加值并返回新值：`int addAndGet(int delta)`
- 加1后返回新值：`int incrementAndGet()`
- 获取当前值：`int get()`
- 用CAS方式设置：`int compareAndSet(int expect, int update)`

Atomic类是通过无锁（lock-free）的方式实现的线程安全（thread-safe）访问。它的主要原理是利用了CAS：Compare and Set。

```java 
public final int incrementAndGet() {
    return unsafe.getAndAddInt(this, valueOffset, 1) + 1;
}
```

```java
public final int getAndAddInt(Object var1, long var2, int var4) {
    int var5;
    do {
        var5 = this.getIntVolatile(var1, var2);
    } while(!this.compareAndSwapInt(var1, var2, var5, var5 + var4));

    return var5;
}
```

以`incrementAndGet`为例，该方法实际上是调用了unsafe实例的getAndAddInt方法，Unsafer类提供了一些底层操作，atomic包下的原子操作类的也主要是通过Unsafe类提供的compareAndSwapInt，compareAndSwapLong等一系列提供CAS操作的方法来进行实现。

### 示例：i++

```java
public class AtomicityDemo {
    private static AtomicInteger atomicInteger = new AtomicInteger(0);
    public static void main(String[] args) throws InterruptedException {

        List<Thread> list = new ArrayList<>();
        for (int i = 0; i < 5; i++) {
            Thread t = new Thread(() -> {
                for (int j = 0; j < 100000; j++) {
                    atomicInteger.incrementAndGet();
                }
            });
            t.start();
            list.add(t);
        }

        for (Thread t : list) {
            t.join();
        }

        System.out.println(atomicInteger.get());
    }
}
```

### 总结

使用`java.util.concurrent.atomic`提供的原子操作可以简化多线程编程：

- 原子操作实现了无锁的线程安全；
- 适用于计数器，累加器等。

### CAS的原子性
