interrupt&suspend&resume&stop：

都是为了中断/终止线程运行：

**interrupt：**对于正常运行的线程，对其做interrupt操作，只会让它设置其中断标志位，表明它被中断过，但是不会影响到它的继续运行；

对于处于wait sleep nioIO等阻塞状态下的线程：如果对它调用interrupt，会导致该线程接收到Interrupt Exception，提前退出阻塞状态，并清除其中断标志位（清除之后值为false）

`在Core Java中有这样一句话：" 没有任何语言方面的需求要求一个被中断的程序应该终止。中断一个线程只是为了引起该线程的注意，被中断线程可以决定如何应对中断 "。`

`如果线程被阻塞，它便不能核查共享变量，也就不能停止。这在许多情况下会发生，例如调用Object.wait()、ServerSocket.accept()和DatagramSocket.receive()时，他们都可能永久的阻塞线程。即使发生超时，在超时期满之前持续等待也是不可行和不适当的，所以，要使用某种机制使得线程更早地退出被阻塞的状态。很不幸运，不存在这样一种机制对所有的情况都适用，但是，根据情况不同却可以使用特定的技术。使用Thread.interrupt()中断线程正如Example1中所描述的，Thread.interrupt()方法不会中断一个正在运行的线程。这一方法实际上完成的是，在线程受到阻塞时抛出一个中断信号，这样线程就得以退出阻塞的状态。更确切的说，如果线程被Object.wait, Thread.join和Thread.sleep三种方法之一阻塞，那么，它将接收到一个中断异常（InterruptedException），从而提早地终结被阻塞状态。`

**suspend&resume&stop：**均已不推荐使用

suspend导致线程在持有锁的情况下进入睡眠状态，且别人不对它resume，该线程就无法苏醒

stop导致线程立即停止，如果该线程是处于同步代码块内的，会导致该线程没有完全执行完同步代码块就退出，破坏了原子性，同步代码块上的锁就没有意义了

