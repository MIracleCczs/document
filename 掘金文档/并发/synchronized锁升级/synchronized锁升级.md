#### 偏向锁概念

通过生产实践统计了解到**锁常常不存在多线程竞争，而是总是由一个线程多次获得**。为了让线程获取锁的代价更低就引入了偏向锁的概念。偏向锁就是当一个线程进入同步代码块时，会将线程ID存储在对象头中，而不需要再对此线程进行加锁和解锁操作。

> 使用-XX:-UseBiasedLocking参数来开启和关闭偏向锁优化。（默认开启）

#### 偏向锁的获取和撤销

1. 访问1访问同步代码块，判断当前锁状态，如果是无锁状态，判断是否处于可偏向状态;
2. 如果是可偏向状态，则CAS替换Mark Word线程ID;
3. 如果是已偏向状态，就检查Mark Word中的线程ID是否为当前线程，如果不是则CAS替换Mark Word为当前线程ID;
4. 成功替换Mark Word则开启偏向锁，否则进行偏向锁撤销并暂停原来持有的偏向锁线程。等待原持有偏向锁到达安全点，如果此时原偏向锁线程未处于活动状态或者是已退出同步代码块，则唤醒原持有偏向锁的线程，如果原偏向锁线程处于同步代码块，则将偏向锁升级为轻量级锁。

![无锁状态-_偏向锁-_撤销偏向锁](F:\miracle\java going\并发\无锁状态-_偏向锁-_撤销偏向锁.png)

#### 轻量级锁

轻量级锁使用的是**自旋锁**，即另外一个线程竞争锁资源时，当前线程会在原地循环等待，自旋锁是非常消耗CPU的，所以轻量级锁适用于那些同步代码块执行很快的场景，这样，线程原地等待时间很短就能够获得锁。

在JDK1.6之后，引入了自适应自旋锁，指的是自旋的次数并不是固定不变的，而是根据前一次在同一个锁上自旋的时间以及锁的拥有者的状态来决定。 如果在同一个锁对象上，自旋等待刚刚成功获得过锁，并且持有锁的线程正在运行中，那么虚拟机就会认为这次自旋也是很有可能再次成功，进而它将允许自旋等待持续相对更长的时间。**在自旋超过N次后仍然没有获取到锁，则升级未重量级锁**。

轻量级解锁时，会使用CAS 操作来将 Mark Word 替换回到对象头，如果成功，则表示没有竞争发生。**如果失败，表示当前锁存在竞争，锁就会膨胀成重量级锁**。

![轻量级锁及膨胀过程](F:\miracle\java going\并发\轻量级锁及膨胀过程.png)

### 错误的加锁

> 使用不同的锁保护同一个资源

```java
	private static int i = 0;
    public synchronized int get() {
        return i;
    }
    public static synchronized void add() {
        i += 1;
    }
```

![使用多把锁保护资源](F:\suning\old\1\binfa\使用多把锁保护资源.png)

因此这两个临界区（方法）并没有互斥性，线程A和线程B还是可以并行执行，那add()操作对get()操作也是不可见的，所以还是会导致并发问题。

>使用可变对象加锁

```java
public class WrongLockDemo implements Runnable {
    public static Integer i = 0;
    
    @Override
    public void run() {
        for (int j = 0; j < 100000; j++) {
            synchronized (i) {
                i++;
            }
        }
    }

    public static void main(String[] args) throws InterruptedException {
        Thread t1 = new Thread(new WrongLockDemo());
        Thread t2 = new Thread(new WrongLockDemo());
        t1.start();
        t2.start();
        t1.join();
        t2.join();

        System.out.println(i);
    }
}
```

![发编译](F:\suning\old\1\binfa\发编译.png)

反编译这段代码的run()方法，在执行i++时，真实执行的是Integer.valueOf(i.intValue()+1),查看valueOf源码：

```java
    public static Integer valueOf(int i) {
        if (i >= IntegerCache.low && i <= IntegerCache.high)
            return IntegerCache.cache[i + (-IntegerCache.low)];
        return new Integer(i);
    }
```

默认Integer将-128到127提前实例化放到缓存中，但是超出时更倾向于创建一个Integer实例并将它的引用赋值给i。因此，每次获取到的锁都是不同的Integer对象，从而导致并发问题。

