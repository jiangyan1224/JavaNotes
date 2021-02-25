##### class的字节码执行引擎：输入字节码二进制流，输出执行结果

栈帧：局部变量表 + 操作数栈 + 动态连接 + 返回地址

<font color = "orang">局部变量表的大小、操作数栈的深度，在编译源文件生成class文件的时候就已经确定</font>

当前栈帧-当前方法：只有位于栈顶的栈帧/方法是正在被执行的

###### 局部变量表：方法参数（包括实例方法的隐藏参数this）+ catch定义的异常 + 方法内部定义的局部变量

为了不浪费内存，上述内容不会一次性全部存入局部变量表，jvm会根据变量的作用域来确定局部变量表的大小

局部变量表分配内存的最小单位是**变量槽slot**

一个变量槽要求可以放入32位及以下大小的变量，64位用两个变量槽存储

（reference类型，可以根据引用，直接/间接访问到 实例数据 和 实例所属类型数据，也放于局部变量表

returnAddress类型：已经很少用，异常的跳转改用异常表实现，也放于局部变量表）

<font color = "red">变量槽的分配：如果是实例方法，0号索引存的是**"this"引用**，然后按照参数表顺序排列存储形参变量值；然后按照其他局部变量的出现顺序和作用域依次分配变量槽（**retun值**会把这个值放入最后一个变量槽，在ireturn之前重新读入操作数栈顶，再作为返回值使用）</font>

局部变量表的访问：通过索引实现，如果要访问的是32位类型，就访问索引为N的变量槽；64位，访问索引为N和N+1的变量槽

局部变量可作为GCRoots，如果代码执行已经离开了某个变量的作用域，对应的变量槽就可以被复用，对应的空间就可以被回收（如果不实际用其他变量复用空间，即时编译器也可以优化，让内存正常被回收）

类变量即使没有被在代码中显式赋值，在类的准备阶段也可以赋默认值，所以代码中可以不赋值直接使用；但是局部变量如果不显式赋值，无法直接使用

###### 操作数栈：

32位数据所占栈容量为1；64位占据容量为2

在方法刚开始执行的时候，操作数栈为空，期间会不断地写入和弹出内容；调用其他方法也通过操作数栈传递参数值

操作数栈中的类型和对应字节码指令中的类型必须严格一致，比如字节码需要两个int，操作数栈不能弹出long/float...

这个代码，出现异常、和不出现异常，返回值分别应该是多少？

```java
public int inc(){
	int x;
	try{
		x = 1;
		return x;
	}catch (Exception e){
		x = 2;
		return x;
	}finally{
		x = 3;
	}
}
```

try:首先变量槽1号写入值为1，最后一号写入x当前值1并读入操作数栈顶，作返回值

catch:更新变量槽1号的值为2，在最后一号写入2并读入操作数栈顶，作返回值

finally:更新变量槽1号的值为3. （出现异常返回2，否则返回1）

<font color ="orang">如果try块 或者 catch块中可以成功执行return/遇到没有处理的异常，方法就直接返回，不会继续执行try_catch块后面的代码（如果遇到没有处理的异常，说明压根就没有try_catch，直接从出现异常的这一句结束方法执行并返回），但是finally不管怎样一定会执行</font>

###### 方法返回地址：

方法只有两种退出方法：

1.遇到return，正常退出：返回地址一般是主调方法的PC计数器值

2.遇到异常，且这个异常没有在这个方法内得到处理：返回地址要通过异常处理器表来确定

##### 方法调用/动态连接：确定被调用的方法是哪一个。

##### 有些调用在类加载的解析过程就可以确定方法的直接引用<font color ="red">（解析调用）</font>；但是另外一些调用需要到运行期间才能确定方法的直接引用<font color = "red">（分派调用）</font>。

###### 解析调用：编译的时候，被调用的方法就已经确定，在类加载的解析阶段直接就能生成对应的直接引用

满足解析引用的有invokestatic指令 和 invokespecial指令。对应的有静态方法、私有方法、实例构造器、父类方法、和 final方法，即 非虚方法

静态方法只能属于类型，不能被覆盖/隐藏

###### 分派调用：静态、动态、单分派、多分派

**静态类型&实际类型：**

```java
Human man = new Man();//对于这一句代码，Human为变量man的静态类型；Man为变量man的实际类型。
```

