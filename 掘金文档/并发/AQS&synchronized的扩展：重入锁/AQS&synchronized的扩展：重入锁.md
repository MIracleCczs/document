### synchronized的扩展：重入锁

#### 简单应用

> java并发包中提供了ReentrantLock类用以提供对synchronized的扩展，ReentrantLock完全可以替代synchronized。

```java
static final ReentrantLock lock = new ReentrantLock();
static int i = 0;
@Override
public void run() {
    lock.lock();
    try {
        for (int j = 0; j < 10000; j++) {
            i++;
        }
    } finally {
        lock.unlock();
    }
}
```

从上面的示例代码可以看出，相比于synchronized，ReentrantLock需要自己执行何时加锁，何时释放锁。

#### 何为重入

```java
lock.lock();
lock.lock();
try {
    for (int j = 0; j < 10000; j++) {
        i++;
    }
} finally {
    lock.unlock();
    lock.unlock();
}
```

ReentrantLock支持连续多次获取同一把锁。这对一些特定场景提供了更灵活的加锁方式。

#### 重要的扩展

1. 能够响应中断。synchronized 的问题是，持有锁 A 后，如果尝试获取锁 B 失败，那么线程就进入阻塞状态，一旦发生死锁，就没有任何机会来唤醒阻塞的线程。但如果阻塞状态的线程能够响应中断信号，也就是说当我们给阻塞的线程发送中断信号的时候，能够唤醒它，那它就有机会释放曾经持有的锁 A。

2. 支持超时。如果线程在一段时间之内没有获取到锁，不是进入阻塞状态，而是返回一个错误，那这个线程也有机会释放曾经持有的锁。

3. 非阻塞地获取锁。如果尝试获取锁失败，并不进入阻塞状态，而是直接返回，那这个线程也有机会释放曾经持有的锁。

   > 响应的提供了三类API

   ```java
   // 支持中断的API
   void lockInterruptibly() throws InterruptedException;
   // 支持超时的API
   boolean tryLock(long time, TimeUnit unit) throws InterruptedException;
   // 支持非阻塞获取锁的API
   boolean tryLock();
   ```

这三个扩展可以破坏不可抢占条件，在一些死锁场景中可以很轻易的解决。

```java
示例一：
static Lock lock1 = new ReentrantLock();
static Lock lock2 = new ReentrantLock();
public static void main(String[] args) throws InterruptedException {

    Thread t1 = new Thread(new ThreadDemo(lock1, lock2));//该线程先获取锁1,再获取锁2
    Thread t2 = new Thread(new ThreadDemo(lock2, lock1));//该线程先获取锁2,再获取锁1
    thread.start();
    thread1.start();
    thread.interrupt();//是第一个线程中断
}

static class ThreadDemo implements Runnable {
    Lock firstLock;
    Lock secondLock;
    public ThreadDemo(Lock firstLock, Lock secondLock) {
        this.firstLock = firstLock;
        this.secondLock = secondLock;
    }
    @Override
    public void run() {
        try {
            firstLock.lockInterruptibly();
            TimeUnit.MILLISECONDS.sleep(10);//更好的触发死锁
            secondLock.lockInterruptibly();
        } catch (InterruptedException e) {
            e.printStackTrace();
        } finally {
            firstLock.unlock();
            secondLock.unlock();
            System.out.println(Thread.currentThread().getName()+"正常结束!");
        }
    }
}

```

线程t1和t2启动后，t1先占用lock1，再占用lock2;t2先占用lock2,再占用lock1。因此，t1和t2很容易形成相互等待，发生死锁。使用```lockInterruptibly()```方法可以对中断进行响应，线程接收到中断信号后，释放自己获得的锁。

```java
示例二：

static ReentrantLock lock = new ReentrantLock();

@Override
public void run() {
    try {
        if (lock.tryLock(5, TimeUnit.SECONDS)) {
            Thread.sleep(6000);
        } else {
            System.out.println("get lock failed");
        }
    } catch (InterruptedException e) {
        e.printStackTrace();
    } finally {
        if (lock.isHeldByCurrentThread()) {
            lock.unlock();
        }
    }
}
```

```boolean tryLock(long time, TimeUnit unit)```接收两个参数，一个表示等待时长，一个表示单位，这里设置为5秒，表示在这个锁请求中最多等待5秒，超过5秒就会返回false。

