#### [注解Annotation](https://www.runoob.com/w3cnote/java-annotation.html)

注解本质上是个接口，一个注解和一个保留策略相关联、和多个ElementType相关联

@Retention @Documented @Target @Inherited为元注解，标注其他注解属性

@Inherited标志注解属性，被它标志的注解A，如果修饰类，那么其子类会自动也拥有这个注解A

