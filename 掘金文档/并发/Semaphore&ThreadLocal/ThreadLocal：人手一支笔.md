除了控制资源的访问外，我们还可以通过增加资源来保证所有对象的线程安全。比如，让100个人填写个人 信息表，如果只有一只笔，那么大家就得挨个填写，对于管理人员来说，必须保证大家不会去哄抢这仅存的一支笔，否则，谁也填不完。从另外一个角度出发，我们干脆就准备100支笔，人手一支，那么所有人都可以很快完成表格的填写。

如果说锁是使用第一种思路，那么ThreadLocal就是使用第二种思路了。

### ThreadLocal的简单使用

```java
public class ThreadLocalDemo {

    private static final ThreadLocal<SimpleDateFormat> tl = new ThreadLocal<>();

    public static void main(String[] args) {
        ExecutorService executorService = Executors.newFixedThreadPool(10);
        for (int i = 0; i < 10000; i++) {
            executorService.execute(new ParseDate(i));
        }
        executorService.shutdown();
    }

    static class ParseDate implements Runnable {

        int i = 0;

        public ParseDate(int i) {
            this.i = i;
        }

        @Override
        public void run() {
            try {
                // 1
                if (tl.get() == null) {
                    tl.set(new SimpleDateFormat("yyyy-MM-dd HH:mm:ss"));
                }
                Date date = tl.get().parse("2020-06-28 19:25:" + i % 60);
                System.out.println(i+ ":" + date);
            } catch (ParseException e) {
                e.printStackTrace();
            }
        }
    }
}
```

`SimpleDateFormat.parse()`并不是线程安全的，我们使用ThreadLocal为每一个线程都产生一个`SimpleDateFormat`对象实例。

注释1的地方判断如果当前线程不持有`SimpleDateFormat`对象实例。那么就新建一个并设置到当前线程中，如果已经持有，则直接使用。从这里可以看到，为每一个线程人手分配一个对象的工作并不是由ThreadLocal来完成的，而是需要在应用层面保证的。如果在应用上为每一个线程分配了相同的对象实例，那么ThreadLocal也不能保证线程安全。

> 为每一个线程分配不同的对象，需要在应用层面保证。ThreadLocal只是起到了简单的容器作用。

### ThreadLocal实现原理

关于ThreadLocal我们需要关注的是ThreadLocal的set()和get()两个方法。

```java
public void set(T value) {
    Thread t = Thread.currentThread();
    ThreadLocalMap map = getMap(t);
    if (map != null)
        map.set(this, value);
    else
        createMap(t, value);
}
```

在set时，首先获得当前线程对象，然后通过getMap()拿到线程的ThreadLocalMap，并将值设入ThreadLocalMap中。

```java 
public T get() {
    Thread t = Thread.currentThread();
    ThreadLocalMap map = getMap(t);
    if (map != null) {
        ThreadLocalMap.Entry e = map.getEntry(this);
        if (e != null) {
            @SuppressWarnings("unchecked")
            T result = (T)e.value;
            return result;
        }
    }
    return setInitialValue();
}
```

在get时，首先获得ThreadLocalMap对象。然后将自己作为key取得内部的实际数据。

当使用ThreadLocal时，维护的变量会维护在Thread类内部的ThreadLocalMap中，这也意味着只要线程不退出，对象的引用就会一直存在。

因此如果我们使用线程池，那就意味着当前线程未必会退出。如果这样，将一些大对象设置到ThreadLocal中，可能会使系统出现内存泄露的可能。

此时，如果你希望及时回收对象，最好使用ThreadLocal.remove()方法将这个变量移除。就像我们习惯性地关闭数据库连接一样。如果你确实不需要这个对象了，那么就应该告诉虚拟机，请把它回收掉，防止内存泄露。

```java 
static class Entry extends WeakReference<ThreadLocal<?>> {
    /** The value associated with this ThreadLocal. */
    Object value;

    Entry(ThreadLocal<?> k, Object v) {
        super(k);
        value = v;
    }
}
```

ThreadLocalMap类似于一个Map，其中的key为当前定义的ThreadLocal变量的this弱引用，value为我们使用set方法设置的值。

![huishou](C:\Users\18073632\Desktop\binfa\ThreadLocal\huishou.png)

