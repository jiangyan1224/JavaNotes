#### Volatile特性补充以及syn和Lock保证可见性

#### 首先，syn块/Lock块也是可以保证可见性的

[伪共享问题](E:\ConcurrentHashMap(volatile伪共享问题)&ConcurrentLinkedQueue：.md)

volatile的可见性以及volatile修饰引用类型：

https://stackoverflow.com/questions/5173614/java-volatile-array

https://stackoverflow.com/questions/53753792/java-volatile-array-my-test-results-do-not-match-the-expectations

https://segmentfault.com/q/1010000022516112?_ea=42109409

https://segmentfault.com/q/1010000017341320

关于这种情况其实我也没有自己找到官方的解释，通过StackOverflow segmentfault等网站看的解释：

总结：

**1、volatile修饰引用类型时，本身只能保证引用的可见性，并不能保证对象内部值的可见性；**

**2、一个线程在访问了一个volatile变量之后，所有该线程可见的变量，不管是引用类型还是基本类型，该线程在读的时候都需要去主存更新最新值。**

https://segmentfault.com/q/1010000022516112?_ea=42109409这个链接中，instance的可见性，其实syn块就可以保证（后面说一下syn和lock如何保证可见性），那么这个volatile的作用，我认为是要禁止初始化指令的重排序：如果不用volatile修饰instance，在线程Anew实例的时候，指令重排序让instance引用指向了一个还没有初始化完成的对象，如果此时线程B使用了instance引用（并不一定是跟A一样，进入getInstance方法，可能是自己直接拿引用来使用），就会出问题

对于总结的第2点，只要某一个线程访问了一个volatile，那么在这个线程内部访问的所有变量，不管是基本类型还是引用类型内部属性值，只要这些变量对这个线程可见，都要去主存更新最新值https://stackoverflow.com/questions/53753792/java-volatile-array-my-test-results-do-not-match-the-expectations

但是，在没有涉及到volatile变量时的可见性，我就很不懂了，感觉很玄学，比如：

在下面代码中，如果没有 `System.out.println("waiting");`这一行的输出，线程B永远无法知道共享变量ints内部值的变化；但是，如果有了这一行标准输出（或者sleep也可以），B就可以得知变化；

理由未知。

```java
public class Test {
    public static int[] ints = new int[5];
    public static void main(String[] args) throws Exception {
        Object o = new Object();
        new Thread(() -> {
            //线程A
            try {
                TimeUnit.MILLISECONDS.sleep(100);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            ints[0] = 2;
        }).start();
        new Thread(() -> {            //线程B
            while (true) {
                if (ints[0] == 2) {
                    System.out.println("结束");
                    break;
                }
                System.out.println("waiting");//这一行的输出
                //或者不用标准输出，而是在这里sleep一会，也有相同效果
            }
        }).start();
    }
}
```

#### synchronized可见性保证：

http://www.cs.umd.edu/~pugh/java/memoryModel/jsr-133-faq.html#incorrectlySync

JSR的happens-before中有这么一条原则：

`对一个监视器的解锁操作happens-before于每一个后续对同一个监视器的加锁操作。`

happens-before实际上规定了操作之间的内存可见性，A操作happens-beforeB操作，那么A操作的执行结果（比如变量值的修改）一定对B操作可见

而对于synchronized同一个监视器来说，对同一个监视器的unlock一定happens-before同一个监视器的lock；也就是说，某个线程A在unlock之前的执行结果（变量值的修改），对另一个线程B的lock操作，一定是可见的：B拿到锁之后，会清空其工作内存中值，去主存更新；写入之后，在退出syn块之前，会强制把对内存的写入，刷新到主存。

*But there is more to synchronization than mutual exclusion. Synchronization ensures that memory writes by a thread before or during a synchronized block are made visible in a predictable manner to other threads which synchronize on the same monitor. After we exit a synchronized block, we **release** the monitor, which has the effect of flushing the cache to main memory, so that writes made by this thread can be visible to other threads. Before we can enter a synchronized block, we **acquire** the monitor, which has the effect of invalidating the local processor cache so that variables will be reloaded from main memory. We will then be able to see all of the writes made visible by the previous release.*

*但是同步不仅仅是相互排斥。**同步确保以可预测的方式使线程在同步块之前或期间对内存的写入对于在同一监视器上同步的其他线程可见。退出同步块后，我们释放监视器，其作用是将缓存刷新到主内存，以便该线程进行的写入对其他线程可见。在进入同步块之前，我们需要获取监视器，该监视器具有使本地处理器缓存无效的作用，以便可以从主内存中重新加载变量**。然后，我们将能够看到以前版本中所有可见的写入。*

***Important Note:*** *Note that it is important for both threads to synchronize on the same monitor in order to set up the happens-before relationship properly. It is not the case that everything visible to thread A when it synchronizes on object X becomes visible to thread B after it synchronizes on object Y. The release and acquire have to "match" (i.e., be performed on the same monitor) to have the right semantics. Otherwise, the code has a data race.*

*请注意，两个线程必须在同一监视器上同步，以便正确设置事前发生关系。并非如此，线程A在对象X上同步后，在线程X上与对象X同步时所有可见的东西都变为线程B可见。释放和获取必须“匹配”（即，在同一监视器上执行）才能具有正确的语义。否则，代码将发生数据争用。*

**所以，在同一个监视器上，可以保证变量可见性；不是同一个监视器，则不能保证这一点。**

#### Lock的可见性保证：

reentrantlock如何保证可见性？ - Forest Wang的回答 - 知乎 https://www.zhihu.com/question/41016480/answer/551056899

`对volatile字段的写入操作happens-before于每一个后续的对同一个volatile字段的读操作。`

**理解1：**其实这一点我觉得不看上面知乎的链接，只根据前面斜体字对于happens-before的理解和volatile的happens-before原则，以及Lock中volatile的使用，就可以知道如何保证可见性了。

AQS中，共享变量state是用volatile修饰的，而在每次lock和unlock都会涉及到对volatile的读写；和synchronized关键字类似的：在线程A进行unlock时，unlock涉及到volatile的写操作；另一个线程在lock，对volatile读的时候，由于volatile的happens-before，**A在unlock之前的所有写操作，都对B可见**

**理解2：**或者，根据前面对volatile可见性的总结第二点，也可以这么理解：

不管怎样，每次lock都需要访问volatile，一旦访问了volatile变量，那么该线程能见到的所有变量值，都需要去主存更新最新值。