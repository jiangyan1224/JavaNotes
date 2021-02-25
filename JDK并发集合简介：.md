#### JDK并发集合简介：

ConcurrentHashMap：线程安全的HashMap

ConcurrentLinkedQueue：线程安全的LinkedList

[这两个都有看过源码](E:\ConcurrentHashMap(volatile伪共享问题)&ConcurrentLinkedQueue：.md)

CopyOnWriteArrayList：线程安全的ArrayList，后面介绍

BlockingQueue：表示阻塞队列的接口

ConcurrentSkipListMap：跳表的实现，这是一个Map，使用跳表的数据结构进行快速查找

Collections.synchronizedXXX方法：其实都是在List/Map/Set上加了一个Object mutex，在对这些集合操作的时候，外面包了一个synchronized(mutex){}，从而实现线程安全

##### CopyOnWriteArrayList：适用于多读少写，读的时候完全没有加锁，而写操作和写操作之间需要加锁，写和读之间不用加锁。

写的时候，是直接复制当前数组一个副本出来，在副本上修改，修改完了再更新数组引用指向新的数组；因为数组引用使用volatile修饰，所以读的时候，可以立即看到新引用。

```java
private transient volatile Object[] array;
```

```java
public E get(int index) {
    return get(getArray(), index);
}
final Object[] getArray() {
    return array;
}
private E get(Object[] a, int index) {
   return (E) a[index];
}
```

```java
public boolean add(E e) {
    final ReentrantLock lock = this.lock;
    lock.lock();
    try {
        Object[] elements = getArray();
        int len = elements.length;
        Object[] newElements = Arrays.copyOf(elements, len + 1);
        newElements[len] = e;
        setArray(newElements);
        return true;
    } finally {
        lock.unlock();
    }
}
```

##### BlockingQueue：[ArrayBlockingQueue LinkedBlockingQueue](E:\阻塞队列&ForkJoin问题！！！：.md)

##### ConcurrentSkipListMap：线程安全的Map，使用跳表实现Map，和用哈希实现Map的区别在于：用跳表实现可以保存元素顺序；而哈希不会。

跳表类似于平衡树，查询性能和平衡树类似，nlogn；但是对平衡树插入/删除节点时，由于可能涉及到整个树结构的调整，就需要全局锁；而对于跳表，局部锁就够了