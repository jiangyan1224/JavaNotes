##### 编译是生成class文件的字节码

###### 类初始化的触发时机：（初始化会调用clinit方法，包含静态代码块）

1.new对象实例、读取/设置静态字段(被final修饰，值已经放入class文件常量池的除外)、调用静态方法的时候，如果该类还没有初始化，进行初始化

<font color = "orang">（注1：这个需要初始化的类一定是直接定义了这个字段/方法的类，而从父类继承对应字段/方法的子类，不会被初始化。即假设Zi继承Fu类，如果new/读取/调用了一个Fu类的实例/静态字段/方法，只会触发Fu类的初始化；对于静态成员内部类同理，如果只调用了外部类的静态成员只会导致外部类的初始化，不会触发静态成员内部类的初始化）</font>

<font color = "orang">（注2：如果一个类是数组类，new一个数组的时候，触发的是对应数组类型的初始化而不是数组元素类型的初始化。即对于Integer[] arr = new Integer[10]，触发的是 [java.util.Integer 这个数组类的初始化）</font>

2.一个定义了default方法的接口，对应的实现类如果需要初始化，该接口需要在其之前初始化

3.初始化某个类时，如果对应的父类还没有初始化，先初始化父类

<font color ="red">即，类初始化的时候，如果它实现了带有default方法的接口，那么这些接口要先初始化，否则实现的接口是不需要初始化的；但是所有父类都要先初始化完成</font>

4.jvm启动时的主类需要先被初始化

5.使用reflect包下的方法对某个类反射调用，如果该类还没有初始化，先进行初始化

6.动态语言支持。。。。。。

###### 接口初始化的触发时机：（接口编译后同样生成class文件，成员变量为public static final；成员方法为default/public abstract）

跟类初始化接近，区别在于：<font color ="red">类初始化的时候要求所有父类都初始化完成；接口初始化不要求所有父接口初始化完成</font>



#### Java类加载过程：

##### 加载

1.根据类的全限定名获取二进制字节流

<font color = "orang">（注1：对于非数组类型，直接使用jvm自带的引导类加载器（bootstrap）/用户自定义的类加载器进行加载。而数组类本身不是通过类加载器创建，由jvm直接在内存中动态构造。加载过程取决于数组元素的类型：</font>

<font color = "orang">数组元素为引用类型，递归加载该引用类型，数组类会被标志在加载该引用类型的类加载器的类名称空间上；</font>

<font color = "orang">数组元素非引用类型，将数组类标记，表示其和引导类加载器（bootstrap）相关联。）</font>

<font color = "orang">（注2：数组元素如果是引用类型，数组类的访问权限和数组元素一致；否则为public）</font>

2.<font color = "red">将字节流所代表的静态存储结构转为方法区的运行时数据结构（写入运行时常量池）</font>

3.在方法区生成代表这个类的class对象<font color = "orang">（此时类成员还没有在class对象中分配空间和赋值）</font>

##### 验证：验证class文件安全合法性

1.文件格式验证（class文件中各字段的数值范围 类型等是否合法）

2.元数据验证（校验类的元数据，比如非Object但没有父类、重写父类final方法、继承final类。。。）

3.字节码验证（校验方法体会不会违法：比如非法的类型转换、跳转指令指示到方法体以外，StackMapTable作为Code属性的子属性，加快这一步骤）

4.符号引用验证（用于解析阶段，which将[符号引用](E:\类文件结构1.md)转为直接引用，主要检验对应的字段、方法、类是否存在和可访问性。）

##### 准备：为类变量分配空间和赋值<font color = "orang">（在class对象中分配空间和赋值；实例变量要等到实例化对象的时候一起分配在堆中）</font>

{该类变量有[ConstantValue属性](E:\类文件结构1.md)，则赋值为final对应的定义值；（**如果给final变量用一个函数定义赋值，那么还是要执行对应的定义函数赋值，**但是在执行定义函数之前，短暂赋值为默认零值）

{否则，赋值为默认零值：int -> 0 , boolean -> false , float -> 0.0f

##### 解析：将常量池内的符号引用转为直接引用（直接引用：指向实际目标的指针/相对偏移量/间接定位到实际目标的句柄）

解析结果缓存（lambda 接口的default方法底层调用的invokedynamic指令除外，invokedynamic指令要求，程序运行到相应位置，才会进行解析）

###### 类或者接口解析：把一个符号引用/全限定名转为一个类或接口的直接引用（应该是对应class对象的直接引用）

如果这个类非数组类型，jvm把对应全限定名给当前代码所处的类加载器，加载这个类，这个过程中可能会需要递归地加载其父类或实现的接口<font color ="orang">（的确可能会触发当前类中的各成员类的加载，但是，如果没有满足本文档首部中定义的初始化触发时机，也不会触发成员类的初始化。）</font>

如果这个类是数组类型，先加载数组元素类型，jvm再生成一个代表其维度和元素类型的数组对象（应该是个类似于class对象的数组对象，而不是实际的数组示例）

再进行符号引用验证，权限检查

###### 字段解析：如果找得到，返回这个字段的直接引用

根据字段表的class_index找到该字段是属于哪个class的，解析这个class

如果这个class本身就有这个简单名称和描述符都符合的字段，返回直接引用

再递归看这个class实现的接口们以及对应的父接口们有没有

再递归看这个class的父类

找不到，NoSuchFieldError

权限检查

<font color ="red">从父类/父接口继承来的字段，不会写入字段表</font>

<font color ="orang">如果自己没有这个字段，但是它同时出现在父类和实现的接口中 / 同时出现在自己实现的多个接口 / 同时出现在父类实现的多个接口，Oracle的javac编译器会拒绝编译ambiguous</font>

###### 类方法解析：如果找得到，返回这个方法的直接引用

1.根据方法表的class_index找到该方法是属于哪个class的，解析这个class，并判断这个class是不是类，如果是接口，抛异常：

首先，类/接口的成员方法只有被引用到，即在其他方法中被使用到，对应的方法引用信息才会被写入到class常量池中，即会有一项Methodref / InterfaceMathodRef 信息。（对于类实现了接口的方法，那叫重写，<font color ="red">从接口/父类继承下来的方法只有被重写，才会被写入方法表</font>中，但是同样只有在其他地方被引用，才会有一项Methodref信息）

示例：

![](C:\Users\123\Pictures\JVM\方法被引用.png)

![](C:\Users\123\Pictures\JVM\MethodRef.png)

![](C:\Users\123\Pictures\JVM\接口方法被引用.png)

![](C:\Users\123\Pictures\JVM\InterfaceMethodRef.png)



其次，类方法 和 接口方法的常量项定义不同（即一个是MethodRef一个是InterfaceMethodRef），对于方法表中的某一项，其中的Code属性，会以索引的形式标志这个方法的引用信息，即是类方法还是接口方法，以及对应是属于哪一个class：如果某一个方法的Code表示它是一个MethodRef，但是这一项MethodRef指示的class是一个接口（可以指示的class的访问标志判断是类还是方法），会直接抛出java.lang.IncompatibleClassChangeError异常

![](C:\Users\123\Pictures\JVM\Code属性标志.png)

![](C:\Users\123\Pictures\JVM\MethodRef.png)

fun的 #9 指示它是一个类方法，如果后面的gc/Process2不是一个类而是接口，就会抛出java.lang.IncompatibleClassChangeError异常。

2.在这个类中查找，是不是真的有一个简单名称和描述符都符合的方法

3.如果没有，在其父类递归查找这个方法

4.如果还没有，在这个类实现的接口们以及对应的父接口查找，如果找到了，说明这个类实现了接口但是没有完全实现接口方法，这个类是一个抽象类，这个方法是一个抽象方法，调用抽象方法，抛出java.lang.AbstractMethodError。[关于此异常](https://blog.csdn.net/dnc8371/article/details/106706862)

5.就是找不到NoSuchMethodError

6.如果找到了，但是没权限访问，IllegalAccessError

###### 接口方法解析：如果找得到，返回这个方法的直接引用

和上面的类方法解析很类似，

先根据方法表的class_index，找到和解析出这个方法所属的class，判断这个class是不是一个接口；

然后在上一步找到的接口中找这个方法；

再在其多个（接口可以多继承）父接口中递归查找这个方法，如果找到就返回，<font color ="orang">如果多个父接口都有这个接口方法，看实际的javac编译器处理；</font>

如果真没有，NoSuchMethodError；

jdk9之前，接口的方法都是public，没有无权访问的问题。但是jdk9，接口可以有private static，所以之后就有可能无权访问了。

##### 初始化：执行类构造器\<clinit>方法（类变量赋值 + 静态代码块执行）

clinit方法由javac编译器自动生成，包含所有类变量的赋值和静态代码块语句，顺序和源文件一致

在静态代码块后面定义的类变量，代码块可以赋值但是不可以访问

如果类中没有静态代码块或者没有给类变量赋值，编译器可以不生成clinit方法

同一个类加载器下，一个类的类构造方法只会被执行一次；多个线程都要执行clinit时，只有一个线程能进入执行，其他线程阻塞

[父子类clinit init顺序](E:\class的字节码执行引擎)

<font color ="red">**类变量的赋值有两/三次：（clinit应该包含类变量定义、静态代码块）**</font>

一次是准备阶段，赋默认值/final值；

一次是初始化阶段执行clinit的类变量定义的时候，如 static int i = 12;//赋值为12

还可以有一次，在clinit的静态代码块中：static{ i = 13;}//赋值为13；

**<font color ="red">实例变量的赋值：（init包含实例变量定义、实例代码块、构造方法）最多四次</font>**

在创建对象中的初始化零值阶段，会给实例字段赋默认值（**final如果用函数赋值，对应函数的执行是在init函数之前，也就是说不管final是直接用常量还是函数定义的值，对应的赋值/执行定义函数都是在init之前就执行了的）**

然后在实例构造器\<init>中赋值：

先赋值为实例变量声明定义时的值

再赋值为init的实例代码块的值

再赋值为init的构造方法中的值

```java
public class ThreadLocalTest {
    private final int value = func();
    public static int val = 12;

    {
        System.out.println("实例代码块");
    }
    public ThreadLocalTest(){
        System.out.println("constror"+value);
    }
    private int func() {
        System.out.println("func" + value);
        return 123;
    }
    public int getValue(){
        return value;
    }

    public static void main(String[] args){
        System.out.println(new ThreadLocalTest().getValue());
        System.out.println(ThreadLocalTest.val);
    }
}

func0
实例代码块
constror123
123
12
```



##### 类加载器

###### 启动/引导类加载器 -> 扩展类加载器 -> 应用程序类加载器

类的唯一性由 加载它的类加载器 + 类本身，共同决定（class对象的isInstance()、equals()、isAssignableFrom()方法，和对象关系判断instanceof会受影响）

自定义类加载器时，把parent置为null，即把加载请求委派给引导类加载器处理

###### 双亲委派模型

`我们知道，每个 classloader 负责的加载域是不一样的，启动类加载器需根据 t 给出的类全限定名（如 com.Test）在其所负责的域里搜寻此类字节码，如果找到，则加载之；如果找不到，则表示无法加载，把代理权限往下**（父->子）**转移，直到某个加载器在负责的加载域中找到该类为止。`

###### 破坏双亲委派模型

双亲委派模型的**可见性**原则：假设ALoader是BLoader的上级，则B可以看到A加载的类，但是A看不到其子B加载的类

`Visibility principle allows child class loader to see all the classes loaded by parent ClassLoader, but parent class loader can not see classes loaded by child.`

JNDI服务：资源的集中管理 [关于JNDI](https://blog.csdn.net/wn084/article/details/80729230)

JNDI调用SPI接口：

[以数据库连接为例：](https://www.zhihu.com/question/49667892)

```
Connection connection = DriverManager.getConnection("jdbc:mysql://xxxxxx/xxx", "xxxx", "xxxxx");
```

java.sql.DriverManager本身由引导类加载器进行加载

Java方提供接口（如java.sql.Driver）,但是具体数据库驱动的实现由各厂商自己完成。这些其他厂商的具体实现类不可能放入lib让引导类加载器加载，只能由其他比如AppLoader加载

DriverManager使用ServiceLoader，通过扫包的形式拿到指定类，完成DriverManager初始化。导致整体的效果为：

BootstrapLoader加载的DriverManager，拿到了由AppLoader加载的其它厂商的具体实现类，违背了可见性

实际上：BootstrapLoader加载了SPI，SPI使用线程上下文类加载器（默认是AppLoader），显式调用这个子加载器，来调用服务实现方的具体实现类，造成整体上的一个破坏双亲委派模型的感觉。



