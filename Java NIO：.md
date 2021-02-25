### Java NIO：

**（强转类型）某方法a.某方法b   这是把a b 方法执行完得到的结果强转类型**

在channel和selector使用register关联生成SelectionKey的时候，必须告诉selector自己对什么事件感兴趣

SelectionKey是包含它所关联的channel、selector、和channel感兴趣的事件 这些信息的

在channel调用register时，会生成SelectionKey并存入selector的**Registered key set**

在调用selector.select()时，selector会遍历**Registered key set**，对每一个SelectionKey进行判断，如果当前SelectionKey的channel中，channel感兴趣的事件已经就绪，selector就把这个SelectionKey写入到**Selected key set**

https://www.cnblogs.com/snailclimb/p/9086334.html

> 通道触发了一个事件意思是该事件已经就绪。比如某个Channel成功连接到另一个服务器称为“ **连接就绪** ”。一个Server Socket Channel准备好接收新进入的连接称为“ **接收就绪** ”。一个有数据可读的通道可以说是“ **读就绪** ”。等待写数据的通道可以说是“ **写就绪** ”。

selector.selectedKeys()方法就能返回这个**Selected key set**，而这个集合中的元素，selector本身无法remove，必须要程序员自己remove。否则，在某次循环中处理了的SelectionKey，不手动remove，下一次再次selector.select()和selector.selectedKeys()，就会重复处理之前已经处理过了的SelectionKey

### 对于selector.select()方法：（本来想看select方法的源码，但是jad不能完全反编译，加上外面那群小屁孩太吵，看不下去所以作罢。后面看看底层的系统调用）

select方法的注释：

> Selects a set of keys whose corresponding channels are ready for I/O operations.
> **This method performs a blocking selection operation.** It returns only after at least one channel is selected, this selector's wakeup method is invoked, or the current thread is interrupted, whichever comes first.
>
> Returns:
> The number of keys, possibly zero, whose ready-operation sets were updated
> Throws:
> java.io.IOException – If an I/O error occurs
> java.nio.channels.ClosedSelectorException – If this selector is closed

https://www.cnblogs.com/buptleida/p/12713514.html

https://stackoverflow.com/questions/9939989/java-nio-selector-select-returns-0-although-channels-are-ready

行为：会造成线程阻塞。（阻塞不一定是wait等造成的阻塞，也可以是IO阻塞，IO阻塞不影响线程本身状态）让所有已经就绪的SelectionKey进入SelectedKeySet；如果当前SelectedKeySet不为空会直接返回，如果当前没有就绪的key会一直阻塞，直到有key就绪/interrupt/wakeup()

返回：从调用select()开始，readyOps发生变化的key数量



https://stackoverflow.com/questions/7132057/why-the-key-should-be-removed-in-selector-selectedkeys-iterator-in-java-ni

因为Selector本身不会删除SelectedKeySet的项，所以在用迭代器获取一个key之后，需要从SelectedKeySet删除这个key，否则下一次从select返回，再次遍历SelectedKeySet时，会再次遍历到这个已经被处理，理应被删除的key，导致这个key被重复处理。



https://stackoverflow.com/questions/3745413/can-selectionkey-iswritable-be-true-without-op-write-in-interestops

```java
/**
 * 获取已就绪的SelectionKey集合，一定是channel【感兴趣的事件们中至少有一个】就绪了
 * readyOps集合是interestOps的子集；
 * isWritable返回true，必须满足：该channel对写事件感兴趣，并且对写事件已经就绪，即write是属于readyOps的。
 * 但是对SocketChannel随时都可以调用write方法
 * （channel的读写都是通过sk获取到对方的SocketChannel，再read/write，没有对ServerSocketChannel读写的）
 * */
Set readyKeys = selector.selectedKeys();
```



**当一个客户端连接到来时，OP_ACCEPT事件就绪。**



《实战Java高并发程序设计》中，p242，对于NIO和BIO两个程序得到的spend时间的差距猜测：

BIO中，一旦客户端开始发数据，BIOserver就建一个线程连接，等着拿数据

而NIO中，要等到客户端把数据准备好，再开始读进入doRead方法，没准备好之前，NIOserver阻塞在select上

实验：BIOclient端，打印从开始print到最后一个print结束的开始结束时间

NIOserver：打印进入doRead方法到退出doWrite的开始结束时间

BIOserver：打印从获取OutputStream到读完结果的开始结束时间

BIOclient  --  NIOserver：

```java
NIOserver:
spend:1612523267878-1612523267876=2ms
BIOclient:
bio-client-print-start1612523265875
bio-client-print-end1612523267876
```

BIOclient  --  BIOserver：

```java
BIOserver:
spend1612523483617-1612523481630=1987ms
BIOclient:
bio-client-print-start1612523481616
bio-client-print-end1612523483617
```

猜测基本成立，但是NIOserver怎么知道客户端数据准备好，什么时候可以进入doRead

https://blog.csdn.net/zzh920625/article/details/107727770

> If the selector detects that the corresponding channel is ready for reading, has reached end-of-stream, has been remotely shut down for further reading, or has an error pending, then it will add 'OP_READ' to the key's ready-operation set and add the key to its 'selected-key' set.



[几种常见的IO模型：](E:\JavaNotes\Java IO底层原理：.md)

[select/poll/epoll](E:\JavaNotes\Linux对于IO多路复用(Java NIO)的三种系统调用实现：.md)