```tryLock()```使用方式类似，当前线程会尝试获取锁，如果锁被其他线程占用，则立刻返回false。

#### 公平锁

重入锁允许对公平性进行设置。它有如下的构造函数：

```java
public ReentrantLock(boolean fair) {
    sync = fair ? new FairSync() : new NonfairSync();
}
```

当fair为true时，表示锁是公平的。公平锁看起来很优美，但是要实现公平锁必然对性能有些影响。如果没有特殊的需要，就不要用公平锁。

```java
public static ReentrantLock lock = new ReentrantLock(true);

@Override
public void run() {
    while (true) {
        try {
            lock.lock();
            System.out.println(Thread.currentThread().getName() + "获得锁");
        } finally {
            lock.unlock();
        }

    }
}
执行结果：
Thread-0获得锁
Thread-1获得锁
Thread-0获得锁
Thread-1获得锁
...
```

#### Condition条件

实现阻塞队列：

```java
public class BlockQueue<T> {

    private int size;
    final List<T> elements = new ArrayList<>();
    
    final Lock lock = new ReentrantLock();
    // 条件变量：队列不满
    final Condition notFull = lock.newCondition();
    // 条件变量：队列不空
    final Condition notEmpty = lock.newCondition();

    public BlockQueue() {
        this.size = Integer.MAX_VALUE;
    }

    public BlockQueue(int size) {
        this.size = size;
    }

    // 入队
    void enq(T x) throws InterruptedException {
        lock.lock();
        try {
            while (elements.size() >= size){
                // 等待队列不满
                notFull.await();
            }
            elements.add(x);
            // 入队后, 通知可出队
            notEmpty.signal();
        }finally {
            lock.unlock();
        }
    }
    // 出队
    T deq() throws InterruptedException {
        lock.lock();
        try {
            while (CollectionUtils.isEmpty(elements)){
                // 等待队列不空
                notEmpty.await();
            }
            T t = elements.remove(0);
            // 出队后，通知可入队
            notFull.signal();
            return t;
        }finally {
            lock.unlock();
        }
    }
}
```

线程等待和通知需要调用 await()、signal()、signalAll()，它们的语义和 wait()、notify()、notifyAll() 是相同的, 但是不要相互使用。

### ReentrantLock实现以及AQS源码阅读

`AbstractQueuedSynchronizer`这个类是分析Java并发包避不开的话题，它是实现并发包的基础工具类，是`ReentrantLock`、`CountDownLatch`、`Semaphore`、`FutureTask`等类的基础。

今天将从ReentrantLock这个类出发，看看AbstractQueuedSynchronizer是怎么工作的。

ReentrantLock内部使用了Sync类来管理锁，其继承自AbstractQueuedSynchronizer。

```java
abstract static class Sync extends AbstractQueuedSynchronizer {
}
```

#### AQS 结构

```java
// 等待队列的头节点，懒加载。只能通过setHead()方法修改
// 如果头节点存在，则waitStatus不能是CANCELLED
private transient volatile Node head;

// 等待队列的尾节点，懒加载。只能通过enq()方法添加
private transient volatile Node tail;

// 锁的状态，0表示未锁定，大于0表示锁定
// 这个值也可以用来实现锁的可重入，每次加锁+1，释放锁-1
private volatile int state;

// 当前持有锁的线程继承自AbstractOwnableSynchronizer
private transient Thread exclusiveOwnerThread;
```

![AQS阻塞队列](F:\miracle\java going\并发\AQS&synchronized的扩展：重入锁\AQS阻塞队列.png)

等待队列中的所有线程实例被包装成一个Node，数据结构是链表。结构如下：

```java
// 状态 取值如下：
// SIGNAL=-1 此节点的后继节点将被唤醒
// CANCELLED=1 由于超时或中断，该节点被取消
// CONDITION=-2 表示该节点在条件队列中
// PROPAGATE=-3 
// 0 初始或者释放
volatile int waitStatus;
// 前驱节点
volatile Node prev;
// 后继节点
volatile Node next;
// 当前线程
volatile Thread thread;
// 下一个condition队列等待节点
Node nextWaiter;
```

以上就是AQS的数据结构，记住这些，我们来看下ReentrantLock内部是如何实现的（以非公平锁为例）。

#### 线程抢锁

