#### ConcurrentHashMap粗略解释：

initTable：以CAS线程安全的方式初始化table：`Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n];`初始化容量为n = sizeCtl > 0 ? sizeCtl : DEFAULT_CAPACITY(16)。初始化table之后sizeCtl=n*0.75

initTable应该能保证n为2的幂次方

##### 对于putVal方法的粗略理解：

```java
final V putVal(K key, V value, boolean onlyIfAbsent) {
    if (key == null || value == null) throw new NullPointerException();
    int hash = spread(key.hashCode());
    int binCount = 0;
    for (Node<K,V>[] tab = table;;) {
        Node<K,V> f; int n, i, fh;
        if (tab == null || (n = tab.length) == 0)
            tab = initTable();//如果当前table还没有初始化，需要先初始化。初始化之后会因为外层死循环再次进入，尝试放入值
        else if ((f = tabAt(tab, i = (n - 1) & hash)) == null) {//如果当前table对应位置为空，说明这里还没有头节点，直接插入
            if (casTabAt(tab, i, null,
                         new Node<K,V>(hash, key, value, null)))
                break;                   // no lock when adding to empty bin
        }
        else if ((fh = f.hash) == MOVED)//如果当前头节点状态为MOVED，只要当前头节点的移动状态还没有结束，所有执行putVal方法的线程都要进入helptransfer方法，帮忙转移节点
            tab = helpTransfer(tab, f);
        else {//开始真正的put新节点
            V oldVal = null;
            synchronized (f) {//只对头节点加锁
                if (tabAt(tab, i) == f) {//这里的判断和DCL类似，如果有线程在进入putVal之后helpTransfer之前阻塞，等其他线程完成了节点转移之后，这里的条件就不再成立，需要回到最外层死循环更新tab引用
                    if (fh >= 0) {
                        binCount = 1;
                        /**
                        这里的for循环只做了两件事：
                        如果当前放入的key和当前table中已有的头节点相同，说明表头结点和要放入的节						   					点重复了，根据onlyIfAbsent判断是否要覆盖原节点；
                        如果当前表头结点没有后继，直接新建一个Node，加入到表头结点的后面；
                        binCount是用来记录当前表头结点以及其后续一共有多少个节点，用来判断后面要不要把链表转为红黑树
                        */
                        for (Node<K,V> e = f;; ++binCount) {
                            K ek;
                            if (e.hash == hash &&
                                ((ek = e.key) == key ||
                                 (ek != null && key.equals(ek)))) {
                                oldVal = e.val;
                                if (!onlyIfAbsent)
                                    e.val = value;
                                break;
                            }
                            Node<K,V> pred = e;
                            if ((e = e.next) == null) {
                                pred.next = new Node<K,V>(hash, key,
                                                          value, null);
                                break;
                            }
                        }
                    }
                    //如果表头结点本来就是红黑树节点，按照红黑树的方式加入新结点
                    else if (f instanceof TreeBin) {
                        Node<K,V> p;
                        binCount = 2;
                        if ((p = ((TreeBin<K,V>)f).putTreeVal(hash, key,
                                                       value)) != null) {
                            oldVal = p.val;
                            if (!onlyIfAbsent)
                                p.val = value;
                        }
                    }
                }
            }
            if (binCount != 0) {
                if (binCount >= TREEIFY_THRESHOLD)
                    treeifyBin(tab, i);
                //如果之前的putVal最后是以覆盖原节点结束的，oldVal!=null成立，直接return，不会执行后面的addCount方法
                if (oldVal != null)
                    return oldVal;
                break;
            }
        }
    }
    //addCount方法：用于增加map的元素个数，第一个参数是用来记录增加多少个元素；第二个参数只是用来判断是否需要扩容
    addCount(1L, binCount);
    return null;
}
```

addCount方法的调用可能会触发map的扩容，因为涉及到CountCells数组的使用，所以在对addCount解释之前，先解释一下用来获取整体元素个数与的size方法：

```java
public int size() {
    long n = sumCount();
    return ((n < 0L) ? 0 :
            (n > (long)Integer.MAX_VALUE) ? Integer.MAX_VALUE :
            (int)n);
}

final long sumCount() {
        CounterCell[] as = counterCells; CounterCell a;
        long sum = baseCount;//因为有可能一开始对map的操作并没有涉及到线程竞争问题，所以一开始元素的计数，是用baseCount这个值记录的，后面出现竞争之后，就不再对baseCount改动，而是使用CountCell数组记录每个线程加入的元素个数
        if (as != null) {
            for (int i = 0; i < as.length; ++i) {
                if ((a = as[i]) != null)
                    sum += a.value;
            }
        }
        return sum;//返回的结果=baseCount + 遍历CountCell数组所有元素值之和
    }
```

