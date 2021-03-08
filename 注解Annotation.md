#### [注解Annotation](https://www.runoob.com/w3cnote/java-annotation.html)

注解本质上是个接口，一个注解和一个保留策略相关联、和多个ElementType相关联

@Retention @Documented @Target @Inherited为元注解，标注其他注解属性

@Inherited标志注解属性，被它标志的注解A，如果修饰类，那么其子类会自动也拥有这个注解A



2021.3.4

再看注解相关，个人理解：注解其实就是一个接口，给代码中的某些地方（ElementType）标记，提醒编译器/虚拟机去做某些事情，比如提醒编译器提醒某个方法是不是重写了父类方法。为了起到这个作用，有时候需要给接口内的字段赋值。

