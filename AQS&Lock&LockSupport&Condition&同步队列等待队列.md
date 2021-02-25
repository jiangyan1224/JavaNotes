### Java中的Lock：

ReentrantLock继承了Lock接口，就要实现Lock接口里的方法，实现加锁解锁的逻辑；而这些加锁解锁的逻辑实现，又可以通过内置一个实现了AQS抽象类的静态内部类，重写AQS方法，简化实现

AQS子类实现上，有些方法可以直接用（模板方法），比如lock,lock()可以直接调用AQS的acquire()；但是acquire内部调用的方法tryAcquire没有具体实现，就需要子类重写自己实现
**AQS同步队列中，首节点代表的线程一定是已经获取到了同步锁的；但是这个首节点在下一个节点拿到锁之前还会一直停留，不被移出同步队列**

AQS同步队列中，只有前驱节点为头节点的，才能尝试获取锁

<font color ="orang">对于共享锁：首节点A拿到锁之后，调用**setHeadAndPropagate，如果还有剩余资源，内部调用unparkSuccessor**，唤醒后继节点B；如果B唤醒之后也拿到了锁，同样调用**setHeadAndPropagate**，如果仍有剩余资源，继续唤醒B的后续节点C......</font>

<font color ="orang">对于独占锁：首节点A拿到锁之后，直接返回。但是在A用完释放锁之后，会调用unparkSuccessor，唤醒后继节点B。</font>

**总之：不管是独占锁还是共享锁，一旦线程进入同步队列，就只能FIFO，节点的唤醒一定是从前到后依次唤醒；但是在进入同步队列之前，线程第一次调用lock() -> acquire()时：对acquire的实现不同，就可以实现公平或者非公平。**以ReentrantLock中的FairSync和NonFairSync为例，这两个AQS子类的lock方法都调用的是 `acquire(1);`acquire方法也都是调用AQS自己的模板实现：

```java
//AQS模板方法acquire
public final void acquire(int arg) {
        if (!tryAcquire(arg) &&
            acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
            selfInterrupt();
    }
```

但是内部，tryAcquire的实现，在FairSync和NonFairSync中是不同的：

FairSync：

```java
//FairSync内部的tryAcquire实现：
protected final boolean tryAcquire(int acquires) {
            final Thread current = Thread.currentThread();
            int c = getState();
            if (c == 0) {
                if (!hasQueuedPredecessors() &&
                    compareAndSetState(0, acquires)) {
                    setExclusiveOwnerThread(current);
                    return true;
                }
            }
            else if (current == getExclusiveOwnerThread()) {
                int nextc = c + acquires;
                if (nextc < 0)
                    throw new Error("Maximum lock count exceeded");
                setState(nextc);
                return true;
            }
            return false;
        }
```

NonFairSync：

```java
//NonFairSync内部的tryAcquire：从名字就能看出来是用的不同实现
protected final boolean tryAcquire(int acquires) {
            return nonfairTryAcquire(acquires);
        }
```



tryAcquire tryAcquireShared tryRelease tryReleaseSahred这些方法在AQS中都没有具体实现，而这些方法也可以看出对应的是独占式还是共享式锁；自定义同步器的时候，除了这四个方法，主要就还有一个isHeldExclusively还需要自己实现：

`不同的自定义同步器争用共享资源的方式也不同。自定义同步器在实现时只需要实现共享资源state的获取与释放方式即可，至于具体线程等待队列的维护（如获取资源失败入队/唤醒出队等），AQS已经在顶层实现好了。自定义同步器实现时主要实现以下几种方法：`

- `isHeldExclusively()：该线程是否正在独占资源。只有用到condition才需要去实现它。`
- `tryAcquire(int)：独占方式。尝试获取资源，成功则返回true，失败则返回false。`
- `tryRelease(int)：独占方式。尝试释放资源，成功则返回true，失败则返回false。`
- `tryAcquireShared(int)：共享方式。尝试获取资源。负数表示失败；0表示成功，但没有剩余可用资源；正数表示成功，且有剩余资源。`
- `tryReleaseShared(int)：共享方式。尝试释放资源，如果释放后允许唤醒后续等待结点返回true，否则返回false。`

