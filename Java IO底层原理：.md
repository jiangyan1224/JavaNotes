### Java IO底层原理：（Linux中）

https://www.cnblogs.com/crazymakercircle/p/10225159.html

#### 1.1 IO读写原理

不管是Socket读写/文件读写，在Java层面的应用开发/Linux系统底层开发，都属于对输入input和输出output的处理，简称为IO读写。在原理和处理流程上都是一致的，区别在于参数的不同。

用户程序进行IO读写，基本上会用到read&write两大系统调用，不同操作系统，对应的名称可能不同，但是功能大致一样。

IO整体目标：把数据从物理设备（如磁盘）读到最终目标---内存（如进程内存）；把数据从内存（如进程内存）写入到物理设备（如磁盘）。

而read和write系统调用本身只负责**内核缓冲区和进程缓冲区之间**的数据复制；底层的读写交换，是由内核自己完成的。

![](C:\Users\123\Pictures\JVM\read&write.png)

##### 1.1.1内核缓冲区和进程缓冲区（Linux中，内核有内核缓冲区，每个进程有自己独立的进程缓冲区）

缓冲区的目的：是为了减少频繁的系统IO调用。（进程在发起系统调用之后，进程状态要从用户态转为内核态；而原来用户态的信息就需要保存，在后面恢复到用户态的时候会用到。）

https://blog.csdn.net/Zong__Zong/article/details/76460899

> **系统调用时的当前进程要进行模式的切换**，要从用户态切换到内核态，那么用户态的一些寄存器信息，就要进行压栈的操作，**压入内核栈**，以免下次进行现场恢复（恢复到用户空间）的时候，直接从内核栈弹出寄存器的信息，并让eax寄存器带上内核完成系统调用时的结果；

可以看出，发起系统调用是一个比较损耗时间和性能的操作，为了尽量减少这个操作，出现了缓冲区：

有了缓冲区之后，read和write函数大部分时候是在读写进程自己的进程缓冲区，当缓冲区数据的量达到一定上下阈值，就需要真正的发起一次系统调用。

##### 1.1.2一次Java服务端处理网络请求的典型过程：

![](C:\Users\123\Pictures\JVM\SocketIO.png)

1.客户端请求：Linux通过网卡读取客户端的请求数据，读取到内核缓冲区

2.获取请求数据：服务端从内核缓冲区读数据到Java进程缓冲区

3.服务端业务处理：服务端在自己的进程缓冲区中处理客户请求

4.服务端返回响应数据：服务端把响应数据写入到内核缓冲区

5.发送到网卡：Linux把内核缓冲区的数据写入到网卡，网卡通过底层通信协议，把数据发回给目标客户端。

#### 1.2四种常见的IO模型（其中的名字和Java的BIO NIO AIO不是完全对应）

首先说一下自己对阻塞&非阻塞，同步&异步的理解：（在百度/谷歌上搜了一下，但是没有找到能完全说服我的说法）

阻塞&非阻塞：比如我发起了一个请求，但是这个时候对方还没有准备好我要的数据：如果我陷入了阻塞，直到数据准备好，我才返回-----这叫阻塞；如果我带着没有数据的失败结果直接返回，然后开始重试，一直发送请求，返回失败结果，直到对方准备好数据，我的请求得到了成功结果，我才返回----这叫非阻塞。（即，阻塞和非阻塞的区别在于，如果要请求的数据还没准备好，在准备好之前，我是直接进入阻塞，还是不阻塞，做一些事情，比如重试请求。）

同步&异步：比如我发起了一个请求，同步：我一直在等这个结果，（等的期间，我可以阻塞，也可以非阻塞一直重试）等到结果再返回。异步：我发起请求之后立即返回去做其他事情，请求的结果由对方通过信号/回调等方式通知我。

##### 1.2.1同步阻塞IO（Blocking IO）

![BlockingIO](C:\Users\123\Pictures\JVM\BlockingIO.jpg)

1.用户线程发起了read系统调用，内核开始等待数据准备好，（当数据到达并拷贝进内核缓冲区之后，数据就准备好了），此时用户线程因为没拿到数据，进入阻塞。

2.内核开始把数据从内核缓冲区，拷贝到用户缓冲区（用户内存），拷贝完成之后，返回结果

3.用户线程退出阻塞状态，重新运行，从用户缓冲区读取数据。

Blocking IO的特点：用户线程在内核准备数据和复制数据的两个阶段，都进入阻塞。

但是，这种情况下，一般每个客户端连接都有单独的线程维持读写，一旦有大量的客户端，线程的切换等开销会很大。

##### 1.2.2同步非阻塞NIO（None Blocking IO，跟Java的NIO不一样！！！）

![](C:\Users\123\Pictures\JVM\NoneBlockingIO.jpg)

