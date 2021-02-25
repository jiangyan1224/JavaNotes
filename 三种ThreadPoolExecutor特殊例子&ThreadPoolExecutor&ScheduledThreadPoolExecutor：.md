**FixedThreadPoolExecutor、SingledThreadPoolExecutor、CachedThreadPool这三种线程池实际上都是设置了特殊参数值的ThreadPoolExecutor；而ScheduledThreadPoolExecutor则是ThreadPoolExecutor的子类：**



**FixedThreadPoolExecutor：**

核心线程数和最大线程数相等，非核心线程的空闲等待时间为0，使用无界队列

又因为默认情况下allowCoreThreadTimeOut=false，所以这个线程池内的所有非核心线程都是一空闲就被remove（实际上这个线程池并不存在非核心线程）；所有核心线程一直阻塞



**SingledThreadPoolExecutor：**核心线程=最大线程数=1的FixedThreadPoolExecutor



**CachedThreadPool：**没有核心线程，最大线程数为Integer最大值，每来一个任务，先进阻塞队列，如果有活着的线程在调用poll，就被拿走执行；否则就新建一个线程执行任务



**ThreadPoolExecutor：**

笔者认为，ThreadPoolExecutor中最核心的是这几个部件：Worker和BlockingQueue（和 拒绝策略 和 threadFactory）

1、BlockingQueue：`private final BlockingQueue<Runnable> workQueue;`，存储等待执行的Runnable任务

**submit：**会把提交的Runnable/Callable包装为FutureTask（Runnable会用Executors.callable转成Callable，默认result=null）

**execute：**只接受Runnable，不会像submit一样，对提交的任务做额外处理。

然后元素在被取出执行的时候，调用元素的run方法

所以，在执行execute的任务的run时，就只是简单的执行了一下Runnable的run，没有返回值等；

在执行submit的任务的run，即FutureTask.run时：

```java
if (c != null && state == NEW) {
    V result;
    boolean ran;
    try {
        result = c.call();//
        ran = true;
     } catch (Throwable ex) {
       result = null;
       ran = false;
       setException(ex);
     }
   if (ran)
   	   set(result);//这里set的值就是后面Future.get获取到的值，即result
}
```

对于`result = c.call();`：

如果是Runnable转换而来的Callable，转的时候传入的是什么参数，得到的result也就是什么参数，没有改动；

如果本来就是Callable，那就是返回Callable.call的执行结果

注：Runnable -> Callable：

```java
//还有一个重载版本是没有result参数的，实际上相当于传入result=null
public static <T> Callable<T> callable(Runnable task, T result) {
        if (task == null)
            throw new NullPointerException();
        return new RunnableAdapter<T>(task, result);
}

static final class RunnableAdapter<T> implements Callable<T> {
        final Runnable task;
        final T result;
        RunnableAdapter(Runnable task, T result) {
            this.task = task;
            this.result = result;
        }
        public T call() {
            task.run();
            return result;//没有对result做任何改动
        }
}
```

2、Worker：实现了Runnable接口

内含一个Thread：`this.thread = getThreadFactory().newThread(this);`，也就是说，这个线程run的时候也是执行的Worker的run方法

addWorker中新建worker对象，`this.thread = getThreadFactory().newThread(this);`并调用worker.thread.start()。所以就是新建一个线程，执行Worker的run方法->runWorker方法，在runWorker中，循环从阻塞队列中take/poll取出任务，如果成功拿到，就执行队列元素的run方法：`task.run();`



**ScheduledThreadPoolExecutor：extends ThreadPoolExecutor**

DelayedWorkQueue：也是阻塞队列的一种

放元素的时候，是以优先级进行堆排序，决定元素在数组中的存放位置

取元素的时候，尝试从数组头部[0]取出元素，取出之后，finishPoll方法会把数组最后一个元素移动到数组头部，并进行堆的维护，将取出的元素下标置为-1并返回该元素

重写了它继承下来的所有execute和submit，全都转到自己的schedule方法。在schedule内部，把Runnable/Callable组装成ScheduledFutureTask（同时继承和实现了FutureTask和Runnable，相当于FutureTask的加强版。和FutureTask一样，如果传入的是Runnable，会把Runnable->Callable再组装）

组装好之后，把任务扔到阻塞队列DelayedWorkQueue，（定时任务，所以该阻塞队列的take方法实现也有特殊之处：）

