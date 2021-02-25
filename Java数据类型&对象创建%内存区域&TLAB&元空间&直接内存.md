# 虚拟机栈和本地方法栈

<font color ="red">对于HotSpot虚拟机来说，两个是一个东西，因为hotspot并不区分二者</font>

并且对于二者可能抛出的OOM 和 StackOverFlowError：

StackOverFlowError：当线程请求栈深度 > 虚拟机允许的最大深度，抛出

OOM：如果**栈内存允许自动扩展**，在扩展栈容量的时候，无法申请到足够内存，OOM

<font color ="red">而HotSpot的虚拟机栈是不允许自动扩展的，所以HotSpot的虚拟机栈/本地方法栈，不管是因为栈深度太大，还是栈帧太大导致栈的内存不足，都只会抛出StackOverFlowError，不会出现OOM</font>

# Java数据类型：

Java虚拟机可以操作的数据类型分为两类：原始类型（基本类型） + 引用类型（reference类型）

**原始类型：**数值类型（整数类型、浮点类型）、Boolean类型（实际上字节码中没有用boolean类型的值，用int类型替代）、returnAddress类型（这种类型的值指向一条虚拟机指令的操作码）

**引用类型：**类类型、数组类型、接口类型

引用类型的值分别指向动态创建的类实例

# Java创建对象实例过程：

- 先检查这个符号引用有没有被类加载过；如果没有，先进行这个类的类加载clinit（有可能会递归触发父类的加载）
- 分配内存（编译之后，一个对象需要多大内存存储已经确定）（CAS失败重试/TLAB分配内存；指针碰撞/空闲列表）

- 除了对象头之外，零值初始化实例化字段

- 对象头设置（Mark Word + 类型指针）

- [init方法执行](E:\class的字节码执行引擎)（递归触发父类的init）

# 元空间：是对方法区的实现

#### 方法区存储类信息、常量、静态变量、即时编译器编译后的代码等信息

- ##### 类信息（类的元数据：类全限定名、方法名、访问权限、返回值等）在元空间

- ##### 常量池都移到堆存储

- ##### 静态变量在Class对象存储，Class对象在堆存储

- ##### JIT编译后的代码在方法区，实际上JIT编译后的代码存放于CodeCache区域，这个区域是堆外内存中，jconsole显示CodeCache和元空间Meta Space是同一个级别的。可能是在逻辑上把这块区域划入方法区 [链接](https://blog.csdn.net/wtopps/article/details/107067700)

[关于CodeCache:](https://juejin.cn/post/6844903796221427725)

**CodeCache本身属于堆外内存，当编译后的方法版本不能再使用（可能由于激进优化），这个编译后的数据就会被标记zombie，不能再使用这个编译版本，这块内存就可以被回收**

JDK8开始，元空间替代了永久代，方法区四分五裂，用多个区域进行存储

其中，字符串常量池、运行时常量池、class文件常量池全部移到堆中（String.intern()行为改变）

<font color = "red">运行时常量池作为方法区的一部分，物理上存于堆区，class文件常量池的东西会在类加载阶段中的加载阶段放入运行时常量池，但是运行时常量池和class常量池不同，具有动态性</font>

https://blog.csdn.net/Xu_JL1997/article/details/89433916?utm_medium=distribute.pc_relevant.none-task-blog-BlogCommendFromMachineLearnPai2-1.compare&depth_1-utm_source=distribute.pc_relevant.none-task-blog-BlogCommendFromMachineLearnPai2-1.compare

class对象同样在堆区，为了迎合移除永久代，静态变量也被移到了class对象，也在堆区

而类的元数据，即类的全限定名、方法名、访问权限、返回值等信息是在方法区/元空间的

类加载器的元数据，可能也是在元空间<font color = "red">[存疑?]</font>



元空间属于本地内存/直接内存，默认大小不受限，只受限于本地内存大小。本地内存和直接内存是同一个东西，大小受限于os机器内存，默认大小和堆的最大值一致



https://www.ibm.com/developerworks/library/j-nativememory-linux/#how

https://www.jianshu.com/p/4a469874e50b

JVM使用直接内存：元空间，NIO UnSafe，ThreadLocal<font color ="red">[未完待续]</font>



## 堆外内存的分配和回收

https://blog.csdn.net/nazeniwaresakini/article/details/104220245

堆外内存也称直接内存，Java可使用NIO/UnSafe操作堆外内存

当限制堆外内存的大小，并禁止显式调用System.gc()之后，一直使用NIO的ByteBuffer分配空间会出现OOM

堆外内存空间的分配和回收（以NIO的ByteBuffer为例）

使用ByteBuffer.allocateDirect()分配内存，会创建一个DirectByteBuffer对象

DirectByteBuffer有一个静态内部类Deallocator，实现了Runnable,run方法中，调用UnSafe的方法释放堆外内存；DirectByteBuffer构造方法中调用了reverseMemory方法和创建了一个Cleaner实例

在reverseMemory方法会尝试申请指定大小的空间，如果没有成功，会调用System.gc()尝试释放空间

Cleaner实例在创建的时候，会使用Deallocator做参数，相当于一个回调函数

Cleaner实质上是一个虚引用（在GC的时候，如果一个对象只有虚引用就会被立即加入到引用队列并回收空间。）且以双向队列的形式组织多个Cleaner对象。

在堆内的DirectByteBuffer对象被GC回收之后，对应的Cleaner被GC发现，需要回收，把Cleaner的引用加入到引用队列，这个时候，会判断这是一个Cleaner类型，调用这个节点的clean方法，回收堆外内存。（clean方法内部，会先把这个cleaner节点从双向链表删除，再调用这个节点的回调函数，也就是Deallocator的run方法，对应的堆外内存的空间就被回收了）



### [**TLAB**](https://www.jianshu.com/p/8be816cbb5ed)

Eden区单独划出一块区域，每个线程可以在里面单独拥有一块区域用于分配内存，每个线程使用两个指针start end划分出自己专属的区域，其他线程无法在这个专属区域分配内存，但是可以访问

`TLAB的本质其实是三个指针管理的区域：start，top 和 end，每个线程都会从Eden分配一块空间，例如说100KB，作为自己的TLAB，其中 start 和 end 是占位用的，标识出 eden 里被这个 TLAB 所管理的区域，卡住eden里的一块空间不让其它线程来这里分配。`

