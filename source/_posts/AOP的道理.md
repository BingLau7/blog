---
title: AOP的道理
date: 2017-06-11 16:53:04
tags:
    - Spring
    - Java
categories: 造轮子
---

### 代理模式

#### 动机

在某些情况下，一个客户**不想或者不能直接引用一个对象**，此时可以**通过一个称之为“代理”的第三者来实现 间接引用**。代理对象可以在客户端和目标对象之间起到 中介的作用，并且可以**通过代理对象去掉客户不能看到的内容和服务或者添加客户需要的额外服务**。

<!-- more -->

通过引入一个新的对象（如小图片和远程代理对象）来实现对真实对象的操作或者将新的对象作为真实对象的一个替身，这种实现机制即为代理模式，通过引入代理对象来间接访问一个对象，这就是代理模式的模式动机。

#### UML

![Proxy UML](https://github.com/BingLau7/blog/blob/master/images/blog_22/Proxy.png?raw=true)

#### 代码实例

```Java
package io.github.binglau.proxy;

/**
 * 类ProxySubject.java的实现描述：TODO:类实现描述
 *
 * @author bingjian.lbj 2016-09-17 下午6:42
 */
public interface ProxySubject {
    void operation();
}
```

```Java
package io.github.binglau.proxy;

/**
 * 类RealSubject.java的实现描述：TODO:类实现描述
 *
 * @author bingjian.lbj 2016-09-17 下午6:42
 */
public class RealSubject implements ProxySubject {
    private String requestContent;

    public RealSubject(String requestContent) {
        this.requestContent = requestContent;
    }

    @Override
    public void operation() {
        System.out.println("RealSubject request");
    }
}
```

```Java
package io.github.binglau.proxy;

/**
 * 1. 保存一个引用使得代理可以访问实体。若RealSubject和ProxySubject的接口相同, Proxy会引用ProxySubject。
 * 2. 提供一个与ProxySubject的接口相同的接口,这样代理就可以用来替代实体。
 * 3. 控制对实体的存取,并可能负责创建和删除它。
 * 4. 其他功能依赖于代理的类型:
 *      1) Remote Proxy负责对请求及其参数进行编码,并向不同地址空间中的实体发送已编码的请求。
 *      2) Virtual Proxy可以缓存实体的附加信息,以便延迟对它的访问。
 *      3) Protection Proxy检查调用者是否具有实现一个请求所必需的访问权限。
 * @author bingjian.lbj 2016-09-17 下午6:42
 */
public class Proxy implements ProxySubject{
    private String requestContent;
    private RealSubject subject;

    public Proxy(String requestContent) {
        this.requestContent = requestContent;
    }

    @Override
    public void operation() {
        if (subject == null) {
            subject = new RealSubject(requestContent);
        }
        System.out.println("Proxy request");
        subject.operation();
    }
  
    public static void main(String[] args) {
        ProxySubject subject = new Proxy("test");
        subject.request();
    }
}
```

#### 使用环境

-  远程(Remote)代理：为一个位于不同的地址空间的对象提供一个本地的代理对象，这个不同的地址空间可以是在同一台主机中，也可是在另一台主机中，远程代理又叫做大使(Ambassador)。
-  虚拟(Virtual)代理：如果需要创建一个资源消耗较大的对象，先创建一个消耗相对较小的对象来表示，真实对象只在需要时才会被真正创建。
-  Copy-on-Write代理：它是虚拟代理的一种，把复制（克隆）操作延迟到只有在客户端真正需要时才执行。一般来说，对象的深克隆是一个开销较大的操作，Copy-on-Write代理可以让这个操作延迟，只有对象被用到的时候才被克隆。
-  保护(Protect or Access)代理：控制对一个对象的访问，可以给不同的用户提供不同级别的使用权限。
-  缓冲(Cache)代理：为某一个目标操作的结果提供临时的存储空间，以便多个客户端可以共享这些结果。
-  防火墙(Firewall)代理：保护目标不让恶意用户接近。
-  同步化(Synchronization)代理：使几个用户能够同时使用一个对象而没有冲突。
-  智能引用(Smart Reference)代理：当一个对象被引用时，提供一些额外的操作，如将此对象被调用的次数记录下来等。

### Java 中的动态代理

```Java
package io.github.binglau.proxy.static_proxy;

public interface Hello {  // 1. 公共接口
    void say(String name);
}
```

```Java
package io.github.binglau.proxy.static_proxy;

public class HelloImpl implements Hello { // 1. 委托类
    @Override
    public void say(String name) {
        System.out.println("Hello! " + name);
    }
}
```

```Java
package io.github.binglau.proxy.dynamic;

import io.github.binglau.proxy.static_proxy.Hello;
import io.github.binglau.proxy.static_proxy.HelloImpl;

import java.lang.reflect.InvocationHandler;
import java.lang.reflect.Method;
import java.lang.reflect.Proxy;

// 主要是解决静态代理中对于不同的接口都需要实现一个代理类的情况
public class DynamicProxy implements InvocationHandler {  // 2
    private Object target;

    public DynamicProxy(Object target) {
        this.target = target;
    }

    /**
    在代理对象调用任何一个方法时都会调用的，方法不同会导致第二个参数method不同，第一个参数是代理对象（表示哪个代理对象调用了method方法），第二个参数是 Method 对象（表示哪个方法被调用了），第三个参数是指定调用方法的参数。
    **/
    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        before();
        Object result = method.invoke(target, args);
        after();
        return result;
    }

    private void before() {
        System.out.println("Before");
    }

    private void after() {
        System.out.println("After");
    }

    @SuppressWarnings("unchecked")
    public <T> T getProxy() {
        /*
         * params:
         * 1. ClassLoader
         * 2. 该实现类的所有接口
         * 3. 动态代理对象
         */
        return (T) Proxy.newProxyInstance(
                target.getClass().getClassLoader(),
                target.getClass().getInterfaces(),
                this
        );
    }

    public static void main(String[] args) {
        Hello hello = new HelloImpl();
      
        DynamicProxy dynamicProxy = new DynamicProxy(hello);
       
        Hello helloProxy = dynamicProxy.getProxy(); // 3

        helloProxy.say("Jack");
        System.out.println(Proxy.getInvocationHandler(helloProxy)); // 获得代理对象对应的调用处理器对象
        /**
        Proxy 其他方法：
        Class getProxyClass(ClassLoader loader, Class[] interfaces): 根据类加载器和实现的接口获得代理类。
        **/
    }
}

/**
Before
Hello! Jack
After
**/
```

#### Java实现动态代理的大致步骤如下：

1. 定义一个委托类和公共接口。

2. 自己定义一个类（调用处理器类，即实现 `InvocationHandler` 接口），这个类的目的是指定运行时将生成的代理类需要完成的具体任务（包括Preprocess和Postprocess），即代理类调用任何方法都会经过这个调用处理器类。

3. 生成代理对象（当然也会生成代理类），需要为他指定：

   1. 委托对象
   2. 实现的一系列接口(3)调用处理器类的实例。

   因此可以看出一个代理对象对应一个委托对象，对应一个调用处理器实例。

### CGLib 动态代理

```Java
package io.github.binglau.proxy.cglib;

import io.github.binglau.proxy.static_proxy.Hello;
import io.github.binglau.proxy.static_proxy.HelloImpl;
import net.sf.cglib.proxy.Enhancer;
import net.sf.cglib.proxy.MethodInterceptor;
import net.sf.cglib.proxy.MethodProxy;

import java.lang.reflect.Method;

// 为了代理没有接口的类
public class CGLibProxy implements MethodInterceptor {
    public <T> T getProxy(Class<T> cls) {
        // Enhancer是CGLib的字节码增强器，可以方便的对类进行扩展，内部调用GeneratorStrategy.generate方法生成代理类的字节码
        return (T) Enhancer.create(cls, this);
    }

    // 同 JDK 动态代理一样，每个方法的调用都会调用 intercept 方法
    @Override
    public Object intercept(Object obj, Method method, Object[] args, MethodProxy proxy) throws Throwable {
        before();
        Object result = proxy.invokeSuper(obj, args);
        after();
        return result;
    }

    private void before() {
        System.out.println("Before");
    }

    private void after() {
        System.out.println("After");
    }

    public static void main(String[] args) {
        CGLibProxy cgLibProxy = new CGLibProxy();
        Hello helloProxy = cgLibProxy.getProxy(HelloImpl.class);
        helloProxy.say("Jack");
    }
}

/**
Before
Hello! Jack
After
**/
```

### Spring 的 AOP

AOP 框架不会去修改字节码，而是在内存中临时为方法生成一个 AOP 对象，这个 AOP 对象包含了目标对象的全部方法，并且在特定的切点做了增强处理，并回调原对象的方法。

Spring AOP 中的动态代理主要有两种方式，JDK 动态代理和 CGLIB 动态代理。具体可看上文介绍。

如果目标类没有实现接口，那么 Spring AOP 会选择使用 CGLib 来动态代理目标类。CGLib（Code Generation Library），是一个代码生成的类库，可以在运行时动态的生成某个类的子类，注意，CGLib 是通过继承的方式做的动态代理，因此如果某个类被标记为 `final` ，那么它是无法使用 CGLib 做动态代理的。



### 参考

[说说 cglib 动态代理](http://blog.jobbole.com/105423/)

[Spring AOP的实现原理](http://www.importnew.com/24305.html)

《从零开始写 Java Web 框架》