1.用户线程发起了read系统调用，此时内核缓冲区还没有准备好数据，用户线程立即返回，返回一个失败结果。

2.用户线程不断重试，重试发起read系统调用。

3.直到某一时刻，内核缓冲区数据准备完毕。**用户线程进入阻塞。**内核开始把内核缓冲区的数据拷贝到用户缓冲区，然后返回结果。

4.用户线程退出阻塞状态，重新运行，读取用户缓冲区的数据

None Blocking IO特点：用户线程需要发起多次系统调用，只要当前内核还没有准备好数据，用户线程就立即返回；如果内核准备好了数据，用户线程就进入阻塞，直到内核复制数据完成。

但是None Blocking IO需要用户线程不断地重复发起系统调用，不断地对内核轮询，占用大量的CPU时间。也会造成大量性能消耗。

##### 1.2.3IO多路复用模型（IO multiplexing，这个才是Java的NIO，又称New IO）

https://www.jianshu.com/p/4543c92b2fbd

###### Java NIO下，`selector = SelectorProvider.provider().openSelector();`这一行，会根据当前系统，返回不同OS对于NIO的实现，即不同选择器selector的实现 --> 在使用selector.select() --> 发起各OS对于NIO实现的系统调用

Windows下：

[关于Java nio在Windows下的实现](https://cloud.tencent.com/developer/article/1432124)

> -  **jdk8和以前，java nio的windows实现，在底层是基于winsock2的select**。
> - 但是**winsock2的select是否是基于轮询的，是不是我们常说的select/poll/epoll中的select，我无法查证，毕竟windows不是开源的**。如果是轮询，那效率是相当低的。所以说windows就这点不好>_<。
> - 一次select可返回的最大数量是1024。

Linux下：

select/poll/epoll 都是Linux对 I/O 多路复用的具体实现，从select -> poll -> epoll改进，后面补充三者的区别。

![](C:\Users\123\Pictures\JVM\IOmultiplexing.jpg)

以Linux下的IO多路复用为例：

1.选择器（xx线程）发起select/poll/epoll系统调用，查询就绪连接。（有可能会让线程进入阻塞）当内核有准备好的数据，就返回

2.用户线程获得目标连接，再向内核发起read系统调用，用户线程被阻塞，直到内核复制数据完成，把数据从内核缓冲区复制到用户缓冲区

3.用户线程退出阻塞，重新运行，读取用户缓冲区的数据。

IO multiplexing的特点：

IO多路复用模型建立在OS内核本身能够提供多路分离系统调用（如select poll epoll）

IO多路复用要用到两个系统调用：一个是查询调用（如select poll epoll），一个是IO的读取调用

Java的NIO实现中，一般都会设置连接为非阻塞（不然在注册事件的时候会报错）

IO multiplexing的优点在于，可以使用少量线程就可以管理成千上万的连接（connection），获取到连接之后，对应的处理可以直接扔给线程池，由线程池内部使用线程处理。

https://github.com/CyC2018/CS-Notes/issues/194

**但是IO multiplexing并不是异步IO，Linux的select poll epoll都是同步的，在调用时可以根据指定参数来决定阻塞/非阻塞。（AIO是异步的）且在进行read系统调用时，用户线程是阻塞的，直到内核复制数据完成，才退出阻塞。**

##### 1.2.3异步IO模型（asynchronous IO，又叫AIO）

AIO是完全的异步

![](C:\Users\123\Pictures\JVM\AIO.jpg)

1.用户线程发起read系统调用，立刻返回，就可以做其他事情，用户线程不阻塞。

2.内核开始准备数据，准备好之后，开始复制数据，即把内核缓冲区的数据复制到用户缓冲区

3.复制完成，会通过某种方式通知用户线程，（信号/回调函数）告诉用户线程read操作完成了

4.用户线程读取用户缓冲区的数据。

AIO特点：

不管是准备数据还是复制数据阶段，用户线程都没有阻塞。数据复制完成之后，由内核通知用户线程。

目前，Windows下对AIO的支持有IOCP；而Linux下还没有成熟的实现。

##### 1.2.4信号驱动IO（简单扯一句）

原文里说异步IO又叫信号驱动IO，我觉得应该还是不一样：

信号驱动IO和异步IO类似，在内核数据准备阶段，用户线程都是发起系统调用之后就直接返回，非阻塞；但是在内核完成数据准备之后，信号驱动IO中，内核会通知用户线程，可以开始数据复制，并且这个阶段用户线程会进入阻塞；

而在异步IO中，内核是在结束数据复制之后，再通知用户线程，用户线程完全没有阻塞。

相当于：异步IO中，内核是通知用户线程去用户缓冲区读数据；信号驱动IO中，内核通知用户线程到内核缓冲区读数据，并且读的过程中用户线程阻塞。

https://blog.csdn.net/uestcprince/article/details/90734564

![](C:\Users\123\Pictures\JVM\Signal&asynchronousIO.png)

[select/poll/epoll](E:\JavaNotes\Linux对于IO多路复用(Java NIO)的三种系统调用实现：.md)

#### 1.3在客户端和服务端通信时，对应的Java-BIO、Java-NIO和上述IO模型的联系和区别：

https://blog.csdn.net/dreamer23/article/details/80903978

对于BIO：客户端发起连接请求之后，BIO会直接开一个线程，用于处理这个连接，但是连接本身可能只有一小部分时间在真的发送数据，当没有数据的时候，服务端这个线程就进入阻塞，相当于1.2.1中BIO的read系统调用，直到真的有数据发过来并且从网卡复制到内核缓冲区再到用户缓冲区，服务端线程才退出阻塞处理数据；

```java
//EchoServerBIO:
//定义线程池，用来处理客户端连接
private static ExecutorService es = Executors.newCachedThreadPool();

public static void main(String[] args){
        ServerSocket serverSocket = null;
        Socket clientSocket = null;
        try{
            serverSocket = new ServerSocket(8000);
            while (true){
                clientSocket = serverSocket.accept();//等待连接，阻塞
                System.out.println(clientSocket.getRemoteSocketAddress() + "accept!");
                es.execute(new HandleMsg(clientSocket));//拿到一个连接，扔给线程池的线程处理
            }
        }catch (IOException e){
            e.printStackTrace();
        }
    }
    
 static class HandleMsg implements Runnable{
        Socket clientSocket;//客户端传来的Socket
        HandleMsg(Socket clientSocket){this.clientSocket = clientSocket;}

        @Override
        public void run() {
            BufferedReader br = null;
            PrintWriter pw = null;
            try {
                //br读取clientSocket发送来的数据，当客户端没有数据发送的时候，阻塞
                br = new BufferedReader(new InputStreamReader(clientSocket.getInputStream()));
                //拿到数据之后，处理.......
        	}
   		}
}
```

对于NIO：客户端发起连接请求之后，NIO不是直接开一个线程处理，而是先让这个连接连到选择器Selector上，同时，服务端这边调用select/poll/epoll系统调用，尝试获取活跃连接集合（也就是真的有数据发送的连接），获取目标连接之后，再对应地开启服务端线程，对读写事件进行处理

```java
//EchoServerNIO:
public static void main(String[] args) throws IOException {
        EchoServerNIO server = new EchoServerNIO();
        server.startServer();
}

private void startServer() throws IOException {
        selector = SelectorProvider.provider().openSelector();//根据具体的OS实现获取对应的选择器
        ServerSocketChannel serverSocketChannel = ServerSocketChannel.open();
        serverSocketChannel.configureBlocking(false);//这是为了后面accept时，不让主线程阻塞在accept上
        InetSocketAddress inetSocketAddress = new InetSocketAddress("localhost", 8000);
        serverSocketChannel.socket().bind(inetSocketAddress);

        //ssc在选择器上注册，表示自己对ACCEPT事件感兴趣
        SelectionKey acceptKey =
                serverSocketChannel.register(selector, SelectionKey.OP_ACCEPT);

        //开始轮询，获取感兴趣并且已经就绪的SelectionKey，从对应的channel获取数据并处理
        for(;;){
            int num = selector.select();//执行select/poll/epoll系统调用，获取活跃连接集合，只可能阻塞当前这一个线程
            /**
             * 获取已就绪的SelectionKey集合，一定是channel【感兴趣的事件们中至少有一个】就绪了
             * readyOps集合是interestOps的子集；
             * isWritable返回true，必须满足：该channel对写事件感兴趣，并且对写事件已经就绪，即write是属于readyOps的。
             * 但是对SocketChannel随时都可以调用write方法
             * （channel的读写都是通过sk获取到对方的SocketChannel，再read/write，没有对ServerSocketChannel读写的）
             * https://stackoverflow.com/questions/3745413/can-selectionkey-iswritable-be-true-without-op-write-in-interestops
             * */
            Set readyKeys = selector.selectedKeys();
            Iterator iterator = readyKeys.iterator();
            long end = 0;
            while (iterator.hasNext()){
                SelectionKey sk = (SelectionKey) iterator.next();
                iterator.remove();
                if (sk.isAcceptable()){//如果当前SelectionKey是准备好接收客户端连接
                    doAccept(sk);
                }
                else if (sk.isValid() && sk.isReadable()){//如果当前SelectionKey是准备好读数据
                    doRead(sk);//内部就可以用channel对数据读取处理
                }
                else if (sk.isValid() && sk.isWritable()){//如果当前SelectionKey是准备好写数据
                    doWrite(sk);          
                }
            }
        }
}
```

