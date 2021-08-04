### 信号量模型

> 一个计数器，一个等待对列，三个方法

![信号量模型](C:\Users\18073632\Desktop\binfa\信号量模型.png)

1. init：设置计数器的初始值。

2. down：计数器的值减1；如果此时计数器的值小于0，则当前线程被阻塞，否则当前线程可以继续执行。

3. up：计数器的值加1；如果此时计数器的值小于或者等于0，则唤醒等待队列中的一个线程执行，并将其从等待队列中移除。

down和up操作也被称为PV操作（PV原语），对应java并发包下对应acquire()和release()。

### Semaphore 

Semaphore是JUC提供的信号量的封装，它是用来控制同时访问特定资源的线程数量，它通过协调各个线程，以保证合理的使用公共资源。Semaphore 跟锁（synchronized、Lock）有点相似，不同的地方是，锁同一时刻只允许一个线程访问某一资源，而 Semaphore 则可以控制同一时刻多个线程访问某一资源。

它是一种基于计数的信号量。它可以设定一个阈值，基于此，多个线程竞争获取许可信号，做自己的申请后归还，超过阈值后，线程申请许可信号将会被阻塞。Semaphore可以用来构建一些对象池，资源池之类的，比如数据库连接池，我们也可以创建计数为1的Semaphore，将其作为一种类似互斥锁的机制，这也叫二元信号量，表示两种互斥状态。

Semaphore有两个构造函数：

```java
public Semaphore(int permits)
public Semaphore(int permits, boolean fair)
```

permits定义了许可资源的个数，而fair则表示是否支持FIFO的顺序。

### 如何使用信号量

> 假设餐厅有一个位置，现在有两个人就餐

```java
class SemaphoreThread extends Thread {
    final Semaphore semaphore;

    public SemaphoreThread(Semaphore semaphore) {
        this.semaphore = semaphore;
    }

    @Override
    public void run() {
        if (semaphore.availablePermits() <= 0) {
            System.out.println(Thread.currentThread().getName()+ "等位中。。。");
        }
        try {
            semaphore.acquire();
            System.out.println(Thread.currentThread().getName()+ "开始就餐了。。");
            Thread.sleep(1000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        System.out.println(Thread.currentThread().getName() + "吃完了。。");
        semaphore.release();
    }
}
public static void main(String[] args) {
    Semaphore semaphore = new Semaphore(1);
    for (int i = 0; i < 2; i++) {
        new SemaphoreThread( semaphore).start();
    }
}
执行结果：
Thread-0开始就餐了。。
Thread-1等位中。。。
Thread-0吃完了。。
Thread-1开始就餐了。。
Thread-1吃完了。。
```

两个线程代表两个人T1、T2，当它们同时调用` acquire() `的时候，由于`acquire()` 是一个原子操作，所以只能有一个线程（假设 T1）把信号量里的计数器减为 0，另外一个线程（T2）则是将计数器减为 -1。对于线T1，信号量里面的计数器的值是 0，大于等于 0，所以线程 T1 会继续执行；对于线程 T2，信号量里面的计数器的值是 -1，小于 0，按照信号量模型里对 down() 操作的描述，线程 T2 将被阻塞。

当线程 T1 执行` release() `操作，也就是 `up() `操作的时候，信号量里计数器的值是 -1，加 1 之后的值是 0，小于等于 0，按照信号量模型里对 up() 操作的描述，此时等待队列中的 T2 将会被唤醒。

### 对象池实现

```java
public class ObjectPool<T, R> {

    final List<T> objPool;
    final Semaphore sem;

    public ObjectPool(int size, T t) {
        this.objPool = new Vector<>(size);
        for (int i = 0; i < size; i++) {
            this.objPool.add(t);
        }
        this.sem = new Semaphore(size);
    }

    R exec(Function<T, R> func) throws InterruptedException {
        T t = null;
        try {
            sem.acquire();
            t = objPool.remove(0);
            return func.apply(t);
        } finally {
            // 模拟延时
            Thread.sleep(1000);
            objPool.add(t);
            sem.release();
        }
    }
}
```

我们用一个 List来保存对象实例，用 Semaphore 实现限流器。关键的代码是 ObjectPool里面的 exec() 方法，这个方法里面实现了限流的功能。在这个方法里面，我们首先调用 acquire() 方法（与之匹配的是在 finally 里面调用 release() 方法），假设对象池的大小是 10，信号量的计数器初始化为 10，那么前 10 个线程调用 acquire() 方法，都能继续执行，相当于通过了信号灯，而其他线程则会阻塞在 acquire() 方法上。对于通过信号灯的线程，我们为每个线程分配了一个对象 t（这个分配工作是通过 pool.remove(0) 实现的），分配完之后会执行一个回调函数 func，而函数的参数正是前面分配的对象 t ；执行完回调函数之后，它们就会释放对象（这个释放工作是通过 pool.add(t) 实现的），同时调用 release() 方法来更新信号量的计数器。如果此时信号量里计数器的值小于等于 0，那么说明有线程在等待，此时会自动唤醒等待的线程。