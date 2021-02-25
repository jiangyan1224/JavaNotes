#### FutureTask：

FutureTask不仅可以被用在线程池中，也可以单独使用。它本身作为一个高级版的Callable（Runnable会被转成Callable），不仅可以被线程拿去执行，也可以被cancel、interrupt、和获取执行结果。

FutureTask内部的实现也考虑到了线程安全，用到了state、等待队列和CAS保证。

假设当前多个线程执行多个任务，但是又要求每个任务只被执行一次，不能被多个线程重复执行，每个任务都要有返回结果，就可以用FutureTask：

```java
private ConcurrentHashMap<String, Future<String>> taskCache
            = new ConcurrentHashMap<>();
    public String executeFuture(String taskName){
        if (taskName == null) return null;
        while(true){//这个死循环，可以让在只有一个线程a成功执行任务，其他线程都在get处阻塞时，如果任务的执行抛出了异常，
            //a可以删除对应键值对，新建Callable，再次尝试执行
            Future future = taskCache.get(taskName);
            if (future == null){
                Callable c = new Callable() {
                    @Override
                    public String call() throws Exception {
                        return taskName;
                    }
                };
                FutureTask<String> task = new FutureTask<String>(c);
                future = taskCache.putIfAbsent(taskName, task);
                if (future == null){//如果某个线程迟了一步，其他线程先执行了putIfAbsent，future就是oldVal而不是null
                    future = task;//防止从这个if出去之后，到future.get出现空指针异常
                    task.run();
                }
            }
            try {
                return (String) future.get();
            } catch (InterruptedException | ExecutionException e) {
                //remove(key)和remove(key,value)区别在于，一个是只要匹配key就删除，另一个是key和value都匹配才删除
                //在这里，我觉得两个remove都可以
                taskCache.remove(taskName, future);
                e.printStackTrace();
            }
        }
    }
```

（其实这个代码中，真正保证线程安全的应该是ConcurrentHashMap，FutureTask的作用在于让所有不在执行对应任务的线程处于阻塞，等待任务完成之后唤醒它们。如果一堆线程都去执行同一个FutureTask的run方法，FutureTask本身的CAS也可以保证只有一个线程能进入call，run之后调用get就可以获取任务的执行结果/异常）

[学习链接](https://segmentfault.com/a/1190000016572591)

**FutureTask三要素：state状态、WaitNode等待队列（阻塞在get上的线程们）、CAS**

1、state状态：表明当前任务所处状态

一个FutureTask如果顺利执行，不考虑被打断或者取消，从新建到执行过程结束之前，都是NEW

```
private volatile int state;
private static final int NEW          = 0;
private static final int COMPLETING   = 1;
private static final int NORMAL       = 2;
private static final int EXCEPTIONAL  = 3;
private static final int CANCELLED    = 4;
private static final int INTERRUPTING = 5;
private static final int INTERRUPTED  = 6;
```

![](C:\Users\123\Pictures\JVM\FutureTask状态转换.png)

2、WaitNode：相对于AQS的等待队列，这里只是一个简单的单链表：

`值得一提的是，FutureTask中的这个单向链表是当做栈来使用的，确切来说是当做Treiber栈来使用的，不了解Treiber栈是个啥的可以简单的把它当做是一个线程安全的栈，它使用CAS来完成入栈出栈操作(想进一步了解的话可以看这篇文章)。为啥要使用一个线程安全的栈呢，因为同一时刻可能有多个线程都在获取任务的执行结果，如果任务还在执行过程中，则这些线程就要被包装成WaitNode扔到Treiber栈的栈顶，即完成入栈操作，这样就有可能出现多个线程同时入栈的情况，因此需要使用CAS操作保证入栈的线程安全，对于出栈的情况也是同理。`

`由于FutureTask中的队列本质上是一个Treiber栈，那么使用这个队列就只需要一个指向栈顶节点的指针就行了，在FutureTask中，就是waiters属性：`

```java
/** Treiber stack of waiting threads */
private volatile WaitNode waiters;
//事实上，它就是整个单向链表的头节点。
```

大部分直接看链接的文章都可以看懂，有一个地方：在说run方法的finally块的时候，对于为什么会进入handlePossibleCancellationInterrupt方法：

```java
    public void run() {
        if (state != NEW ||
            !UNSAFE.compareAndSwapObject(this, runnerOffset,
                                         null, Thread.currentThread()))
            return;
        try {
            Callable<V> c = callable;
            if (c != null && state == NEW) {
                V result;
                boolean ran;
                try {
                    result = c.call();
                    ran = true;
                } catch (Throwable ex) {
                    result = null;
                    ran = false;
                    setException(ex);
                }
                if (ran)
                    set(result);
            }
        } finally {
            // runner must be non-null until state is settled to
            // prevent concurrent calls to run()
            runner = null;
            // state must be re-read after nulling runner to prevent
            // leaked interrupts
            int s = state;
            if (s >= INTERRUPTING)
                handlePossibleCancellationInterrupt(s);
        }
    }
```

假设有三个线程ABC：

A线程第一个进入run，顺利调用call()执行任务，对于A来说，不管执行任务是否成功，最后任务的state都被置为NORMAL或者EXCEPTIONAL；

B调用了cancel取消任务，如果此时任务还没有完成，state依然是NEW，B就会把任务状态设置为NEW -> INTERRUPTING -> INTERRUPTED（如果Bcancel的时候，A已经完成了任务，cancel是直接返回的）

```java
    public boolean cancel(boolean mayInterruptIfRunning) {
        if (!(state == NEW &&
              UNSAFE.compareAndSwapInt(this, stateOffset, NEW,
                  mayInterruptIfRunning ? INTERRUPTING : CANCELLED)))//如果任务已经完成/被cancel过，直接退出
            return false;
        try {    // in case call to interrupt throws exception
            if (mayInterruptIfRunning) {
                try {
                    Thread t = runner;
                    if (t != null)
                        t.interrupt();
                } finally { // final state
                    UNSAFE.putOrderedInt(this, stateOffset, INTERRUPTED);
                }
            }
        } finally {
            finishCompletion();
        }
        return true;
    }
```

而C在Bcancel成功之后，进入run，由于此时state已经是INTERRUPTING/INTERRUPTED，就直接进入finally块，在handlePossibleCancellationInterrupt中一直自旋，等待任务的state进入终止态EXCEPTIONAL/INTERRUPTING/INTERRUPTED再退出。