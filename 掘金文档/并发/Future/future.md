### Future简介

Future模式是多线程并发编程的扩展，这种方式支持返回结果，使用get()方法阻塞当前线程。它的核心思想是异步调用，当我们执行某个函数时，它可能很慢，但是我们又不着急要结果。因此，我们可以让它立即返回，让它异步去执行这个请求，当我们需要结果时，阻塞调用线程获取结果。

一个`Future<V>`接口表示一个未来可能会返回的结果，它定义的方法有：

- `get()`：获取结果（可能会等待）
- `get(long timeout, TimeUnit unit)`：获取结果，但只等待指定的时间；
- `cancel(boolean mayInterruptIfRunning)`：取消当前任务；
- `isDone()`：判断任务是否已完成。

### 烧水泡茶

![烧水泡茶](http://www.duanle.net/upload/2020/06/烧水泡茶-0d3041ff45e84f60bdd8e96201e8f5e5.png)


```java
static class T1Task implements Callable<String> {

    final FutureTask<String> t2;

    public T1Task(FutureTask<String> t2) {
        this.t2 = t2;
    }

    @Override
    public String call() throws Exception {
        System.out.println("T1:洗水壶...");
        TimeUnit.SECONDS.sleep(1);
        System.out.println("T1:烧开水...");
        TimeUnit.SECONDS.sleep(15);
        // 阻塞等待t2执行结果
        String t2result = t2.get();
        System.out.println("T1:拿到茶叶："+t2result);
        System.out.println("T1:泡茶...");
        return "上茶: " + t2result;
    }
}

static class T2Task implements Callable<String> {

    @Override
    public String call() throws Exception {
        System.out.println("T2:洗茶壶...");
        TimeUnit.SECONDS.sleep(1);
        System.out.println("T2:洗茶杯...");
        TimeUnit.SECONDS.sleep(2);
        System.out.println("T2:拿茶叶...");
        TimeUnit.SECONDS.sleep(1);
        return "龙井";
    }
}

public static void main(String[] args) throws ExecutionException, InterruptedException {
    FutureTask<String> ft2 = new FutureTask<>(new T2Task());
    FutureTask<String> ft1 = new FutureTask<>(new T1Task(ft2));

    ExecutorService executorService = Executors.newFixedThreadPool(2);
    executorService.submit(ft1);
    executorService.submit(ft2);

    String result = ft1.get();
    System.out.println(result);
    executorService.shutdown();
}
```

通过优化，充分利用了烧水这个耗时的过程去执行其他的操作，提高响应速度。

### Future增强：CompletableFuture

使用`Future`获得异步执行结果时，要么调用阻塞方法`get()`，要么轮询看`isDone()`是否为`true`，这两种方法都不是很好，因为主线程也会被迫等待。

从Java 8开始引入了`CompletableFuture`，它针对`Future`做了改进，可以传入回调对象，当异步任务完成或者发生异常时，自动调用回调对象的回调方法。

```java
public class CompletableFutureDemo {
    public static void main(String[] args) throws Exception {
        // 创建异步执行任务:
        CompletableFuture<Double> cf = CompletableFuture.supplyAsync(
            CompletableFutureDemo::fetchPrice);
        // 如果执行成功:
        cf.thenAccept((result) -> System.out.println("price: " + result));
        // 如果执行异常:
        cf.exceptionally((e) -> {
            e.printStackTrace();
            return null;
        });
        // 主线程不要立刻结束，否则CompletableFuture默认使用的线程池会立刻关闭:
        Thread.sleep(200);
    }

    static Double fetchPrice() {
        try {
            Thread.sleep(100);
        } catch (InterruptedException e) {
        }
        if (Math.random() < 0.3) {
            throw new RuntimeException("fetch price failed!");
        }
        return 5 + Math.random() * 20;
    }
}
```

创建一个`CompletableFuture`是通过`CompletableFuture.supplyAsync()`实现的，它需要一个实现了`Supplier`接口的对象：

```java
public interface Supplier<T> {
    T get();
}
```

这里我们用lambda语法简化了一下，直接传入`CompletableFutureDemo::fetchPrice`，因为`CompletableFutureDemo.fetchPrice()`静态方法的签名符合`Supplier`接口的定义（除了方法名外）。

紧接着，`CompletableFuture`已经被提交给默认的线程池执行了，我们需要定义的是`CompletableFuture`完成时和异常时需要回调的实例。完成时，`CompletableFuture`会调用`Consumer`对象：

```java
public interface Consumer<T> {
    void accept(T t);
}
```

异常时，`CompletableFuture`会调用：

```java
public interface Function<T, R> {
    R apply(T t);
}
```

可见`CompletableFuture`的优点是：

- 异步任务结束时，会自动回调某个对象的方法；
- 异步任务出错时，会自动回调某个对象的方法；
- 主线程设置好回调后，不再关心异步任务的执行。

> 除此之外，`CompletableFuture`还允许将多个`CompletableFuture`进行组合。

**CompletionStage接口**：

>描述and汇聚关系：

1. thenCombine:任务合并，有返回值
2. thenAccepetBoth:两个任务执行完成后，将结果交给thenAccepetBoth消耗，无返回值。
3. runAfterBoth:两个任务都执行完成后，执行下一步操作（Runnable）。

>描述or汇聚关系

1. applyToEither:两个任务谁执行的快，就使用那一个结果，有返回值。
2. acceptEither: 两个任务谁执行的快，就消耗那一个结果，无返回值。
3. runAfterEither: 任意一个任务执行完成，进行下一步操作(Runnable)。

**CompletableFuture类自己也提供了anyOf()和allOf()用于支持多个`CompletableFuture`并行执行**。

示例：
```java
// thenCombine
private static void thenCombine() {
    CompletableFuture<String> f1 = CompletableFuture.supplyAsync(() -> {
        String result = "f1";
        System.out.println(result);
        return result;
    });
    CompletableFuture<String> f2 = CompletableFuture.supplyAsync(() -> {
        String result = "f2";
        System.out.println(result);
        return result;
    });

    CompletableFuture<String> f3 = f1.thenCombine(f2, (s, s2) -> {
        System.out.println(s + "_" + s2);
        return s + s2;
    });

    String join = f3.join();
    System.out.println(join);
}
// applyToEither
private static void applyToEither() {
    CompletableFuture<String> f1 = CompletableFuture.supplyAsync(() -> {
        String result = "f1";
        sleep(2);
        System.out.println(result);
        return result;
	});
	CompletableFuture<String> f2 = CompletableFuture.supplyAsync(() -> {
        String result = "f2";
        sleep(3);
        System.out.println(result);
        return result;
	});
	CompletableFuture<String> f3 = f1.applyToEither(f2, s -> {
        System.out.println(s);
        return s;
	});
	String join = f3.join();
	System.out.println(join);
}

// anyOf() 从jd、ali、sn平台查询商品价格，任意一个返回就将其作为返回结果
private static void queryProductPriceByCode() throws InterruptedException {
	// 三个CompletableFuture执行异步查询:
	CompletableFuture<Double> ft1 = CompletableFuture.supplyAsync(
            () -> queryProductPriceByCode("1", "jd"));
	CompletableFuture<Double> ft2 = CompletableFuture.supplyAsync(
            () -> queryProductPriceByCode("1", "ali"));
	CompletableFuture<Double> ft3 = CompletableFuture.supplyAsync(
            () -> queryProductPriceByCode("1", "sn"));

	// 用anyOf合并为一个新的CompletableFuture:
	CompletableFuture<Object> ft4 = CompletableFuture.anyOf(ft1, ft2, ft3);

	// 最终结果:
	ft4.thenAccept((result) -> {
		System.out.println("price: " + result);
	});
	// 主线程不要立刻结束，否则CompletableFuture默认使用的线程池会立刻关闭:
    Thread.sleep(200);
}
```

**重写烧水泡茶**

```java
CompletableFuture<Void> cf1 = CompletableFuture.runAsync(() -> {
	System.out.println("T1:洗水壶...");
	sleep(1);
	System.out.println("T1:烧开水...");
	sleep(15);
});

CompletableFuture<String> cf2 = CompletableFuture.supplyAsync(() -> {
	System.out.println("T2:洗茶壶...");
	sleep(1);
	System.out.println("T2:洗茶杯...");
	sleep(2);
	System.out.println("T2:拿茶叶...");
	sleep(1);
	return "龙井";
});

CompletableFuture<String> cf3 = cf1.thenCombine(cf2, (__, tf) -> {
	System.out.println("T1:拿到茶叶:" + tf);
	System.out.println("T1:泡茶...");
	return "上茶:" + tf;
});
```

### 批量执行异步任务

我们思考下这个场景：从三个电商询价，然后保存在自己的数据库里。通过之前所学，我们可能这么实现。

```java
// 创建线程池
ExecutorService executor = Executors.newFixedThreadPool(3);
// 异步向电商 S1 询价
Future<Integer> f1 = executor.submit(()->getPriceByS1());
// 异步向电商 S2 询价
Future<Integer> f2 = executor.submit(()->getPriceByS2());
// 异步向电商 S3 询价
Future<Integer> f3 = executor.submit(()->getPriceByS3());
    
// 获取电商 S1 报价并保存
r=f1.get();
executor.execute(()->save(r));
  
// 获取电商 S2 报价并保存
r=f2.get();
executor.execute(()->save(r));
  
// 获取电商 S3 报价并保存  
r=f3.get();
executor.execute(()->save(r));
```

上面的这个方案本身没有太大问题，但是有个地方的处理需要你注意，那就是如果获取电商 S1 报价的耗时很长，那么即便获取电商 S2 报价的耗时很短，也无法让保存 S2 报价的操作先执行，因为这个主线程都阻塞在了 `f1.get()`,那我们如何解决了？

我们可以增加一个阻塞队列，获取到 S1、S2、S3 的报价都进入阻塞队列，然后在主线程中消费阻塞队列，这样就能保证先获取到的报价先保存到数据库了。下面的示例代码展示了如何利用阻塞队列实现先获取到的报价先保存到数据库。

```java
// 创建阻塞队列
BlockingQueue<Integer> bq = new LinkedBlockingQueue<>();
// 电商 S1 报价异步进入阻塞队列  
executor.execute(() -> bq.put(f1.get()));
// 电商 S2 报价异步进入阻塞队列  
executor.execute(() -> bq.put(f2.get()));
// 电商 S3 报价异步进入阻塞队列  
executor.execute(() -> bq.put(f3.get()));
// 异步保存所有报价  
for (int i=0; i<3; i++) {
  Integer r = bq.take();
  executor.execute(()->save(r));
}  
```

在Java8中提供了一个新的接口CompletionService 和其实现类ExecutorCompletionService对上述进行了封装，以支持**任务先完成可优先获取到，即结果按照完成先后顺序排序**。CompletionService 内部也维护了一个阻塞队列`BlockingQueue<Future<V>> completionQueue`，不过它不是用来存放任务执行结果的而是直接将任务执行结果的Future对象加入到阻塞队列。

**重写电商询价：**

```java
ExecutorService es = Executors.newFixedThreadPool(3);
try {
    // 询价
    CompletionService<Integer> cs = new ExecutorCompletionService<>(es);
    cs.submit(() -> getPriceByCode(1));
    cs.submit(() -> getPriceByCode(2));
    cs.submit(() -> getPriceByCode(3));

    for (int i = 0; i < 3; i++) {
        Integer result = cs.take().get();
        System.out.println(result);
    }
} catch (Exception var) {
    var.printStackTrace();
} finally {
    es.shutdown();
}
```