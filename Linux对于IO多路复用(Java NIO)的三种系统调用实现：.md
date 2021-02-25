### 文件描述符和inode等相关基本预备知识：

1.扇区&块：

https://www.i3geek.com/archives/1275

http://www.ruanyifeng.com/blog/2011/12/inode.html

> ### 1、什么是扇区和（磁盘）块？
>
> 物理层面：一个磁盘按层次分为 **磁盘组合 -> 单个磁盘 -> 某一盘面 -> 某一磁道 -> 某一扇区**
>
> 扇区，顾名思义，每个磁盘有多条同心圆似的磁道，磁道被分割成多个部分。每部分的弧长加上到圆心的两个半径，恰好形成一个扇形，所以叫做扇区。**扇区是磁盘中最小的物理存储单位。通常情况下每个扇区的大小是512字节。（由于不断提高磁盘的大小，部分厂商设定每个扇区的大小是4096字节）**
>
> 逻辑层面： 磁盘块（虚拟出来的）。 **块是操作系统中最小的逻辑存储单位。操作系统与磁盘打交道的最小单位是磁盘块。**

> 文件储存在硬盘上，硬盘的最小存储单位叫做"扇区"（Sector）。每个扇区储存512字节（相当于0.5KB）。
>
> 操作系统读取硬盘的时候，不会一个个扇区地读取，这样效率太低，而是一次性连续读取多个扇区，即一次性读取一个"块"（block）。这种由多个扇区组成的"块"，是文件存取的最小单位。"块"的大小，最常见的是4KB，即连续八个 sector组成一个 block。

2.inode：

http://www.ruanyifeng.com/blog/2011/12/inode.html

- 每个文件都有对应inode节点，用于存储文件的元信息（如读写权限、拥有者等，**但是不存储文件名**）

- 硬盘格式化的时候，OS会把硬盘分成两个区域：数据区（存储文件数据）和inode区（inode table），inode区的大小和其中inode的数量由OS限定，因为每个文件都要有一个inode节点，所以有可能出现inode用光，但是硬盘实际还未存满，导致无法创建新文件的情况。

- inode号码：Linux内部使用inode号码而不是文件名来识别文件。

  表面上，用户通过文件名，打开文件。实际上，系统内部这个过程分成三步：首先，系统找到这个文件名对应的inode号码；其次，通过inode号码，获取inode信息；最后，根据inode信息，找到文件数据所在的block，读出数据。

- Unix/Linux系统中，目录（directory）也是一种文件。打开目录，实际上就是打开目录文件。

  目录文件的结构非常简单，就是一系列目录项（dirent）的列表。每个目录项，由两部分组成：所包含文件的文件名，以及该文件名对应的inode号码。

- 硬链接&软链接：

  一般来说，文件名和inode号码一一对应；但是实际上，Linux是允许多个文件名对应一个inode号码的

  比如文件A，对应inodeA，另外有一个文件B：

  硬链接：B的inodeB == inodeA，即文件A B都指向同一个inode，即使删除文件B，也不会影响A文件的访问

  > 反过来，删除一个文件名，就会使得inode节点中的"链接数"减1。当这个值减到0，表明没有文件名指向这个inode，系统就会回收这个inode号码，以及其所对应block区域。

  软链接：B有另外不同的inodeB != inodeA，但是B文件的内容写的是A文件的存储路径，读取B会自动导向到A文件读取，此时如果删除A，打开文件B的时候就会出错："No such file or directory"

3.文件描述符fd：

https://segmentfault.com/a/1190000009724931

### Linux对于IO多路复用(Java NIO)的三种系统调用实现：

**select / poll / epoll**

#### 1.select

https://zh.wikipedia.org/wiki/Select_(Unix)

https://www.shangmayuan.com/a/78c97829fce84e9280929fa2.html

```c
int select(int nfds, fd_set* readfds, fd_set* writefds, fd_set* errorfds, struct timeval* timeout);//select系统调用
```

nfds = 需要监听的多个文件描述符中的max + 1

后面三个xxxfds：表示需要检查是否可xxx的fd对应的fdset（这里文件描述符指的文件指客户端连接）

timeout：等待检查完成的最长时间

行为：