```java
// NonfairSync 提供了两个方法，来分析下其内部实现
static final class NonfairSync
 extends Sync {
    private static final long serialVersionUID = 7316153563782823691L;
    // 争锁
    final void lock() {
        // CAS进行枪锁，如果抢锁成功，获取锁返回
        if (compareAndSetState(0, 1))
            setExclusiveOwnerThread(Thread.currentThread());
        else
            // 该方法继承自AbstractQueuedSynchronizer类
            // 判断锁是否释放，如果是直接CAS抢锁
            acquire(1);
    }

    // 非公平方式尝试获取锁
    protected final boolean tryAcquire(int acquires) {
        // 执行Sync的nonfairTryAcquire方法
        return nonfairTryAcquire(acquires);
    }
}

// AbstractQueuedSynchronizer.acquire()
public final void acquire(int arg) {
    // 首先使用tryAcquire(1)试一试看看是否能够成功
    if (!tryAcquire(arg) &&
        // tryAcquire没有成功将线程挂起,放入等待队列中
        acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
        selfInterrupt();
}

// 此时发现tryAcquire又回到了NonfairSync的tryAcquire方法

// Sync.nonfairTryAcquire()
final boolean nonfairTryAcquire(int acquires) {
    final Thread current = Thread.currentThread();
    int c = getState();
    // state=0,当前没有线程持有锁
    if (c == 0) {
        // CAS抢锁,如果不成功说明有人同时抢锁
        if (compareAndSetState(0, acquires)) {
            // 抢到锁,标记当前线程为锁持有者
            setExclusiveOwnerThread(current);
            return true;
        }
    }
    // 会进入这个分支,说明锁重入了,需要state+1
    else if (current == getExclusiveOwnerThread()) {
        int nextc = c + acquires;
        if (nextc < 0) // overflow
            throw new Error("Maximum lock count exceeded");
        setState(nextc);
        return true;
    }
    // 走到这里说明没有获取到锁
    return false;
}

// 假设nonfairTryAcquire返回false
// 就会执行acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
// 先看下addWaiter(Node.EXCLUSIVE)

// AbstractQueuedSynchronizer.addWaiter()
// Node.EXCLUSIVE代表独占模式
private Node addWaiter(Node mode) {
    Node node = new Node(Thread.currentThread(), mode);
    // Try the fast path of enq; backup to full enq on failure
    // 以下代码就是新增节点放到阻塞队列最后
    Node pred = tail;
    // 如果前驱节点不为空
    if (pred != null) {
        // 将当前节点的前驱设置为队尾
        node.prev = pred;
        // 使用CAS将当前节点设置到队尾,成功说明tail=node
        if (compareAndSetTail(pred, node)) {
            pred.next = node;
            return node;
        }
    }
    // 能走到这里说明有两种情况
    // 1.pred==null
    // 2.CAS设置队尾时失败
    // 遇到这两种情况,AQS采用自旋的方式入队
    enq(node);
    return node;
}
// AbstractQueuedSynchronizer.enq()
private Node enq(final Node node) {
    for (;;) {
        Node t = tail;
        // 1.head和tail节点在初始化的时候为null
        if (t == null) { // Must initialize
            if (compareAndSetHead(new Node()))
                tail = head;
        } else {
            // 2.和addWaiter一样,使用CAS将当前节点设置到队尾
            node.prev = t;
            if (compareAndSetTail(t, node)) {
                t.next = node;
                return t;
            }
        }
    }
}

// 到这里,新节点已经添加到阻塞队列了,接下来将执行acquireQueued()方法
// 这个方法会对线程进行挂起和唤醒抢锁操作
// AbstractQueuedSynchronizer.acquireQueued()
final boolean acquireQueued(final Node node, int arg) {
    boolean failed = true;
    try {
        boolean interrupted = false;
        // 还是自旋,直到返回true
        for (;;) {
            // 获取当前节点的前驱,如果是初始化的那么head=tail
            // 这里也可以看出来,head其实不算阻塞队列中的一员,应该算是当前持有锁的节点
            final Node p = node.predecessor();
            // 如果当前节点等于head并且尝试获取锁成功
            if (p == head && tryAcquire(arg)) {
                setHead(node);
                p.next = null; // help GC
                failed = false;
                return interrupted;
            }
            // 到这里有两种情况
            // 1.node不是head
            // 2.抢锁失败
            // 遇到者两种情况,会挂起线程
            if (shouldParkAfterFailedAcquire(p, node) &&
                parkAndCheckInterrupt())
                interrupted = true;
        }
    } finally {
        // 如果失败,可能是出现异常?
        if (failed)
            cancelAcquire(node);
    }
}

// AbstractQueuedSynchronizer.shouldParkAfterFailedAcquire()
private static boolean shouldParkAfterFailedAcquire(Node pred, Node node) {
    int ws = pred.waitStatus;
    // 前驱节点的 waitStatus == -1 ，说明前驱节点状态正常，当前线程需要挂起，直接可以返回true
    if (ws == Node.SIGNAL)
        return true;
    // 前驱节点的 waitStatus>0,说明前驱节点由于超时或中断，该节点被取消
    // 那么要做的就是循环将已取消的前驱节点移除,直到前驱节点的waitStatus<=0为止
    if (ws > 0) {
        do {
            node.prev = pred = pred.prev;
        } while (pred.waitStatus > 0);
        pred.next = node;
    } else {
        // 将前驱节点的值设置为-1,代表有后继节点等待被唤醒
        compareAndSetWaitStatus(pred, ws, Node.SIGNAL);
    }
    return false;
}

// AbstractQueuedSynchronizer.parkAndCheckInterrupt()
// 这个方法很简单,就是挂起线程
private final boolean parkAndCheckInterrupt() {
    LockSupport.park(this);
    return Thread.interrupted();
}

// 到这里整个acquire()方法结束

// 总结一下，非公平锁的线程抢锁分为下列几步：
// 1.直接CAS抢锁，抢锁成功，标记当前锁持有线程，返回
// 2.抢锁失败，执行acquire(1)，这个方法判断锁是否释放，如果已释放，则进行抢锁
// 3.抢锁失败或者锁未释放，将当前线程加入等待队列并挂起
```

