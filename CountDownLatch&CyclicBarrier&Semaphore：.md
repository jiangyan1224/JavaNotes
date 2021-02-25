#### CountDownLatch：

其实相当于高级版的join：可以让某个线程或者某些线程等待其他线程完成一些事情并调用countDownLatch.countDown()，当CountDownLatch计数器降为0，被CountDownLatch阻塞的线程们就可以继续往下执行。

#### CyclicBarrier：

和CountDownLatch感觉比较类似，区别在于：CountDownLatch感觉更像是某个线程被阻塞，等待其他线程countdown为0之后，被阻塞的线程就可以继续执行；

而CyclicBarrier是自己阻塞自己，阻塞N个线程，只有N个线程都完成某些任务，都调用await，这N个线程才能继续往下执行。

CyclicBarrier的计数器是可以reset重置的（某个线程重置会影响到其他正在await的线程，需要再次同步所有线程。），且CyclicBarrier的构造方法参数支持在所有线程调用await之后，优先先执行某个Runnable，再让所有线程继续往下执行。

#### Semaphore：

限制同时获取共享资源的线程个数，同一时间最多允许N个线程进入共享资源。可做流量控制。