```java
public RunnableScheduledFuture<?> take() throws InterruptedException {
    final ReentrantLock lock = this.lock;//lock 是对 DelayedWorkQueue上锁
    lock.lockInterruptibly();
    try {
        for (;;) {
            RunnableScheduledFuture<?> first = queue[0];//任务列表的头节点
            if (first == null)
                available.await();//available = lock.newCondition();
            else {
                long delay = first.getDelay(NANOSECONDS);//获取第一个任务还需要等待多久才能被取出
                if (delay <= 0)
                    return finishPoll(first);
                first = null; // don't retain ref while waiting
                /**
                leader的设计是为了尽量减少定时等待的线程数目，尽可能让只有一个线程在定时等待，其他线程都是无限期等待
                同一时刻，只有一个线程是leader，其他都是follower，正在运行不在等待的线程是在processing
                */
                if (leader != null)//private Thread leader = null;
                    available.await();
                else {
                    Thread thisThread = Thread.currentThread();
                    leader = thisThread;
                    try {
                        available.awaitNanos(delay);
                    } finally {
                        if (leader == thisThread)
                            leader = null;
                    }
                }
            }
        }
    } finally {
        if (leader == null && queue[0] != null)
            available.signal();
        lock.unlock();
    }
}
```

由Worker执行任务的run方法：

ScheduledFutureTask.run -> FutureTask.run/FutureTask.runAndReset（如果需要，会把这个任务再加入到阻塞队列中）

#### 总结：

**对于ThreadPoolExecutor，**

如果用execute提交的只能是Runnable，不会有任何额外处理；执行的时候直接执行Runnable的run

如果用submit，不管传入的Callable/Runnable，都会转成FutureTask，而FutureTask本身同时实现了Runnable和Future接口，会把Runnable转为Callable；执行的时候执行的是FutureTask.run -> Callable.call

**对于ScheduledThreadPoolExecutor：**

重写了所有execute和submit，都转到schedule，schedule会把Callable/Runnable转成加强版的FutureTask：ScheduledFutureTask。执行的是ScheduledFutureTask.run -> FutureTask的run/runAndReset -> Callable.call

**注：**FutureTask.run/runAndReset本身没有返回执行结果，但是执行完会给FutureTask实例内部字段赋值result

ScheduledThreadPoolExecutor不仅有schedule方法，还有scheduleAtFixedRate和scheduleAtFixeDelay方法，这两个方法和schedule不同：schedule方法不仅可以被其他类直接调用，也可以通过本类的execute/submit调用；而scheduleAtFixedRate和scheduleAtFixeDelay方法均直接由外界调用。

**使用execute/submit/或者直接去调用schedule方法，传入的Runnable/Callable，都是只有delay延迟，period属性值为0；而scheduleAtFixedRate和scheduleAtFixeDelay这两个方法，是可以设置period>0的。period值，后面调用ScheduledFutureTask.run时，会判断任务是不是period>0，如果是，runAndReset；否则只是run**

public ScheduledFuture<?> scheduleAtFixedRate(Runnable command, long initialDelay, long period, TimeUnit unit)：

第一个任务是0+initialDelay执行，后面的任务，最早是在initialDelay + (n-1)*period执行，如果前面一个任务还没有执行完毕，要等前面一个任务执行完，被立即调用。

public ScheduledFuture<?> scheduleWithFixedDelay(Runnable command, long initialDelay, long delay, TimeUnit unit)：

第一个任务是0+initialDelay执行，后面的任务，要在前面一个任务执行完之后，再延迟delay，再被调用。

**但是这两个方法并不能保证任务能被无限期的执行下去，如果任务执行本身抛出了异常，会导致后面所有的任务都被中断。**





对于submit返回的FutureTask：

```java
Runnable r = new Runnable() {
            @Override
            public void run() {
            }
};
Future future = pool.submit(r,"123");//这里的submit只会让线程池中的线程等待任务执行完成，本来main线程只要调用了submit，提交任务之后就可以砖头做其他事情
System.out.println(future.get());//但是一旦main线程调用了get方法，main就被阻塞住了
pool.shutdown();
```

```java
//main线程调用了get方法：
public V get() throws InterruptedException, ExecutionException {
        int s = state;
        if (s <= COMPLETING)//如果当前该FutureTask还没有执行完毕
            s = awaitDone(false, 0L);//阻塞当前线程直到超时或者任务完成，等当前线程park超时或者等任务完成之后unpark当前线程
        return report(s);
    }
```



