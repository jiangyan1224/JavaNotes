### Tomcat学习：

#### ch3：

[Digetster解析xml生成对象：](https://juconcurrent.com/2018/11/19/deep-understand-tomcat-006-digester/)，以及其中涉及到的[SPI和线程上下文类加载器](https://blog.csdn.net/yangcheng33/article/details/52631940)：

> SPI机制简介
> SPI的全名为Service Provider Interface，主要是应用于厂商自定义组件或插件中。在java.util.ServiceLoader的文档里有比较详细的介绍。简单的总结下java SPI机制的思想：我们系统里抽象的各个模块，往往有很多不同的实现方案，比如日志模块、xml解析模块、jdbc模块等方案。面向的对象的设计里，我们一般推荐模块之间基于接口编程，模块之间不对实现类进行硬编码。一旦代码里涉及具体的实现类，就违反了可拔插的原则，如果需要替换一种实现，就需要修改代码。为了实现在模块装配的时候能不在程序里动态指明，这就需要一种服务发现机制。 Java SPI就是提供这样的一个机制：为某个接口寻找服务实现的机制。有点类似IOC的思想，就是将装配的控制权移到程序之外，在模块化设计中这个机制尤其重要。
> SPI具体约定
> Java SPI的具体约定为：当服务的提供者提供了服务接口的一种实现之后，在jar包的META-INF/services/目录里同时创建一个以服务接口命名的文件。该文件里就是实现该服务接口的具体实现类。而当外部程序装配这个模块的时候，就能通过该jar包META-INF/services/里的配置文件找到具体的实现类名，并装载实例化，完成模块的注入。基于这样一个约定就能很好的找到服务接口的实现类，而不需要再代码里制定。jdk提供服务实现查找的一个工具类：java.util.ServiceLoader。

> 因为这句Class.forName(DriverName, false, loader)代码所在的类在java.util.ServiceLoader类中，而ServiceLoader.class又加载在BootrapLoader中，因此传给 forName 的 loader 必然不能是BootrapLoader。这时候只能使用TCCL了，也就是说把自己加载不了的类加载到TCCL中（通过Thread.currentThread()获取，简直作弊啊！）。上面那篇文章末尾也讲到了TCCL默认使用当前执行的是代码所在应用的系统类加载器AppClassLoader。
> ContextClassLoader默认存放了AppClassLoader的引用，由于它是在运行时被放在了线程中，所以不管当前程序处于何处（BootstrapClassLoader或是ExtClassLoader等），在任何需要的时候都可以用Thread.currentThread().getContextClassLoader()取出应用程序类加载器来完成需要的操作。
>
> 到这儿差不多把SPI机制解释清楚了。直白一点说就是，我（JDK）提供了一种帮你（第三方实现者）加载服务（如数据库驱动、日志库）的便捷方式，只要你遵循约定（把类名写在/META-INF里），那当我启动时我会去扫描所有jar包里符合约定的类名，再调用forName加载，但我的ClassLoader是没法加载的，那就把它加载到当前执行线程的TCCL里，后续你想怎么操作（驱动实现类的static代码块）就是你的事了。
>

[关于JNDI（Java Naming and Directory Interface）：](https://www.cnblogs.com/study-everyday/p/6723313.html)对于数据库连接，具体的url dbName等配置信息，不应该出现在具体使用代码中，而是使用JNDI，在配置文件中配置参数，具体代码中直接通过xml节点名称获取到具体的实例化后的数据源对象，再使用。

JNDI不止可以用在数据源设置，其他的查找和获取资源，甚至是其他应用组件，都可以用类似的方法进行管理。