### 阻塞队列：

DelayQueue：队列中的元素必须实现Delayed接口，这些元素在创建的时候可以指定它能从队列中被取出的时间，只有延迟期满的时候才能被提取出来。可用于缓存系统的设计 / 定时任务的调度。

以**ScheduledThreadPoolExecutor.DelayedWorkQueue**为例，它的元素也实现了Delayed接口，它的take方法：

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

#### ArrayBlockingQueue：内部用一个Object数组存储元素，FIFO，循环队列，ReentrantLock全局锁

Condition notFull notEmpty；如果count已满，await

由于put和take都需要ReentrantLock全局锁，所以不允许两个线程同时读写



#### LinkedBlockingQueue：类似于链表存储数据，FIFO，生产和消费各一把锁

Condition notFull notEmpty分别对应putLock和takeLock；默认最大容量是Integer.MAX_VALUE，如果count已满，await



#### [SynchronousQueue：](E:\SynchronousQueue：.md)本身不存储元素，put的时候一直等到由消费者取走元素，否则一直阻塞。

内部的waitQueue、qLock，只用在Queue本身实例的序列化和反序列化，和具体1.8的实现无关。



#### PriorityBlockingQueue：感觉和ArrayBlockingQueue很类似，同样只有一把全局锁，同样内部是用的Object数组存储元素；但是这个队列元素是具有优先级的，且当数组放满，是可以arraycopy增大数组长度的。







Fork/Join框架源码解读 & 该框架哪里用到了工作窃取 &invokeAll&fork区别 & 怎么调试到调用compute方法