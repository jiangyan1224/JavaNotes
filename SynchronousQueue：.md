### SynchronousQueue：

```java
public SynchronousQueue(boolean fair) {
    transferer = fair ? new TransferQueue<E>() : new TransferStack<E>();
}
```

SynchronousQueue内部，其实核心在于TransferQueue和TransferStack，其余的waitQueue、qLock等，都是为了和1.5兼容，在1.8的实现中没有用上。这里以TransferStack为例介绍：

TransferStack中的SNode主要有next match waiter item 和mode这几个主要属性，分别为：指向下一个SNode（组成节点列表）、指向和自己操作匹配的SNode、提交该任务的线程（可以对它进行阻塞唤醒等操作）、要生产的数据/null（取决于当前线程是生产者还是消费者）、REQUEST/DATA/FULFILLING（取决于当前线程是生产者还是消费者）

```java
E transfer(E e, boolean timed, long nanos) {
    SNode s = null; // constructed/reused as needed
    int mode = (e == null) ? REQUEST : DATA;

    for (;;) {
        SNode h = head;
        if (h == null || h.mode == mode) {  // empty or same-mode
            if (timed && nanos <= 0) {      // can't wait
                if (h != null && h.isCancelled())
                    casHead(h, h.next);     // pop cancelled node
                else
                    return null;
            } else if (casHead(h, s = snode(s, e, h, mode))) {
                SNode m = awaitFulfill(s, timed, nanos);//给当前节点的waiter赋值为当前线程（提交该任务的线程），选择是否进行自旋，然后对它park/超时park,只有当waiter被unpark，并且有了match节点，才会返回对应的match节点（如果线程被interrupt，会把match节点置为自己）
                if (m == s) {               // wait was cancelled
                    clean(s);
                    return null;
                }
                if ((h = head) != null && h.next == s)
                    casHead(h, s.next);     // help s's fulfiller
                return (E) ((mode == REQUEST) ? m.item : s.item);
            }
        } else if (!isFulfilling(h.mode)) { // try to fulfill栈顶和自己操作匹配
            if (h.isCancelled())            // already cancelled
                casHead(h, h.next);         // pop and retry
            else if (casHead(h, s=snode(s, e, h, FULFILLING|mode))) {
                for (;;) { // loop until matched or waiters disappear
                    SNode m = s.next;       // m is s's match
                    if (m == null) {        // all waiters are gone
                        casHead(s, null);   // pop fulfill node
                        s = null;           // use new node next time
                        break;              // restart main loop
                    }
                    SNode mn = m.next;
                    if (m.tryMatch(s)) {//tryMatch会设置m节点的match节点，unparkm节点的waiter并把waiter=null，
                        casHead(s, mn);     // pop both s and m
                        return (E) ((mode == REQUEST) ? m.item : s.item);//返回m s二者中消费者节点的item
                    } else                  // lost match
                        s.casNext(m, mn);   // tryMatch失败，说明ms匹配失败，有可能是m节点的线程被interrupt，把m的match置为自己m，所以当前线程就帮助删除m节点，并继续往下找匹配节点。
                }
            }
        } else {                            // help a fulfiller栈顶正在fulfill，帮助把栈顶的下一个节点（和栈顶匹配的节点）match更新为栈顶
            SNode m = h.next;               // m is h's match
            if (m == null)                  // waiter is gone
                casHead(h, null);           // pop fulfilling node
            else {
                SNode mn = m.next;
                if (m.tryMatch(h))          // help match
                    casHead(h, mn);         // pop both h and m
                else                        // lost match
                    h.casNext(m, mn);       // help unlink
            }
        }
    }
}
```