以上就是线程抢锁的逻辑了，代码很长，可能需要读者多看几遍。

#### 线程锁释放

```java
// 解锁    
public void unlock() {
    sync.release(1);
}

// AbstractQueuedSynchronizer.unparkSuccessor().release()
public final boolean release(int arg) {
    // 执行解锁操作，如果成功，头节点不为空并且waitStatus!=0, 将唤醒头节点的后继节点
    if (tryRelease(arg)) {
        Node h = head;
        if (h != null && h.waitStatus != 0)
            unparkSuccessor(h);
        return true;
    }
    return false;
}

// Sync.tryRelease()
// 解锁操作，由子类Sync实现
protected final boolean tryRelease(int releases) {
    // state-1
    int c = getState() - releases;
    // 如果当前线程不是锁的持有者抛出异常
    if (Thread.currentThread() != getExclusiveOwnerThread())
        throw new IllegalMonitorStateException();
    boolean free = false;
    // 可重入问题，判断锁是否释放完，释放完设置锁持有线程为null,并且返回true,否则说明
    // 还有嵌套锁，那就不能释放
    if (c == 0) {
        free = true;
        setExclusiveOwnerThread(null);
    }
    setState(c);
    return free;
}

// AbstractQueuedSynchronizer.unparkSuccessor()
// 唤醒头节点的后继节点
private void unparkSuccessor(Node node) {
    int ws = node.waitStatus;
    // 清除头节点状态
    if (ws < 0)
        compareAndSetWaitStatus(node, ws, 0);

    // 判断头节点的后继节点是不是等于null并且没有中断，如果不是则循环直到找到满足条件的后继节点
    Node s = node.next;
    if (s == null || s.waitStatus > 0) {
        s = null;
        for (Node t = tail; t != null && t != node; t = t.prev)
            if (t.waitStatus <= 0)
                s = t;
    }
    // 如果后继节点不为null,唤醒
    if (s != null)
        LockSupport.unpark(s.thread);
}

// 线程解锁的操作要比加锁要简单的多，就不再多说了
```

#### 总结

在并发条件下加锁和解锁主要通过以下来完成：

1. 锁状态：锁状态为0时代表可以锁空闲，可以抢锁，那么AQS执行CAS操作将state设置为1，代表抢到锁。如果是锁重入则state加1，解锁就是state减1，直到state等于0。
2. 线程挂起和线程唤醒：AQS中采用LockSupport.park()和LockSupport.unpark()来操作。
3. 阻塞队列：一个线程获取到锁，则其他线程需要等待，AQS提供了一个FIFO的链表来实现。