1.用户程序发起select系统调用，会把需要监听的fdset传递给内核，（fdset是一个数组，最大为1024.每个数组元素标志某一个文件描述符的状态，这里就是指客户端连接的状态。fdset的构建和初始化需要用户程序本身在发起select调用之前完成）这一步需要把fdset整个从用户态复制到内核态。

2.然后由内核遍历fdset，（应该是从0开始，顺序遍历nfds个元素）如果发现当前元素fdset[i]已经可读/可写，或者说该连接已就绪，对该元素fdset[i]置位。遍历完成之后，返回已就绪的文件描述符数量。这一步需要把修改后的fdset从内核拷贝到用户空间。

3.用户程序根据fdset，就拿到了当前已就绪的连接。

特点：select会阻塞用户程序；而且需要把整个fdset在用户空间和内核空间来回复制，消耗性能；且内核和应用程序都需要遍历fdset，同样消耗性能。

https://www.cnblogs.com/binarylei/p/11130079.html

https://www.cnblogs.com/NerdWill/p/4996476.html

为了减少数据拷贝带来的性能消耗，select对fdset集合大小做限制：默认限制为1024

`fd事件回调函数是pollwake，只是将本进程唤醒，本进程需要重新遍历全部的fd检查事件，然后保存事件，拷贝到用户空间，函数返回。`

#### 2.poll

poll的实现和select类似，区别在于poll使用的集合不是select的fdset，而是pollfd。且poll没有select的1024最大集合大小限制。

poll和select一样，都需要在用户态和内核态之间来回复制fd集合，且内核和用户程序都需要遍历fd集合，对性能消耗较大。

#### 3.epoll

（因为自己对操作系统底层不太熟悉，加上中文资料乱七八糟，所以只看了个大概）

https://blog.csdn.net/Eunice_fan1207/article/details/99674021

https://zh.wikipedia.org/wiki/Epoll

1.OS启动时，会为epoll注册一个新的文件系统eventpollfs，并在内核创建缓冲区，用于存放一个红黑树和一个双向链表

2.调用epoll_create时：内核会在eventpollfs中创建一个新file节点；并且在内核中创建一个红黑树和一个双向链表。（都在eventpoll结构中）

3.调用`int epoll_ctl(int epfd, intop, int fd, struct epoll_event *event);`时：

> 向 epfd 对应的内核`epoll` 实例添加、修改或删除对 fd 上事件 event 的监听。op 可以为 `EPOLL_CTL_ADD`, `EPOLL_CTL_MOD`, `EPOLL_CTL_DEL` 分别对应的是添加新的事件，修改文件描述符上监听的事件类型，从实例上删除一个事件。如果 event 的 events 属性设置了 `EPOLLET` flag，那么监听该事件的方式是边缘触发。

即有新的fd（socket连接）注册进来，使用eventpoll来管理：新的fd会被被拷贝到内核中，以epitem结点的形式加入到内核的红黑树中，并且和对应的物理设备（如网卡）建立回调关系，这个回调函数的作用：当该fd对应的事件就绪，（如可读）回调函数会把对应的epitem节点插入到位于内核中的双向链表。

4.调用`int epoll_wait(int epfd,struct epoll_event *events,int maxevents, int timeout);`

> 当 timeout 为 0 时，epoll_wait 永远会立即返回。而 timeout 为 -1 时，epoll_wait 会一直阻塞直到任一已注册的事件变为就绪。当 timeout 为一正整数时，epoll 会阻塞直到计时 timeout 毫秒终了或已注册的事件变为就绪。因为内核调度延迟，阻塞的时间可能会略微超过 timeout 毫秒。

直接去查看内核中的双向链表是否为空：如果不为空，说明有就绪节点，把内核中的该链表拷贝到用户空间，返回就绪fd和对应事件，以及就绪结点的数量。

**总结：epoll通过回调，避免了像select epoll那样需要多次遍历fd集合，也避免了在内核空间和用户空间之间来回复制整个fd集合，只需要在注册新fd的时候往内核中copy一次，在获取就绪链表的时候，把结果集从内核往用户空间copy一次，减少了不需要的遍历和拷贝。**

**epoll没有用到mmap！！！！**



### 