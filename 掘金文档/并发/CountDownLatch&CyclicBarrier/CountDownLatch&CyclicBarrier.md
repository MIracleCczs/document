### CountDownLatch

`CountDownLatch`是一个非常实用的多线程控制工具类。latch是门闩的意思，该工具是为了解决某些操作只能在一组操作全部执行完成后才能执行的情景。例如，游乐园里的过山车，一次可以坐10个人，为了节约成本，通常是等够10个人了才开。`CountDown`是倒数计数，所以CountDownLatch的用法通常是设定一个大于0的值，该值即代表需要等待的总任务数，每完成一个任务后，将总任务数减一，直到最后该值为0，说明所有等待的任务都执行完了，“门闩”此时就被打开，后面的任务可以继续执行。

`CountDownLatch`的构造函数接收一个整数作为参数，它表示当前计数器的个数。

```java
public CountDownLatch(int count)
```

#### 核心方法

```java
// 计数器减1
public void countDown()
// 等待
public void await() throws InterruptedException
// 带超时机制等待
public boolean await(long timeout, TimeUnit unit) throws InterruptedException
```

> countDown()计数逻辑的底层实现

```java
protected boolean tryReleaseShared(int releases) {
    // Decrement count; signal when transition to zero
    for (;;) {
        int c = getState();
        if (c == 0)
            return false;
        int nextc = c-1;
        if (compareAndSetState(c, nextc))
            return nextc == 0;
    }
}
```

该方法的实现很简单，就是获取当前的state值，如果已经为0了，直接返回false；否则通过CAS操作将state值减一，之后返回的是`nextc == 0`，由此可见，**该方法只有在count值原来不为0，但是调用后变为0时，才会返回true，否则返回false**，并且也可以看出，该方法在返回true之后，后面如果再次调用，还是会返回false。也就是说，**调用该方法只有一种情况会返回true，那就是state值从大于0变为0值时**，这时也是所有在门闩前的任务都完成了。

在执行`tryReleaseShared()`方法之后，如果结果为true，则继续执行`doReleaseShared()`方法唤醒等待线程。

#### 简单示例

```java
public class CountDownLatchTest {

    public static void main(String[] args) throws InterruptedException {

        CountDownLatch latch = new CountDownLatch(10);
        ExecutorService executorService = Executors.newFixedThreadPool(10);

        for (int i = 0; i < 10; i++) {
            executorService.submit(
                    () -> {
                        action();
                        latch.countDown(); // -1
                    });
        }
        // 等待完成
        // latch 计数器>0会被阻塞
        latch.await();
        System.out.println("乘客已满，过山车启动");
        executorService.shutdown();
    }

    private static void action() {
        System.out.printf("游客[%s] 正在乘坐过山车...\n", Thread.currentThread().getName());
    }
}
```

过上车有10个位置，只有游客满时过山车才会启动，即线程在`CountDownLatch`上等待，直到计数器为0时，主线程才会继续执行。

### CyclicBarrier

`CyclicBarrier`是另外一种多线程并发控制实用工具。和`CountDownLatch`类似，它也可以实现线程间的计数等待，但它的功能比`CountDownLatch`更加强大。

`CyclicBarrier`可以理解为循环栅栏。它用来阻止线程继续执行，要求线程在栅栏处等待。前面的`Cyclic`意为循环，也就是说这个计数器可以反复使用。比如，我们将计数器设置为10，那么凑齐第一批10个线程后，计数器就会归零，然后接着凑齐下一批10个线程。

#### 构造函数

```java
public CyclicBarrier(int parties)
public CyclicBarrier(int parties, Runnable barrierAction)
```

- parties 是参与线程的个数
- 第二个构造方法有一个 Runnable 参数，这个参数的意思是当计数器减为0时需要调用的回调函数。

#### 核心方法

```java
// CyclicBarrier.await() = CountDownLatch.countDown() + await()
public int await() throws InterruptedException, BrokenBarrierException
```

`await()`方法可能会抛出`java.util.concurrent.BrokenBarrierException`异常，一旦抛出这个异常就代表当前的`CyclicBarrier`损坏了，可能系统已经没有办法等待所有线程到齐了。破坏的原因可能是其中一个线程 await() 时被中断或者超时。

#### 简单示例

```java
public class CyclicBarrierDemo {

    static class TaskThread extends Thread {

        CyclicBarrier barrier;

        public TaskThread(CyclicBarrier barrier) {
            this.barrier = barrier;
        }

        @Override
        public void run() {
            try {
                Thread.sleep(1000);
                System.out.println(getName() + " 到达栅栏 A");
                barrier.await();
                System.out.println(getName() + " 冲破栅栏 A");

                Thread.sleep(2000);
                System.out.println(getName() + " 到达栅栏 B");
                barrier.await();
                System.out.println(getName() + " 冲破栅栏 B");
            } catch (Exception e) {
                e.printStackTrace();
            }
        }
    }

    public static void main(String[] args) {
        int threadNum = 5;
        CyclicBarrier barrier = new CyclicBarrier(threadNum, () -> System.out.println("完成最后任务"));

        for(int i = 0; i < threadNum; i++) {
            new TaskThread(barrier).start();
        }
    }

}
```

### CountDownLatch与CyclicBarrier比较



### 总结

1. `CountDownLatch` 主要用来解决一个线程等待多个线程的场景;
2. `CyclicBarrier` 是一组线程之间互相等待;
3. `CountDownLatch` 的计数器是不能循环利用的，也就是说一旦计数器减到 0，再有线程调用 await()，该线程会直接通过;
4. `CyclicBarrier` 的计数器是可以循环利用的，而且具备自动重置的功能，一旦计数器减到 0 会自动重置到你设置的初始值, 还可以设置回调函数。