静态类型只会在使用的时候变化，且最终的静态类型变化结果在编译期是可以确定的；

而实际类型变化的结果只能在运行期确定

```java
Human human = (new Random()).nextBoolean() ? new Man() : new Woman();//实际类型的变化
(Man) human//静态类型的变化
(Woman) human//静态类型的变化
```

静态分派：静态分派实际上是静态的，发生在编译阶段；

<font color ="red">**重载是静态分派最典型的应用**，编译器在通过形参的类型 数量等来判断使用哪个重载版本；且使用形参的**静态类型**作为判断依据，所以编译阶段就可以确定方法版本</font>

```java
public class StaticDispatch{
	static abstract class Human{}
    static class Man extends Human{}
    static class Woman extends Human{}
    public void sayHello(Human guy){
        System.out.println("hello,human");
    }
    public void sayHello(Man guy){
        System.out.println("hello,Man");
    }
    public void sayHello(Woman guy){
        System.out.println("hello,Woman");
    }
    public static void main(String[] args){
        StaticDispatch sr = new StaticDispatch();
        Human man = new Man();
        Human woman = new Woman();
        sr.sayHello(man);
        sr.sayHello(woman);
    }
}
//输出human 和 human，因为man和woman的静态类型都是Human
```

但是字面量并没有显式的静态类型，只能选择一个“更合适”的版本

char -> int -> long -> float ->double  -> Character  ->装箱后引用类型实现的接口  ->  引用类型的父类（越往上层父类优先级越低）  ->可变参数（char... arg）

（char不能往byte short转换，因为可能会超出范围，不安全）

如果出现了多个相同优先级的重载方法，拒绝编译ambiguous

<font color= "red">**动态分派：运行期确定，重写是动态分派的一个表现，运行期根据其实际类型确定方法版本**</font>

```java
public class DynamicDispatch {
    static abstract class Human{
        protected abstract void sayHello();
    }
    static class Man extends Human{
        @Override
        protected void sayHello() {
            System.out.println("hello,Man");
        }
    }
    static class Woman extends Human{
        @Override
        protected void sayHello() {
            System.out.println("hello,Woman");
        }
    }
    public static void main(String[] args){
        Human man = new Man();
        Human woman = new Woman();
        man.sayHello();//这一句的字节码会把man变量，即方法所有者/接收者压入栈顶
        woman.sayHello();
        man = new Woman();
        man.sayHello();
    }
}
//hello,Man
//hello,Woman
//hello,Woman
```

字节码编译之后，出现的三个sayHello都是”invokecirtual Human方法“，而实际调用不同版本方法，是**通过invokevirtual指令实现：**

**1.找到处于栈顶的对象，确定其实际类型；**

**2.到这个实际类型里面找，有没有简单名称和描述符都符合的方法，并权限检查，如果有但是权限不够，返回异常**

**3.如果没有，到实际类型的父类递归查找**

<font color= "red">**方法有重写/重载，但是字段不存在这些，不参与多态。父子类有同名字段的时候，虽然子类内存会同时存在两个字段，但是子类字段会遮蔽父类对应字段（这里的“遮蔽”是指子类引用指向子类对象，通过这个引用访问字段时，虽然子类内存中有两个字段，但是访问结果是子类的字段值）**</font>

**字段的访问结果（实例.实例字段），应该是根据实例的静态类型决定**

```java
public class FieldHasNoPolymorphic {
    static class Father{
        public int money = 1;
        Father(){
            money = 2;
            showMeTheMoney();//invokevirtual
        }
        public void showMeTheMoney(){
            System.out.println("father has money "+money);//访问的是父类的字段
        }
    }
    static class Son extends Father{
        public int money = 3;
        Son(){
            money = 4;
            showMeTheMoney();//invokevirtual
        }
        public void showMeTheMoney(){
            System.out.println("son has money "+money);//访问的是子类的字段
        }
    }
    public static void main(String[] args){
        Father guy = new Son();
        System.out.println(guy.money);
    }
}
//son has money 0
//son has money 4
//2
```

<font color ="red">**父clinit -> 子clinit -> 父init ->子init**</font>

[类变量赋值&实例变量赋值](E:\Java类加载)

##### 只有当一个类中没有定义任何构造函数，编译器才会为其创建一个无参构造方法