CountCell数组，我自己理解，每个线程都会使用ThreadLocalRandom生成一个自己独有的随机数种子，根据这个种子来确定自己这个线程存值的时候，是在哪一个索引位置存入一个自己对应的CountCell实例（实例内部存值）：（假设CountCell数组名为rs）`rs[(rs.length-1) & ThreadLocalRandom.getProbe() ] = new CounterCell(x) ;`

这样做，也避免了在有多个线程需要改动map元素个数的时候，所有线程都堵在baseCount上同步。

对于CountCell类：

```java
@sun.misc.Contended static final class CounterCell {
    volatile long value;
    CounterCell(long x) { value = x; }
}
```

CountCell类上的注解Contended，是为了避免volatile变量带来的[伪共享问题：](https://mp.weixin.qq.com/s?__biz=MzI1ODI3NzQ5OA==&mid=2247483851&idx=1&sn=7e2099d2e1843ed7fd125b79827b6296&chksm=ea0be8dedd7c61c8c049d022032dd01ee56319ba61c8f92239107591ab060cfd069141561790&scene=21#wechat_redirect)

volatile：在多个线程自己的工作缓存内都缓存了一份主存中的同一份共享数据，某个线程对修改这份数据，更改了自己的工作缓存内部的对应值之后，volatile可以让新值立即被写入主存，其他线程工作缓存内部的旧值被无效化，所以其他线程在使用这个共享变量的时候，就被强制去主存更新新值，从而达到了volatile变量线程可见性的要求。

但是，在线程工作缓存内部的操作，是以缓存行为单位的，（不同型号电脑，缓存行的大小可能会不同）一个缓存行内部，可能并不只存了volatile一个变量，可能还存储了其他普通变量。这就导致volatile致使其他线程中对应的缓存行整个无效，**其他线程必须去主存更新所有涉及到的变量，**带来了性能问题。这叫**伪共享**问题。（Java8之前，是用自己添加无用变量增加volatile用的内存大小，强行扩张到一整个缓存行，但是这样不仅可能会被JVM优化掉无效变量，而且对于不同型号的电脑，无效变量的大小数量也都不确定；Java8之后就使用了Contended注解和` -XX:-RestrictContended` ，让JVM给其添加填充物，达到一整个缓存行。 ）

##### 回到addCount方法：

```java
private final void addCount(long x, int check) {
    CounterCell[] as; long b, s;
    //前面这个if块，就是用来增加元素数量的。先尝试增加baseCount，如果失败，进入fullAddCount，对CountCell数组修改，增加元素数目。
    if ((as = counterCells) != null ||
        !U.compareAndSwapLong(this, BASECOUNT, b = baseCount, s = b + x)) {
        CounterCell a; long v; int m;
        boolean uncontended = true;
        if (as == null || (m = as.length - 1) < 0 ||
            (a = as[ThreadLocalRandom.getProbe() & m]) == null ||
            !(uncontended =
              U.compareAndSwapLong(a, CELLVALUE, v = a.value, v + x))) {
            fullAddCount(x, uncontended);
            return;
        }
        if (check <= 1)
            return;
        s = sumCount();
    }
    //开始检查是否需要扩容resize
    if (check >= 0) {
        Node<K,V>[] tab, nt; int n, sc;
        while (s >= (long)(sc = sizeCtl) && (tab = table) != null &&
               (n = tab.length) < MAXIMUM_CAPACITY) {
            int rs = resizeStamp(n);
            if (sc < 0) {//sizeCtl < 0说明有其他线程正在进行扩容
                if ((sc >>> RESIZE_STAMP_SHIFT) != rs || sc == rs + 1 ||
                    sc == rs + MAX_RESIZERS || (nt = nextTable) == null ||
                    transferIndex <= 0)
                    break;
                if (U.compareAndSwapInt(this, SIZECTL, sc, sc + 1))
                    transfer(tab, nt);
            }
            //sizeCtl >= 0说明当前线程是第一个参与到扩容的线程
            else if (U.compareAndSwapInt(this, SIZECTL, sc,
                                         (rs << RESIZE_STAMP_SHIFT) + 2))
                transfer(tab, null);
            s = sumCount();
        }
    }
}
```

##### 扩容相关的两个方法：helpTransfer和transfer：

[helpTransfer](https://www.jianshu.com/p/39b747c99d32)只是当前线程参与到扩容中

```java
final Node<K,V>[] helpTransfer(Node<K,V>[] tab, Node<K,V> f) {
    Node<K,V>[] nextTab; int sc;
    if (tab != null && (f instanceof ForwardingNode) &&
        (nextTab = ((ForwardingNode<K,V>)f).nextTable) != null) {
        int rs = resizeStamp(tab.length);
        while (nextTab == nextTable && table == tab &&
               (sc = sizeCtl) < 0) {
            //判断当前map的扩容是否已经结束
            if ((sc >>> RESIZE_STAMP_SHIFT) != rs || sc == rs + 1 ||
                sc == rs + MAX_RESIZERS || transferIndex <= 0)
                break;
            //扩容还没有结束，加入transfer，sc=sizeCtl的低16位代表当前参与扩容的线程数目，高16位是一个标志（rs = resizeStamp返回值）
            if (U.compareAndSwapInt(this, SIZECTL, sc, sc + 1)) {
                transfer(tab, nextTab);
                break;
            }
        }
        return nextTab;
    }
    return table;
}
```

[transfer：](https://www.jianshu.com/p/2829fe36a8dd)

```java
private final void transfer(Node<K,V>[] tab, Node<K,V>[] nextTab) {
    int n = tab.length, stride;
    //获取当前CPU核心的个数，如果核心数>1，length/8/CPU核心数得到每个线程处理的桶个数；如果核心数<=1，则一个线程处理所有桶。
    //要求这个结果必须>=16，即每个线程都必须至少处理16个桶
    if ((stride = (NCPU > 1) ? (n >>> 3) / NCPU : n) < MIN_TRANSFER_STRIDE)
        stride = MIN_TRANSFER_STRIDE; // subdivide range
    //初始化nextTable，大小为旧tab的2倍
    if (nextTab == null) {            // initiating
        try {
            @SuppressWarnings("unchecked")
            Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n << 1];
            nextTab = nt;
        } catch (Throwable ex) {      // try to cope with OOME
            sizeCtl = Integer.MAX_VALUE;
            return;
        }
        nextTable = nextTab;
        transferIndex = n;
    }
    int nextn = nextTab.length;
    //ForwardingNode，后面要用这种节点替代旧tab的桶节点，ForwardingNode状态为MOVED，方便让其他访问该桶的线程进入helpTransfer帮助扩容
    //ForwardingNode构造函数的参数指的是当前节点指向的下一个新表
    ForwardingNode<K,V> fwd = new ForwardingNode<K,V>(nextTab);
    boolean advance = true;
    boolean finishing = false; // to ensure sweep before committing nextTab
    for (int i = 0, bound = 0;;) {
        Node<K,V> f; int fh;
        while (advance) {//这个while循环用于设置当前线程扩容的桶区间，即这个线程要给table哪个桶区间的桶们rehash
            //区间为[bound, nextIndex)，nextIndex第一个线程进来的时候是n，即旧tab长度。i从右区间开始--
            int nextIndex, nextBound;
            //如果--i >= bound不成立，说明当前线程已经完成了自己被分配到的区间内桶的rehash
            //如果再加上finishing==false,进入下面的elseif判断nextIndex,以及后面可能有的U.compareAndSwapInt获取下一次任务区间
            if (--i >= bound || finishing)
                advance = false;
            else if ((nextIndex = transferIndex) <= 0) {
                i = -1;
                advance = false;
            }
            //每个线程在进入这个else if之后，就会通过U.compareAndSwapInt方法修改map的nextIndex值为自己任务的左区间
            //这样下一个进入设置区间的线程就从这个左区间再往左，划分自己的任务区间。
            else if (U.compareAndSwapInt
                     (this, TRANSFERINDEX, nextIndex,
                      nextBound = (nextIndex > stride ?
                                   nextIndex - stride : 0))) {
                bound = nextBound;
                i = nextIndex - 1;
                advance = false;
            }
        }//设置扩容区间结束
        
        if (i < 0 || i >= n || i + n >= nextn) {
            int sc;
            if (finishing) {
                nextTable = null;
                table = nextTab;
                sizeCtl = (n << 1) - (n >>> 1);
                return;
            }
            if (U.compareAndSwapInt(this, SIZECTL, sc = sizeCtl, sc - 1)) {
                if ((sc - 2) != resizeStamp(n) << RESIZE_STAMP_SHIFT)
                    return;
                finishing = advance = true;
                i = n; // recheck before commit
            }
        }
        else if ((f = tabAt(tab, i)) == null)//如果桶节点为空（有可能是整体元素数量达到了sizeCtl阈值，但是某些桶可能还是空的）
            advance = casTabAt(tab, i, null, fwd);//把桶节点CAS替换为ForwardingNode
        else if ((fh = f.hash) == MOVED)//说明这个桶已经有线程处理完毕了
            advance = true; // already processed
        else {//进入到这里，说明【当前】这个桶非空，但也还不是ForwardingNode，需要当前线程去替换桶节点为Forwarding
            synchronized (f) {//对这个桶节点加锁
                if (tabAt(tab, i) == f) {//一般来说，在syn块内的线程安全判断if，是为了防止在进入syn块之前，其他线程的操作导致这个条件不再成立。
                    Node<K,V> ln, hn;
                    if (fh >= 0) {//如果当前桶节点是链表节点而不是红黑树节点
                        int runBit = fh & n;//因为n一定是2的幂次方，类似于00000100，所以&的结果一定只有一位是1
                        Node<K,V> lastRun = f;
                        //这个for循环在遍历桶节点后面的节点，通过hash&n判断当前节点p的类型
                        //在for循环结束跳出之后，lastRun以及其后面的节点都和lastRun是同一类型：全都是高位/低位。
                        //lastRun是为了减少在后面新建链表的时候的遍历次数
                        for (Node<K,V> p = f.next; p != null; p = p.next) {
                            int b = p.hash & n;
                            if (b != runBit) {
                                runBit = b;
                                lastRun = p;
                            }
                        }
                        //ln代表最后一个低位节点；hn代表最后一个高位节点，且ln hn后面的所有结点都和自己同一类型
                        if (runBit == 0) {
                            ln = lastRun;
                            hn = null;
                        }
                        else {
                            hn = lastRun;
                            ln = null;
                        }
                        //从桶节点开始遍历，对每个遍历到的节点进行头插法新建
                        for (Node<K,V> p = f; p != lastRun; p = p.next) {
                            int ph = p.hash; K pk = p.key; V pv = p.val;
                            if ((ph & n) == 0)
                                ln = new Node<K,V>(ph, pk, pv, ln);
                            else
                                hn = new Node<K,V>(ph, pk, pv, hn);
                        }
                        //最后把ln hn接的两条链表分别接入到新表的桶中：
                        /**
                        这里对旧节点的分散类似于0/1随机分散：hash&旧n的结果非0即1：
                        假设n=4,某一个hash&(n-1) = 2(0010)  --> hash = xx10
                        新hash=8，还是这个hash,hash&(n-1) = hash & 0111 = 0x10
                        即，resize前后，同一个hash，它的位置变化只有一位，新位置 = 老位置 + 0 or 旧长度，结果变化取决于hash对应位是0还是1，也就是前面说的高位/低位。
                        hash值的某一位为0为1可以看作是一个0/1随机数，以这种方式计算节点rehash的新位置大大减小了用hash&新长度-1 的计算量
                        */
                        setTabAt(nextTab, i, ln);
                        setTabAt(nextTab, i + n, hn);
                        setTabAt(tab, i, fwd);
                        advance = true;
                    }
                    else if (f instanceof TreeBin) {
                        /**
                        如果桶节点不是链表而是树节点
                        和链表节点类似的处理，只是在组织成新的两个Node list之后，判断list的长度，根据长度把list转成链表，						 还是建立一颗新的树。
                        */
                    }
                }
            }
        }
    }
}
```

总的来说，ConcurrentHashMap的put方法，用到了synchronized，但也只是对头节点加锁，其他部分都是用的CAS循环保证线程安全。

##### 加一个get方法的理解：get方法无锁读

```java
public V get(Object key) {
    Node<K,V>[] tab; Node<K,V> e, p; int n, eh; K ek;
    int h = spread(key.hashCode());//先根据key的哈希值映射到h值
    if ((tab = table) != null && (n = tab.length) > 0 &&
        (e = tabAt(tab, (n - 1) & h)) != null) {//根据key哈希映射值得到存储位置
        if ((eh = e.hash) == h) {//如果对应头节点的hash映射值和key的对应值不相等:有可能是哈希冲突了，要去后面的链表/红黑树遍历查找；或者是因为头节点的hash映射值就是个负数，整个map正在扩容，而当前这个桶已经rehash完毕，要找就去nextTable找
            if ((ek = e.key) == key || (ek != null && key.equals(ek)))
                return e.val;
        }
        else if (eh < 0)
            return (p = e.find(h, key)) != null ? p.val : null;
        //遍历比较
        while ((e = e.next) != null) {
            //要求节点的key的哈希映射值相同，但是由于map用拉链法解决哈希冲突，所以还需要比较key本身：
            //要么是同一个key实例，要么是完全相同的两个key实例（equals方法）。
            if (e.hash == h &&
                ((ek = e.key) == key || (ek != null && key.equals(ek))))
                return e.val;
        }
    }
    return null;
}
```



#### ConcurrentLinkedQueue更加粗略解释：（跟着代码走了一遍，但是感觉没看透。。）

[链接](https://www.jianshu.com/p/231caf90f30b)

非阻塞CAS保证线程安全

poll执行的时候，会让旧head的next指向自己（自引用）。在连续添加三个元素并且连续poll三个元素之后，再添加第四个元素的时候，会出现`p==q`的情况，让p找到新的head，继续添加元素

offer方法：

```java
public boolean offer(E e) {
    checkNotNull(e);
    final Node<E> newNode = new Node<E>(e);

    for (Node<E> t = tail, p = t;;) {
        Node<E> q = p.next;
        if (q == null) {
            // p is last node
            if (p.casNext(null, newNode)) {
                if (p != t) //这里更新tail尾节点，实际上是每隔一次调用offer就会更新（即假设连续添加abcd四个节点，只会在添加b d时才更新tail）
                    casTail(t, newNode);  // Failure is OK.
                return true;
            }
            // Lost CAS race to another thread; re-read next
        }
        else if (p == q)//在poll了所有元素，再添加新元素时会进入这种情况。此时pq均指向哨兵节点（即自己指自己，这种节点已经是要被删除的节点了），这个时候要找到新的head节点再继续offer，或者如果发现tail更新了（有可能是其他线程已经做了更新tail工作，让tail正常指向了最后一个/倒数第二个live节点），那么这个时候就可以赌一把，给p赋值为新tail，如果赌成功了，就不用从新head一直往后遍历才找到tail了，方便插入新节点。
            // We have fallen off list.  If tail is unchanged, it
            // will also be off-list, in which case we need to
            // jump to head, from which all live nodes are always
            // reachable.  Else the new tail is a better bet.
            p = (t != (t = tail)) ? t : head;//对于这一行(t != (t = tail))的判断：先获取当前t的值，再赋值t为当前tail的值，如果在单线程下，t!=t肯定不成立，但是一旦tail更新了，导致t被赋了新tail值，旧t!=t自然成立
        else
            // Check for tail updates after two hops.
            p = (p != t && t != (t = tail)) ? t : q;
    }
}
```

poll方法：假设当前队列，先offer一个节点A，再对它poll：

poll时，tail是指向head头结点的

```java
public E poll() {
    restartFromHead:
    for (;;) {
        for (Node<E> h = head, p = h, q;;) {
            E item = p.item;

            if (item != null && p.casItem(item, null)) {
                // Successful CAS is the linearization point
                // for item to be removed from this queue.
                if (p != h) // hop two nodes at a time
                    updateHead(h, ((q = p.next) != null) ? q : p);
                return item;
            }
            else if ((q = p.next) == null) {
                updateHead(h, p);
                return null;
            }
            else if (p == q)
                continue restartFromHead;
            else
                p = q;
        }
    }
}
```

变化图：

![](C:\Users\123\Pictures\JVM\ConcurrentLinkedQueue.jpg)

后面就要执行`updateHead(h, ((q = p.next) != null) ? q : p);`

```java
final void updateHead(Node<E> h, Node<E> p) {
    if (h != p && casHead(h, p))
        h.lazySetNext(h);
}
```

会让head指向后面的节点，而h指向的旧head，则自己指自己，成为哨兵节点。