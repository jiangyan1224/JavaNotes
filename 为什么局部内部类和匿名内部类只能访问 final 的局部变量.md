## 为什么局部内部类和匿名内部类只能访问 final 的局部变量?

[链接](https://blog.csdn.net/tianjindong0804/article/details/81710268)

注：在Java8之后，对应的局部变量可以不用final修饰，但是实际上，这只是语法糖，编译器在底层还是帮忙给你加上了的，如果给这个局部变量初始化之后，如果再改动其值（基本类型改值，引用类型改引用），编译器会报错：

使用了匿名内部类Runnable实现类的代码示例：

```java
public static void main(String[] args){
        Vector<Integer> vector = new Vector<>();
        while(true){
            for (int i = 0; i < 10; i++) {
                vector.add(i);
            }
            Thread removeThread = new Thread(new Runnable() {
                @Override
                public void run() {
                    for (int i = 0; i < vector.size(); i++) {
                        vector.remove(i);//匿名内部类使用了vector局部变量
                        System.out.println(args.toString());//匿名内部类使用了args局部变量
                    }
                }
            });
            removeThread.start();
        }
    }
```

代码中使用了`new Runnable()`，实际上是创建了一个实现了Runnable接口的匿名内部类：

**反编译可以看到，这个类内部是有vector和args两个final成员变量的：**

![](C:\Users\123\Pictures\JVM\匿名内部类.png)

如果vector赋值之后，更改引用，会报错：因为违反了编译器自己给你加上的final特性：

![](C:\Users\123\Pictures\JVM\更改引用报错.png)

## 成员内部类、静态内部类、局部内部类、匿名内部类：

成员内部类的内部含有一个final的指向外围类实例的this，所以成员内部类实例的创建必须通过外围类的对象实例来创建：`Process1.InnerClass obj = new Process1().new InnerClass();`

成员内部类不能有静态方法成员，对于静态属性：必须是编译时常量：即static final+文本字符串/基本类型，否则编译报错

对于静态内部类，静态内部类的初始化和外围类的初始化相互独立：静态内部类的初始化不会触发外围类的初始化；外围类的初始化不会触发静态内部类的初始化（[类初始化的触发时机](E:\Java类加载)）