##### 调用子类构造方法的时候，如果第一行没有显式调用super，会默认的调用super()执行父类无参构造方法

##### 如果显式调用了super，super()调用父类无参init；super(xx,xx...)调用父类有参init

**clinit包含静态变量赋值、静态代码块；init包含实例变量赋值、实例代码块、构造方法（一定是最后一个执行）**

子类加载之前必须先加载父类；**子类的实例构造器默认第一行是“super()”**

new对象的时候，

先进行“初始化零值”阶段，实例字段赋0值；

然后init，先实例字段定义赋值和实例代码块，最后是构造方法



##### JVM动态分派的实现：

在方法区建立一个虚方法表，存储各个方法的实际地址入口（与接口相关的还可以有一个接口方法表）

<font color ="orang">如果子类没有重写父类的方法，那么子类虚方法表中这一项的地址和父类对应方法一致；</font>

<font color ="orang">如果子类重写了，就指向子类实现版本的地址入口</font>

<font color ="orang">虚方法表一般在类加载的连接阶段中，给类变量赋默认值之后，初始化虚方法表</font>



##### 动态类型语言支持：

Java有的检查是在编译期间执行，有的是在运行期执行

Java在编译的时候，就已经把方法完整的符号引用（静态类型信息）写入到class文件；但是动态类型语言，如python，变量没有类型，变量值才有类型（应该是可以看作是没有静态类型只有实际类型）

动态类型语言只有在运行期，才知道变量值的类型，导致Java虚拟机无法直接处理动态类型语言，那么如何在运行期动态获取方法的所属类型，并运行？  -->>  invoke包和invokedynamic指令的出现：

###### invoke包：

方法句柄MethodHandle，感觉类似于C的方法指针：

```java
//句柄使用
public class Invoke_ReflectTest {
    static class classA{
        public void printUser(){
            System.out.println("classA");
        }
    }

    public static MethodHandle getPrintUserMH(Object receiver) throws NoSuchMethodException, IllegalAccessException {
        //方法类型，包含方法返回值和方法参数
        MethodType methodType = MethodType.methodType(void.class);
        //lookup()在某个类中查找指定名字和方法类型的方法，返回具有调用权限的方法句柄
        //bindTo绑定this
        return lookup().findVirtual(receiver.getClass(),  "printUser", methodType).bindTo(receiver);
    }
    public static void main(String[] args) throws Throwable {
        //反射reflect
        Class initialClass = Class.forName("invoke_reflect.User");
        Constructor constructor = initialClass.getConstructor(int.class, String.class);
        Object object = constructor.newInstance(12, "jy");
        System.out.println(object.toString());//object的实际类型是User
        Method method = initialClass.getMethod("printUser");
        method.invoke(object);

        //方法句柄invoke
        Object obj = System.currentTimeMillis() % 2 == 0 ? new User(13,"jy") : new classA();
        getPrintUserMH(obj).invokeExact();

    }
}
```

reflect & invoke：

- Method和MethodHandle所含内容不同，Method包含的东西比MethodHandle多很多

- 反射模拟的是JavaAPI的方法调用；而句柄的使用是对字节码指令的模拟，所以在字节码指令的优化也可以用在invoke上

- 方法句柄中的findStatic findSpecial findVirtual等是对invokestatic invokespecial invokevirtual等指令的模仿，相对反射还有权限检查的校验行为

而且invoke是为了支持其他的动态类型语言包括Java，反射只用于Java

###### invokedynamic指令：（Java的lambda、接口默认方法）

执行过程模拟：每一处含有invokedynamic指令的位置都被称作 动态调用点

