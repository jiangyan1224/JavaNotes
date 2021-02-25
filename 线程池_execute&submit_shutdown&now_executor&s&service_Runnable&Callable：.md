#### 线程池：

提交一个任务：

假设当前正在运行的线程数量为N：（笔者自己实验，觉得也是当前线程池中还活着的线程，**就算一个线程因为当前没有任务可做而陷入阻塞，它也算是正在运行的线程。**）

```java
package ch8;

import java.util.concurrent.*;

public class ExchangerTest {
    private static final Exchanger<String> exchanger = new Exchanger<>();
    private static Object object = new Object();
    public static void main(String[] args) throws InterruptedException {
        ThreadPoolExecutor pool1 = new ThreadPoolExecutor(2,4,2000,
                TimeUnit.MILLISECONDS,new ArrayBlockingQueue<>(10));
        pool1.submit(new Runnable() {
            @Override
            public void run() {
                synchronized (object){
                    try {
                        object.wait();
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                        System.out.println("Thread1 been interrupted");
                    }
                }

            }
        });
        pool1.prestartCoreThread();
        Thread.currentThread().sleep(10000);
//        pool1.shutdown();//interrupt的是所有正在等待任务的线程
        System.out.println(pool1.getQueue().size());
        pool1.submit(new Runnable() {//这个任务提交之后，应该会把正在等待任务的那个线程也算进运行的线程，所以会进入到阻塞队列
            @Override
            public void run() {
                String B = "money B";
                System.out.println(Thread.currentThread().getName());

            }
        });
        System.out.println(pool1.getQueue().size());
//        pool1.shutdownNow();//interrupt的是所有线程
        pool1.shutdown();
        System.out.println(pool1.getQueue().size());
        //线程池中运行的线程，是包括正在等待任务的线程的
    }


}

```

当线程池的线程没有任务陷入阻塞，不是阻塞在线程的run本身，而是在尝试从队列中取任务的take方法或者poll超时方法上！！！！！

**（核心线程默认会一直阻塞在take上，而非核心线程在poll上超时阻塞，最多等待keepAliveTime之后退出，如果还没有拿到任务，被销毁掉）**

所以，没有任务陷入阻塞的核心线程，当然是运行中的线程

```java
final void runWorker(Worker w) {
    Thread wt = Thread.currentThread();
    Runnable task = w.firstTask;
    w.firstTask = null;
    w.unlock(); // allow interrupts
    boolean completedAbruptly = true;
    try {
        //阻塞在这个getTask上！！！！！！！！！！！！
        while (task != null || (task = getTask()) != null) {...}
        completedAbruptly = false;
    } finally {
        processWorkerExit(w, completedAbruptly);//从workers中remove当前worker，真正删除了一个线程
    }
}

getTask()节选：
try {
     Runnable r = timed ?
     workQueue.poll(keepAliveTime, TimeUnit.NANOSECONDS) ://超时阻塞，最多等待keepAliveTime之后退出！！！！！！！！！
     workQueue.take();//阻塞
     if (r != null)
     return r;
     timedOut = true;
    } catch (InterruptedException retry) {
       timedOut = false;
}
```

1、如果当前N < 核心线程数：获取全局锁，创建一个新核心线程，把刚刚提交的任务作为firstTask执行

2、否则，尝试把任务丢进阻塞队列

3、如果进入阻塞队列失败，获取全局锁，新建一个非核心线程

4、如果新建非核心线程失败（可能是因为当前N已经大于等于最大线程数了），那么执行四种拒绝策略之一



**execute&submit：**

submit返回一个future实例，可以对这个future调用get方法，get方法会阻塞当前线程直到任务完成。

execute只接受Runnable，返回void；submit可以接收Runnable/Callable，返回Future实例

**shutdown&shutdownNow：**

都是对线程池中的线程调用interrupt：但是shutdown作用的是所有正在等待任务的线程；shutdownNow作用的是所有线程（当然，正在执行任务的线程，有可能压根就不会响应中断）

**Executor&Executors&ExecutorService：**

Executor：接口，只声明了execute方法

ExecutorService：接口，继承并拓展了Executor接口，增加了shutdown、submit等方法

Executors：没有继承或者实现任何接口/类，是一个工厂类，内含静态方法，可用来创建Fixed/Singled/Scheduled/Cached ThreadPool（这些静态方法的实现内部，其实也是直接new了一个ThreadPoolExecutor/ScheduledThreadPoolExecutor返回，这些线程池可以看作是ThreadPoolExecutor 和 ScheduledThreadPoolExecutor的特例），也可用于Runnable到Callable的转换。

ThreadPoolExecutor 和 ScheduledThreadPoolExecutor均实现了ExecutorService接口。

**Runnable&Callable：**

对于Runnable，由于run方法本身没有返回值，就算用submit和Future.get尝试获取结果，得到的也是null；

对于Callable，可以有返回值，也可以throws异常，可以用submit和Future获取到返回结果

Executors.callable()可以把Runnable -> Callable：[🔗](https://www.zhihu.com/question/36138490/answer/343258744)

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
            return result;
        }
}
```

所以在Runnable ->Callable的时候，传入的result是什么结果，Future.get得到的是完全一致的东西。至于这个参数有啥用，源代码注释：

```java
    /**
     * Returns a {@link Callable} object that, when
     * called, runs the given task and returns the given result.  This
     * can be useful when applying methods requiring a
     * {@code Callable} to an otherwise resultless action.
     * @param task the task to run
     * @param result the result to return
     * @param <T> the type of the result
     * @return a callable object
     * @throws NullPointerException if task null
     */
    public static <T> Callable<T> callable(Runnable task, T result) {
        if (task == null)
            throw new NullPointerException();
        return new RunnableAdapter<T>(task, result);
    }
```

我也没搞懂到底是啥意思，可能是为了对不同的结果作区分？？？