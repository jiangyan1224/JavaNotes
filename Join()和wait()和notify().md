Join()和wait()和notify()：

```java
public class JoinDemo extends Thread{
    int i;
    Thread previousThread; //上一个线程 main
    public JoinDemo(Thread previousThread,int i){
        this.previousThread=previousThread;
        this.i=i;
        this.setName("join"+i);
    }
    @Override
    public void run() {
        try {
            previousThread.join();//main.join()  join1获取到main线程的锁，
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        System.out.println("num:"+i);
    }
    public static void main(String[] args) {
        Thread previousThread=Thread.currentThread();
        for(int i=0;i<10;i++){
            JoinDemo joinDemo=new JoinDemo(previousThread,i);
            joinDemo.start();
            previousThread=joinDemo;
        }
    }
}
```

这个代码使用join可以实现顺序执行，输出num0~num9，递增顺序。

以main线程创建join1线程，join1线程执行到`previousThread.join();`这一行为例：

这里的previousThread是main线程，即join1线程执行到了main.join（），join内部：

```java
    public final synchronized void join(long millis)
    throws InterruptedException {//this是main，synchronized关键字说明这个方法的执行需要先拿到this锁：即join1线程要执行main.join就需要join1线程拿到main线程的锁
        long base = System.currentTimeMillis();
        long now = 0;

        if (millis < 0) {
            throw new IllegalArgumentException("timeout value is negative");
        }

        if (millis == 0) {
            while (isAlive()) {//this.isAlive()  ->  main.isAlive()
                wait(0);//this.wait()  -> main.wait()
            }
        } else {
            while (isAlive()) {
                long delay = millis - now;
                if (delay <= 0) {
                    break;
                }
                wait(delay);
                now = System.currentTimeMillis() - base;
            }
        }
    }
```

对于其中的main.wait（）：

jdk中对wait方法的注释：

<font color ="red">**线程T调用了某个Object的wait方法（T必须先持有这个Object的monitor），就会导致T进入阻塞**</font>，直到其他线程调用了这个Object的notify或者这个Object的notifyAll方法，或者超时，或者T被interrupt（这个时候会抛出InterruptedException）

<img src="C:\Users\123\Pictures\JVM\wait.png" style="zoom:67%;" />

在这里的场景，就是指：join1线程获取了main线程的锁，调用了main.wait()，join1线程被阻塞住了；main继续执行，创建join2，join2线程获取了join1线程的锁，同理被阻塞住。。。直到main创建完所有线程执行完毕，main线程退出的时候，会唤醒所有等待main对象的线程，导致join1被唤醒，紧接着join1执行完退出的时候，唤醒等待join1的join2线程，join2继续执行。。。直到所有线程执行完毕。

Java线程结束执行退出时的源码：

```c++
void JavaThread::exit(bool destroy_vm, ExitType exit_type) {
  assert(this == JavaThread::current(),  "thread consistency check");
  ...
  // Notify waiters on thread object. This has to be done after exit() is called
  // on the thread (if the thread is the last thread in a daemon ThreadGroup the
  // group should have the destroyed bit set before waiters are notified).
  ensure_join(this); 
  assert(!this->has_pending_exception(), "ensure_join should have cleared");
  ...

```

```c++
static void ensure_join(JavaThread* thread) {
  // We do not need to grap the Threads_lock, since we are operating on ourself.
  Handle threadObj(thread, thread->threadObj());
  assert(threadObj.not_null(), "java thread object must exist");
  ObjectLocker lock(threadObj, thread);
  // Ignore pending exception (ThreadDeath), since we are exiting anyway
  thread->clear_pending_exception();
  // Thread is exiting. So set thread_status field in  java.lang.Thread class to TERMINATED.
  java_lang_Thread::set_thread_status(threadObj(), java_lang_Thread::TERMINATED);
  // Clear the native thread instance - this makes isAlive return false and allows the join()
  // to complete once we've done the notify_all below
  //这里是清除native线程，这个操作会导致isAlive()方法返回false 导致当前线程(main)不再isAlive
  java_lang_Thread::set_thread(threadObj(), NULL);
  lock.notify_all(thread);///////注意这里，唤醒所有等待当前线程对象的线程们////////////////////
  // Ignore pending exception (ThreadDeath), since we are exiting anyway
  thread->clear_pending_exception();
}
```

总结：线程A调用了线程B.join（）：就会阻塞A线程，A线程加入B的waitSet；直到B执行完毕退出（或者直到别的线程调用B.notifyAll）

相当于A线程把B线程拉到了自己前面，B不执行完，A就阻塞



wait()和notify()：

首先，要调用**某个Object**的wait/notify，当前线程必须先获取**该Object**的锁

如果某个线程A在获取锁，进入同步块内之后，**调用了该对象的wait之后，进入该Object的waitSet，**只能等待其他线程B调用该Object的notify唤醒自己；但是A醒了之后，如果B线程还没有结束，B依然持有Object锁，**A处于entrySet等待获取锁**。**只有等B结束执行释放锁之后，JVM从entrySet中选一个作为新的owner，owner得到锁并执行同步代码块。** **如果A获取到锁，就回到Await的地方继续执行。（有点类似于A线程在重新获取锁之后被恢复现场，直接到之前被中断的地方继续执行）**

```java
public class WaitTest {
    private static final Object obj = new Object();
    private static final Object obj1 = new Object();
    public static void main(String[] args){
        new Thread(new Runnable() {
            @Override
            public void run() {
                synchronized (obj){
                    try {
//                        obj1.wait();//运行报错IllegalMonitorStateException
                        System.out.println("Thread 1 enter wait");
                        obj.wait();
                        Thread.sleep(1000);
                        System.out.println("Thread 1 从wait处离开");
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                }
            }
        }).start();

        new Thread(new Runnable() {
            @Override
            public void run() {
                synchronized (obj){
                    System.out.println("Thread3 enter syn");
                    obj.notifyAll();
                    System.out.println("Thread3 notify");
                    try {
                        Thread.sleep(5000);
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                    System.out.println("Thread3 finally exit");
                }
            }
        }).start();
    }
}
```