`invokedynamic是java7引入的新指令，主要是用来支持动态语言的方法调用。将调用点 抽象成一个java类，并将原本由jvm控制的方法调用以及方法链接暴露给了应用程序。在运行过程中，每条invokedynamic指令都会绑定一个调用点，且会调用该调用点链接的方法句柄。在第一次执行invokedynamic时，jvm会调用该指令对应的启动方法，来生成调用点，并绑定到invokedynamic指令里。之后再次运行的话，就直接调用绑定的调用点链接的方法句柄。`[关于](https://ershi.blog.csdn.net/article/details/84632879?utm_medium=distribute.pc_relevant.none-task-blog-BlogCommendFromBaidu-1.control&depth_1-utm_source=distribute.pc_relevant.none-task-blog-BlogCommendFromBaidu-1.control)

```java
public class InvokeDynamicTest {
    public static void main(String[] args) throws Throwable {
        INDY_BootstrapMethod().invokeExact("icyfenix");
    }

    public static void testMethod(String s){
        System.out.println("Hello String: "+s);
    }

    public static CallSite BootstrapMethod(MethodHandles.Lookup lookup, String name, MethodType mt) throws NoSuchMethodException, IllegalAccessException {
        return new ConstantCallSite(lookup.findStatic(InvokeDynamicTest.class, name, mt));
    }

    private static MethodType MT_BootstrapMethod(){
        return MethodType.fromMethodDescriptorString("(Ljava/lang/invoke/MethodHandles$Lookup;Ljava/lang/String; " +
                "Ljava/lang/invoke/MethodType;)Ljava/lang/invoke/CallSite;",null);
    }

    /**从当前这个类中，找到名为BootstrapMethod、方法类型为MT_BootstrapMethod的方法句柄 即BSM句柄*/
    private static MethodHandle MH_BootstrapMethod() throws NoSuchMethodException, IllegalAccessException {
        return lookup().findStatic(InvokeDynamicTest.class, "BootstrapMethod", MT_BootstrapMethod());
    }

    /**绑定调用点CallSite对象
     * MH_BootstrapMethod()为BSM句柄，通过这个句柄，找到在本类中，某个方法名和某个方法类型的方法句柄
     * */
    private static MethodHandle INDY_BootstrapMethod() throws Throwable {
        CallSite cs = (CallSite) MH_BootstrapMethod().invokeWithArguments(lookup(),
                "testMethod", MethodType.fromMethodDescriptorString("(Ljava/lang/String;)V", null));
        return cs.dynamicInvoker();
    }
}
```

```java
public class FinalTest {
    static class Test{
        class GrandFather{
            void thinking(){
                System.out.println("i am grandfather");
            }
        }

        class Father extends GrandFather{
            void thinking(){
                System.out.println("i am father");
            }
        }

        class Son extends Father{
            //https://stackoverflow.com/questions/60322808/i-want-to-print-hi-grandfatherbut-it-seems-to-print-hi-father
            void thinking(){//在这里调用其祖父类的thinking方法
                //0
                super.thinking();// i am father
                MethodType mt = MethodType.methodType(void.class);
                try {
                    //1
                    MethodHandle mh = lookup().findSpecial(GrandFather.class, "thinking", mt, getClass());
                    mh.invoke(this); // i am father
                    //2
                    mh = lookup().in(Father.class).findSpecial(GrandFather.class, "thinking", mt, Father.class);
                    mh.invoke(this); // i am grandfather
                    //3  LookUp绕过访问保护
                    Field lookupImpl = MethodHandles.Lookup.class.getDeclaredField("IMPL_LOOKUP");
                    lookupImpl.setAccessible(true);
                    mh = ((MethodHandles.Lookup)lookupImpl.get(null))
                            .findSpecial(GrandFather.class, "thinking", mt, GrandFather.class);
                    mh.invoke(this);// i am grandfather
                } catch (NoSuchMethodException e) {
                    e.printStackTrace();
                } catch (IllegalAccessException e) {
                    e.printStackTrace();
                } catch (Throwable throwable) {
                    throwable.printStackTrace();
                }
            }
        }
    }
    public static void main(String[] args){
        new Test().new Son().thinking();
    }
}
```

##### 基于栈的字节码解释执行引擎

解释执行&编译执行

`字节码无法直接交给硬件执行需要虚拟机翻译成机器码才能执行，“翻译”的策略有两种：解释执行和编译执行。解释执行是没执行一句字节码的时候把字节码翻译成机器码并执行，优点是启动效率快，缺点是整体的执行速度较慢。编译执行预先把所有机器码编译成字节码并一起执行，其特点与解释执行相反，启动较慢执行较快。`

`在jvm虚拟机中是两者混合出现，既有解释执行也有编译执行。首先是解释执行，一条条执行所有字节码，如果JVM发现某个方法被频繁的调用会使用即时编译JIT，把该方法用编译执行的策略编译好，下次执行的时候直接调用机器码，这种方法被称为热点方法，由此可见编译执行是以方法为单位。`

`从业务的角度而言服务端和用户端对代码的执行速度和启动速度的要求是不一样的。比如移动端的应用程序，用户希望程序启动速度较快，服务端的程序，可能对程序的执行速度有更高的要求，为此从java7开始HotSpot采用了分层编译的方式，即引入了两种即使编译器：C1 C2。C1编译器称为client编译器，面向对启动性能有要求的用户端，编译时间段，优化策略简单；C2称为Serve驳岸一起面向对峰值性能有要求的服务器端，编译时间长，优化策略复杂。具体的在编译热点方法的时候先采用C1编译器，热点方法中的热点方法会被C2编译器再次编译。`[关于](https://www.cnblogs.com/AshOfTime/p/10551353.html)

基于栈的指令集&基于寄存器的指令集

istore：操作数栈 -> 局部变量表

iload：局部变量表 -> 操作数栈



###### 字节码生成&动态代理：Java的Proxy要求被代理类至少实现一个接口，且在实现了多个接口的时候，一次只能代理其中一个接口的方法

Proxy.newProxyInstance方法会动态生成一个class文件，如$Proxy0.class，并根据这个class生成一个对象obj，这个obj可以强转成被代理类实现的接口类型；之后obj.被代理类的方法  ->调用调用处理器的invoke方法  ->调用实际被代理对象的对应方法

[关于动态代理](https://www.zhihu.com/question/20794107/answer/658139129)

[动态代理源码解读](https://www.jianshu.com/p/9bcac608c714)

```java
package proxy;

import java.lang.reflect.InvocationHandler;
import java.lang.reflect.Method;
import java.lang.reflect.Proxy;

public class DynamicProxyTest {
    //接口
    interface IHello{
       void sayHello();
       void sayName();
    }
    interface IBye{
        void sayBye();
    }

    //被代理类
    static class Hello implements IHello,IBye{
        @Override
        public void sayHello(){
            System.out.println("hello");
        }

        @Override
        public void sayName() {
            System.out.println("jy");
        }

        @Override
        public void sayBye() {
            System.out.println("bye");
        }
    }

    //调用处理器，内部持有被代理类
    static class DynamicProxy implements InvocationHandler{
        Object proxyeeObj;//被代理对象

        //赋值被代理对象，调用Proxy的静态方法，返回一个代理对象实例
        public Object bind(Object proxyeeObj){
            this.proxyeeObj = proxyeeObj;
            //这个方法最终会生成一个$Proxy0.class文件，内含被代理对象实现的所有接口的所有方法Method
            return Proxy.newProxyInstance(proxyeeObj.getClass().getClassLoader(),
                    proxyeeObj.getClass().getInterfaces(),this);
        }
        //所有执行代理对象的方法，最终都会转化为对这个invoke方法的调用，最终是Method.invoke()方法，调用实际被代理对象实例的对应方法
        @Override
        public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
            System.out.println("proxyee method start!");
            method.invoke(proxyeeObj, args);//方法的动态分派
            System.out.println("proxyee method completed!");
            return null;
        }
    }

    //使用
    public static void main(String[] args){
//        System.getProperties().put("sun.misc.ProxyGenerator.saveGeneratedFiles", "true");
//        Hello helloObj = new Hello();//被代理类的实例
        //这里proxyHandler静态类型不能写InvocationHandler。就算方法版本最终看的是对象的实际类型，但是在编译成的class文件中
        //写的方法描述符写的也是父类InvocationHandler，更何况bind方法本来就不存在于父类InvocationHandler
//        InvocationHandler proxyHandler = new DynamicProxy();
        DynamicProxy proxyHandler = new DynamicProxy();//调用代理器的实例
        IBye helloProxy = (IBye) proxyHandler.bind(new Hello());
        IHello helloProxy1 = (IHello) proxyHandler.bind(new Hello());
        helloProxy1.sayHello();
//        System.out.println("-----------------");
        helloProxy1.sayName();
//        System.out.println("-----------------");
        helloProxy.sayBye();
    }
}

```

`method.invoke(proxyeeObj, args);//方法的动态分派`：

method.invoke(方法底层所属对象/null，new Object[]{实际参数})：静态方法可省略对象，直接用null替代



生成的$Proxy0类：

```java
//
// Source code recreated from a .class file by IntelliJ IDEA
// (powered by Fernflower decompiler)
//

package proxy;

import java.lang.reflect.InvocationHandler;
import java.lang.reflect.Method;
import java.lang.reflect.Proxy;
import java.lang.reflect.UndeclaredThrowableException;
import proxy.DynamicProxyTest.IBye;
import proxy.DynamicProxyTest.IHello;

//这里的$Proxy实现了两个接口，所以如果调用代理对象，只能强转成IHello/IBye两个接口类型
//一方面$Proxy已经继承了Proxy类，所以如果被代理类不实现接口，就没法再强转成和被代理类相关类型，也就无法访问被代理的方法
//一方面如果被代理类的方法是要加入被代理类实现的所有接口的所有方法，如果被代理类不实现任何接口，代理对象就不能加入被代理类的方法
final class $Proxy0 extends Proxy implements IHello, IBye {
    private static Method m1;
    private static Method m4;
    private static Method m3;
    private static Method m2;
    private static Method m5;
    private static Method m0;

    public $Proxy0(InvocationHandler var1) throws  {
        super(var1);
    }

    public final boolean equals(Object var1) throws  {
        try {
            return (Boolean)super.h.invoke(this, m1, new Object[]{var1});
        } catch (RuntimeException | Error var3) {
            throw var3;
        } catch (Throwable var4) {
            throw new UndeclaredThrowableException(var4);
        }
    }

    public final void sayHello() throws  {
        try {
            super.h.invoke(this, m4, (Object[])null);
        } catch (RuntimeException | Error var2) {
            throw var2;
        } catch (Throwable var3) {
            throw new UndeclaredThrowableException(var3);
        }
    }

    public final void sayName() throws  {
        try {
            super.h.invoke(this, m3, (Object[])null);
        } catch (RuntimeException | Error var2) {
            throw var2;
        } catch (Throwable var3) {
            throw new UndeclaredThrowableException(var3);
        }
    }

    public final String toString() throws  {
        try {
            return (String)super.h.invoke(this, m2, (Object[])null);
        } catch (RuntimeException | Error var2) {
            throw var2;
        } catch (Throwable var3) {
            throw new UndeclaredThrowableException(var3);
        }
    }

    public final void sayBye() throws  {
        try {
            super.h.invoke(this, m5, (Object[])null);
        } catch (RuntimeException | Error var2) {
            throw var2;
        } catch (Throwable var3) {
            throw new UndeclaredThrowableException(var3);
        }
    }

    public final int hashCode() throws  {
        try {
            return (Integer)super.h.invoke(this, m0, (Object[])null);
        } catch (RuntimeException | Error var2) {
            throw var2;
        } catch (Throwable var3) {
            throw new UndeclaredThrowableException(var3);
        }
    }

    static {
        try {
            m1 = Class.forName("java.lang.Object").getMethod("equals", Class.forName("java.lang.Object"));
            m4 = Class.forName("proxy.DynamicProxyTest$IHello").getMethod("sayHello");
            m3 = Class.forName("proxy.DynamicProxyTest$IHello").getMethod("sayName");
            m2 = Class.forName("java.lang.Object").getMethod("toString");
            m5 = Class.forName("proxy.DynamicProxyTest$IBye").getMethod("sayBye");
            m0 = Class.forName("java.lang.Object").getMethod("hashCode");
        } catch (NoSuchMethodException var2) {
            throw new NoSuchMethodError(var2.getMessage());
        } catch (ClassNotFoundException var3) {
            throw new NoClassDefFoundError(var3.getMessage());
        }
    }
}

```



[loadClass() & findClass() & defineClass()：](https://blog.csdn.net/mollen/article/details/100840940)

loadClass内部会递归调用父类加载器的loadClass，如果父类无法加载，会调用自己的findClass，返回class对象

findClass：

```java
protected Class<?> findClass(String name) throws ClassNotFoundException {
    throw new ClassNotFoundException(name);
}
class NetworkClassLoader extends ClassLoader {
        String host;
        int port;

         public Class findClass(String name) {
            byte[] b = loadClassData(name);
             return defineClass(name, b, 0, b.length);//调用defineClass，从byte数组构建class对象
         }

         private byte[] loadClassData(String name) {
             // load the class data from the connection
         }
}
```





