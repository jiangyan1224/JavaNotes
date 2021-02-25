#### Java泛型补充：

https://mp.weixin.qq.com/s/ysrehh2b7utL-Viw-Vf3yQ

```java
package syntactic_sugar;

import java.lang.reflect.InvocationHandler;
import java.util.*;

public class GenericTest {
    public <T>void test(T a){//使用T，需要在方法上/类上有所声明
        Class<T> clazzT;
    }

    //通过T确保泛型参数的一致性：要求T要是Number或者其子类型，并且dst src的T要是同一类型
    //就算dst src都是Number子类型，比如dst Float;src Integer，报错；
    //都是Float/Integer才行
    public static <T extends Number> void test1(List<T> dst, List<T> src){

    }

    public static void main(String[] args){
        List<Float> dst = new ArrayList<>();
        List<Float> src = new ArrayList<>();
        test1(dst, src);

        Integer a= 1;
        Integer b = 2;
        Integer c = 3;
        Integer d = 3;
        Integer e = 321;
        Integer f = 321;
        Long g = 3L;
        System.out.println( c == d);//true
        System.out.println(e == f);//false
        System.out.println(c == (a + b));//true
        System.out.println(c.equals(a + b));//true
        //System.out.println(g == d);错误: 不可比较的类型: Long和Integer
        System.out.println(g == (a + b));//true
        System.out.println(g.equals(a + b));//false
    }
}

```

