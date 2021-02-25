# Chapter12 Java内存模型与线程

Java规定，所有变量存储在主内存中，每个线程都有自己独有的工作内存，线程对变量的所有操作都必须保存一份主内存副本到自己的工作内存，写完再同步回主内存（对象的引用、对象中某个被线程访问到的字段是有可能被复制的，但是不会复制一整个对象到工作内存）。多个线程之间传递值也必须要通过主内存传递。

volatile关键字：

特性1：可见性

对于普通变量的写操作，线程在自己的工作内存内修改之后，处理器为了提高处理速度，可能不会立即把新值同步回主存；而且就算写回主存，可能在写回主存之前，其他线程就已经在自己的工作内存中存了副本，保存的是旧值，这个旧值可能还没有更新就被拿去用，这就会出现问题。

而对于volatile变量，线程写操作之后，JVM发出Lock前缀的指令，引起处理器**立即**把新值同步回主存；并且导致其他处理器的缓存失效，这样其他线程在读取同一个变量时会发现缓存失效，直接去主存读取最新值并使用，实现volatile变量的更新能**立刻**反映到其他线程上

特性2：禁止指令重排序优化

Lock指令相当于一个“内存屏障”，使得指令重排序无法越过内存屏障（[这块待续](https://juejin.cn/post/6844903601064640525)）

volatile总结：在工作内存中，每次使用volatile变量之前都必须先从主内存获取最新值；每次修改volatile之后，都必须立刻同步回主内存中；volatile修饰的变量不会被指令重排序优化。（[这块待续](https://juejin.cn/post/6844903601064640525)）

但是，volatile并不能保证线程安全：

```java
public class VolatileTest {
    public static volatile int race = 0;
    public static void increase(){
        race++;
    }
    public static final int THREADS_COUNT = 20;
    public static void main(String[] args) throws InterruptedException {
        Thread[] threads = new Thread[THREADS_COUNT];
        for(int i =0; i < THREADS_COUNT; i++){
            threads[i] = new Thread(new Runnable() {
                @Override
                public void run() {
                    for (int j = 0; j < 10000; j++) {
                        increase();//这个race++操作就算只有一条字节码指令完成，也不能说明这个操作具有【原子性】
                    }
                }
            });
            threads[i].start();
        }
//        while(Thread.activeCount() > 2){
//            Thread.yield();
//        }
        for (int i = 0; i < THREADS_COUNT; i++) {
            threads[i].join();
        }
        System.out.println(race);
    }
}
```

final也具有可见性，因为final一旦被被构造器初始化完成，其他线程都能看到final值，且其值不需要同步就能被正确访问（值/引用值）

Java线程一般都是直接映射到操作系统本身的原生线程上，一切线程调度（Java可以给线程设置优先级，给OS提建议）、阻塞、唤醒均由操作系统完成（但是这种需要内核态用户态转换，耗能）

Java使用抢占式调度，每个线程由操作系统分配执行时间。

线程状态转换图：

<img src="C:\Users\123\Pictures\JVM\线程状态.jpg" style="zoom: 33%;" />

# Chapter13 线程安全与锁优化

[AtomicInteger & Vector（Vector的单次操作线程安全）](E:\AtomicInteger&Vector.md)

线程安全的实现方法：

- 互斥同步（Lock & synchronized）保证共享数据同一时间只有一个/一部分线程可以访问

  synchronized可重入且非公平锁（即并不是先来的线程就优先获取资源）

  线程一旦进入synchronized同步块，只要还没有释放锁，别的线程就无法进入；无法像Lock一样能强制打断正在阻塞等待的线程，也无法强制让正在持有锁的线程释放锁

  同步块一旦抛出异常，JVM能保证synchronized的锁也能被正常释放；但是对于Lock，就需要finally块来保证释放锁

- 非阻塞同步：CAS等

  Java的CAS是由Unsafe类的compareAndSwapInt（）、compareAndSwapLong（）等一系列方法实现

  CAS的ABA问题

- 无同步方案：可重入代码（不管怎么执行，只要输入的数据不变，得到的结果永远不变）；线程本地存储（ThreadLocal等）

### 关于Thread Local：

首先，每个Thread内部有一个ThreadLocalMap类型的Map：threadLocals，k-v对为：ThreadLocal实例->存入的Object；

再者，每个ThreadLocal实例都有一个final的threadLocalHashCode值，[应该是在类加载的时候就已经完成了赋值](E:\Java类加载)，这个值在ThreadLocal.set方法要用到

创建了一个ThreadLocal实例：

`private ThreadLocal<String> myThreadLocal = new ThreadLocal<String>();`

设置值：

`myThreadLocal.set("I'm a threadLocal");`

对于ThreadLocal的set方法：

```java
public void set(T value) {
        Thread t = Thread.currentThread();
        ThreadLocalMap map = getMap(t);
        if (map != null)
            map.set(this, value);
        else
            createMap(t, value);
}

void createMap(Thread t, T firstValue) {
        t.threadLocals = new ThreadLocalMap(this, firstValue);
}
```

如果当前线程内部的ThreadLocalMap为null，给这个线程新建一个ThreadLocalMap，k-v为当前这个ThreadLocal实例->要存入的Object

或者map.set：在当前线程内部的ThreadLocalMap存入键值对：当前ThreadLocal实例->存入的Object，相当于让当前线程和当前这个ThreadLocal实例联系起来。set的时候，和普通HashMap类似，使用ThreadLocal实例的threadLocalHashcode&找到存储的位置，然后存入值

获取值：

`myThreadLocal.get();`

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

获取当前线程内部的ThreadLocalMap，根据当前的ThreadLocal对象，找到对应值返回。

![](C:\Users\123\Pictures\JVM\ThreadLocal.jpg)

ThreadLocalMap中的key在创建的时候是弱引用：

```java
static class Entry extends WeakReference<ThreadLocal<?>> {
    /** The value associated with this ThreadLocal. */
    Object value;

    Entry(ThreadLocal<?> k, Object v) {
        super(k);//k是弱引用
        value = v;
    }
}
```

弱引用导致：在把ThreadLocal的引用置null之后，如果只剩下各个线程的key的弱引用指向实际对象的话，JVM会直接回收只有弱引用的对象实例，导致各线程中对应key=null；所以key这方面不大可能会出现内存泄露的情况，但是value就不一定了：

本身ThreadLocalMap的生命周期和线程本身一致，线程退出的时候的确会强制回收ThreadLocalMap的内存占用；但是如果线程一直不退出（可能是线程池的核心线程），就算key=null，value也还有线程引用->ThreadLocalMap引用->Entry引用，导致不用的value内存无法回收。

ThreadLocal对此的改进：ThreadLocalMap内部的get set remove方法在被调用的时候，都会对key=null的Entry做清理：比如在内部调用expungeStaleEntry方法、expungeStaleEntries、replaceStaleEntry等方法，对key=null的Entry清理或者替代。来尽量避免内存泄漏



Java锁优化：

- 自旋锁&自适应锁

- 锁消除
- 锁粗化
- 轻量级锁
- 偏向锁

对于重写过hashcode方法的对象，其hashcode不存储于对象头的Mark Word中；如果没有重写：

如果计算过identityHashCode，对象不会进入偏向锁状态；如果在偏向锁状态之后计算identityHashCode，偏向锁状态被撤销，膨胀为重量级锁，重量级锁的对象头指向ObjectMonitor，这个monitor内部可以记录非加锁状态下的MarkWord，其identityHashCode自然也可以存储