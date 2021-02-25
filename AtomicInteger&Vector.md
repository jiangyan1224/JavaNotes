Integer内部的值value，定义为：`private final int value;`

AtomicInteger内部的值value，定义为：`private volatile int value;`；同时使用内部含有的Unsafe实例来对value修改值，比如incrementAndGet或者getAndIncrement等等。这些方法本身调用的就是Unsafe类的方法，本身不需要volatile修饰；但是AtomicInteger内部还有一个方法get，直接返回value值，这个时候就需要volatile，来保证读到的值一定是最新的值

Vector非线程安全：虽然其内部的所有方法都是用synchronized修饰的，但是几个方法组合也并不一定线程安全，比如：

（可以直接把removeLast内部的所有操作都包含在synchronized(vector)同步块内保证线程安全）

[链接](https://juejin.cn/post/6844903717674680328)

```java
public class VectorTest {
    private static Vector<Integer> vector = new Vector<>();
    public static void removeLast(){//这个方法内含两个synchronized方法的组合，但是这个方法本身并不线程安全
        int lastIndex = vector.size() -1;
        try {
            Thread.sleep(1000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        vector.remove(lastIndex);
    }
    public static void main(String[] args){
        for (int i = 0; i < 100; i++) {
            vector.add(i);
        }
        new Thread(new Runnable() {
            @Override
            public void run() {
                for (int i = 0; i < 20; i++) {
                    removeLast();
                }
            }
        }).start();
        new Thread(new Runnable() {
            @Override
            public void run() {
                for (int i = 0; i < 20; i++) {
                    removeLast();
                }
            }
        }).start();
    }
}
当线程1计算完lastIndex结束被挂起，线程2成功执行完一整个removeLast方法，会让Vector.elementCount--
此时，线程1再继续运行就会报错，因为此时99已经超过了size(),抛出ArrayIndexOutOfBoundsException
Exception in thread "Thread-1" java.lang.ArrayIndexOutOfBoundsException: Array index out of range: 99
	at java.util.Vector.remove(Vector.java:831)
	at threadSafe.VectorTest.removeLast(VectorTest.java:17)
	at threadSafe.VectorTest$2.run(VectorTest.java:35)
	at java.lang.Thread.run(Thread.java:745)
```

