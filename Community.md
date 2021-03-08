### Community

##### 1.Spring.IOC 典型的工厂模式，通过sessionfactory去注入实例。

https://www.oschina.net/question/2491333_2271558

> 好处是，你用类A实现了一个接口I，在N个地方通过new A()，当需要把类A换为类B的时候，你要找到所有地方把 new A 改为new B。
>
> 如果通过Ioc，你在一个XML定义了Bean后，通过Autowired注入。如果你要换成其他实现，只要改XML定义的就可以，而不需要找到所有new
>
> 还有一种场景是在单元测试的时候，你可以通过import不同的配置，来导入不同的bean，例如有涉及外部资源的访问类，当要测试的时候，用MockBean代替原来的实体类。

https://stackoverflow.com/questions/2572158/what-is-aop-dependency-injection-and-inversion-of-control-in-simple-english

> Dependency injection was explained very well in [How to explain dependency injection to a 5-year-old?](https://stackoverflow.com/questions/1638919/how-to-explain-dependency-injection-to-a-5-year-old/1638961#1638961):
>
> > When you go and get things out of the refrigerator for yourself, you can cause problems. You might leave the door open, you might get something Mommy or Daddy doesn't want you to have. You might even be looking for something we don't even have or which has expired.
> >
> > What you should be doing is stating a need, "I need something to drink with lunch," and then we will make sure you have something when you sit down to eat.

##### 2.Spring.AOP 代理模式（CGlib）

https://www.cnblogs.com/mengdd/archive/2013/05/07/3065619.html代理模式类图

https://juejin.cn/post/6844903853813563405静态代理&动态代理

代理模式：代理对象内部持有被代理对象/实际真实对象的引用，在外界调用真实对象的方法时，该请求是被转发到代理对象上去的。代理对象拿到请求，可以让真实对象完成过响应，除此之外，代理对象自己也可以额外做一些事情，比如打印日志等等。

3.Bean的创建 初始化 和 销毁时机

在bean上注解scope为singleton 和 prototype的区别：

https://blog.csdn.net/j080624/article/details/79779571

> 单实例bean 初始化方法在容器创建时被调用，销毁方法在容器销毁时被调用。多实例bean，初始化方法在第一次获取bean的时候调用(非容器创建时，容器创建时会调用构造方法)，销毁方法容Spring 容器不负责管理。

##### 4.@Configuration + @Bean注解，可以实现让Spring管理第三方Bean，@Bean放在获得bean对象实例的方法上，且[Spring对标记为`@Bean`的方法只调用一次，因此返回的Bean仍然是单例。](https://www.liaoxuefeng.com/wiki/1252599548343744/1308043627200545)

##### 5.Spring MVC

我的理解：

首先，表现层 业务层 数据层指的是后台代码的组织方式（controller service dao包）；MVC感觉像是在浏览器请求到得到响应过程中涉及到的三个组件：model view 和controller。三者不是一一对应的：实际上，MVC三者都是在表现层工作：

如图2：前端控制器获得浏览器的请求，根据访问路径@RequestMapping找到对应是哪个Controller（表现层）负责处理；Controller到后面的业务层发起调用，再到后面的数据层获取数据，返回的是一个组装好的model（MVC的model）；前端控制器拿到model，传给view模板进行组装，最后给浏览器返回一个HTML。可以看出，前端控制器、 M、V、C这四个都是在表现层工作，且MVC受前端控制器调度。

（浏览器拿到HTML之后，如果里面涉及到其他文件，比如需要css文件、js文件、图片等资源，浏览器就需要再次向服务器发起请求。）

![](C:\Users\123\Pictures\JVM\Spring MVC.png)

![mvc](https://docs.spring.io/spring-framework/docs/4.3.26.RELEASE/spring-framework-reference/htmlsingle/images/mvc.png)

##### 6.Controller获取数据

```java
package com.nowcoder.community.controller;

import com.nowcoder.community.entity.Page;
import com.nowcoder.community.service.AlphaService;
import org.aspectj.lang.annotation.RequiredTypes;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Controller;
import org.springframework.web.bind.annotation.*;

import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;
import java.io.IOException;
import java.io.PrintWriter;
import java.util.Enumeration;

@Controller
@RequestMapping("/alpha")
public class AlphaController {

    @Autowired
    private AlphaService alphaService;

    @RequestMapping("/hello")
    @ResponseBody
    public String sayHello(){
        return "hello spring boot";
    }

    @RequestMapping("/data")
    @ResponseBody
    public String getData(){
        return alphaService.find();
    }

    @RequestMapping("/http")
    @ResponseBody
    public void getHTTP(HttpServletRequest request, HttpServletResponse response){
        //请求的第一行
        System.out.println(request.getMethod());
        System.out.println(request.getRequestURL());
        //请求头
        Enumeration<String> enumeration = request.getHeaderNames();
        while (enumeration.hasMoreElements()){
            String name = enumeration.nextElement();
            String value = request.getHeader(name);
            System.out.println(name + ": " + value);
        }
        //请求体
        System.out.println(request.getParameter("code"));

        response.setContentType("text/html;charset=utf-8");
        try (
                PrintWriter writer = response.getWriter();
                ) {
            writer.write("<h1>牛客网</h1>");
        } catch (IOException e) {
            e.printStackTrace();
        }
    }

    // /students?page=1&limit=10
    //GET 一般用于浏览器向服务器请求数据
    @RequestMapping(path = "/students",method = RequestMethod.GET)
    @ResponseBody
    public String getStudents(
            @RequestParam(value = "page", required = false, defaultValue = "1")int page,
            @RequestParam(value = "limit", required = false, defaultValue = "10")int limit){
        System.out.println(page + "," + limit);
        return "getStudents success";
    }

    // /student/123
    @RequestMapping(path = "/student/{id}", method = RequestMethod.GET)
    @ResponseBody
    public String getAStudent(@PathVariable int id){
        System.out.println(id);
        return "getAStudent success";
    }

    //POST 一般用于浏览器向服务器提交数据（GET也可以提交数据，但是参数都是在请求路径上，而请求路径url长度有限）
    @RequestMapping(path = "/addStu", method = RequestMethod.POST)
    @ResponseBody
    public String addStudent(String name, int age){
        System.out.println(name + "," + age);
        return "addStu success";
    }

    //实验，路径后面如果携带了page内部属性同名的参数，page是可以获取到对应值的
    //如 /alpha/student1?page=45&limit=2
    @RequestMapping(path = "/student1", method = RequestMethod.GET)
    @ResponseBody
    public String getPage(Page page){
        System.out.println(page.getCurrent());
        return "getPage success";
    }
}

```

##### 7.响应状态码

https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Status

###### **7.1 2xx：成功**

###### **7.2 3xx：重定向**

浏览器调用删除模块，删除模块本身没有什么网页能反馈给用户；所以一般会给用户反馈查询模块的调用结果；

但是如果在删除模块中直接调用查询模块，直接耦合，如果后面查询模块要改，导致删除模块也要改----这是个问题

这里就可以通过重定向解决：调用删除模块之后，服务器反馈302状态码和一个调用网址，表明“建议浏览器这边再去访问location这个网址”，这样，浏览器就再发起一次请求，调用查询模块，这样就避免了前面的耦合问题。

![image-20210308193542464](C:\Users\123\Pictures\JVM\重定向.png)

###### 7.3 4xx：404服务不存在（一般是路径有问题）

###### 7.4 5xx：500服务端出问题