[链接1](https://www.cnblogs.com/waterystone/p/4920797.html)

[链接2](https://www.cnblogs.com/chengxiao/archive/2017/07/24/7141160.html)

AQS中，线程在**第一次等待获取锁的时候（lock、lockInterruptibly、tryLock。。。）**，可以使用`tryLock(long time, TimeUnit unit)`**设置等待最大时间**；或者使用`void lockInterruptibly()`让等待的过程可以**响应中断**。（`tryLock(long time, TimeUnit unit)`也可以响应中断）

这些实现和上面提到的acquire不同：比如lockInterruptibly（）调用的是acquireInterruptibly：和lock（）调用的acquire方法是不同的。

```java
public final void acquireInterruptibly(int arg)
            throws InterruptedException {
        if (Thread.interrupted())
            throw new InterruptedException();
        if (!tryAcquire(arg))
            doAcquireInterruptibly(arg);
    }
```



##### ReentrantReadWriteLock：

构造这个读写锁的时候，会根据构造函数参数新建一个sync、一个readLock和一个writeLock，两个lock使用同一个同步器sync（sync随构造函数参数决定它是一个FairSync还是NonFairSync）。

sync内部的state同步状态也是两个lock共用：高16位为读状态；低16位为写状态

任何时候，ReentrantReadWriteLock中state的值，一定是当前状态下，当前读锁已经被获取且还没有被释放的个数；以及当前写锁已经被获取还没有被释放的个数；对于读锁，当前持有读锁的线程们内部，各自的ThreadLocal还存储了各自线程当前持有并且还没有释放的读锁个数

一个线程在试图获取写锁时，一定不能有其他线程已获取读锁；且在某个线程获取到写锁之后，其他线程不能再读/写，但是这个已经拿到写锁的线程，可以继续获取写锁/读锁



#### LockSupport：

提供了park unpark等函数，方便阻塞和唤醒一个线程。

`LockSupport.park(this);`、`LockSupport.unpark(threadA)`

[park&unpark和wait&notify不同：](https://www.cnblogs.com/dennyzhangdd/p/7281150.html)

wait&notify需要线程间通过某一个Object产生关联，控制唤醒/阻塞，且notify时如果目标线程并没有阻塞，就发挥不了作用，后面目标线程一旦wait就只能阻塞，除非再次notify；

park&unpark则不需要这个Object，它通过Java线程内部的Parker类作用，设置其中的_counter变量，标志这个线程是否拿到“许可”；且unpark可以在park之前调用也有效：如果B线程先unpark(A)三次，然后A被park，A会直接消耗掉所有许可，继续运行（如果在此之后A又被park，之前B的unpark已经没用了，A会被阻塞）

park也可以传入参数（Object blocker），据说是为了在dump线程的时候可以像synchronized那样有更多的阻塞对象的信息，相比无参park，有参park的内部实现多了一个setBlocker(blocker)，其他部分都是一样的：Unsafe.unpark，所以暂时不知道blocker具体有啥作用。

被park的线程处于WAITING状态，dump出来会标明是被park引起的

park也可以响应中断，**但是park被中断之后，不会抛出InterruptedException，而是默默返回继续执行。**可以获取线程的中断标记看它是不是被中断过。

#### Condition：

Condition的使用感觉和wait很类似：[wait()和notify()](E:\Join()和wait()和notify().md)

**在调用await和wait（或者signal和notify）之前都需要先获取锁，只不过Condition中是要获取对应Lock的锁；wait是要获取wait对应Object的锁**

当前线程调用了await/wait都会释放对应的锁，进入阻塞状态，当前线程被包装成节点，进入等待队列

**某个线程A调用了某个Condition/某个Object的signal/notify之后，都会唤醒被阻塞的线程；但是如果当前线程A还没有执行完、还没有释放自己持有的Lock/Object锁，被唤醒的线程是先退出等待队列，进入同步队列，在重新获取Lock之前无法继续从之前被阻塞的地方恢复执行。**

```java
public class ConditionTest {
    static Lock lock = new ReentrantLock();
    static Condition condition = lock.newCondition();
    public static void main(String[] args){
        new Thread(new Runnable() {
            @Override
            public void run() {
                lock.lock();//同步队列
                try {
                    System.out.println("Thread1 entering await");
                    condition.await();//等待队列
                    System.out.println("Thread1 exiting await");
                }catch (Exception e){
                    e.printStackTrace();
                }finally {
                    lock.unlock();
                }
            }
        }).start();

        new Thread(new Runnable() {
            @Override
            public void run() {
                lock.lock();
                try {
                    System.out.println("Thread2 get lock");
                    condition.signal();
                    System.out.println("Thread2 signaled!");
                    Thread.sleep(5000);
                }catch (Exception e){
                    e.printStackTrace();
                }finally {
                    lock.unlock();
                }
            }
        }).start();
    }
}
```



#### 同步队列&等待队列：

在Node获取到锁之后，Node是先set状态值，成为同步队列的首节点，直到后面一个节点也拿到资源，就会被移出同步队列；

在对应Lock.signal之后，被唤醒线程所在的Node是直接被移出等待队列，CAS地移动到同步队列尾，被unpark之后，然后就可以参与和其他线程一起竞争，但是在拿到锁之前是一直在同步队列的。