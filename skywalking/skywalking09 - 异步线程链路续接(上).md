# skywalking09 - 异步线程链路续接（上）

> 不知道你是不是也出现过，明明打了@Trace，但是死活从http请求的链路进来看不到，反而，它还独自成一条链路，这样一来，根本连不成一个链路来追踪问题了。



## 断掉的链路

### 异步代码

此处，用了线程池去异步执行一个实现Runnable接口的类。

```java
    @Trace
    public void trace() throws InterruptedException {
        Thread.sleep(10);
        doNothing();
        executor.submit(new MyRunnable());
    }
```

以及该接口具体实现

```java
public class MyRunnable implements Runnable{

    @Override
    public void run() {
        System.out.println(TraceContext.traceId());
        doNothing();
    }

    @Trace
    private void doNothing(){
        return;
    }
}
```

### 链路断开示意图

理想中，我们自然也是希望异步线程中@Trace加注的方法也进入对应的链路，但是很遗憾，链路断成两条了：

![image-20211120181452346](/home/wanglh/Documents/Blogs/skywalking/pic/c09_1/01.png)

MyRunnable中的doNothing并没有在/trace/local这个http请求中打印出来。

## 连上的链路

### 链路连上示意图

![image-20211120181804259](/home/wanglh/Documents/Blogs/skywalking/pic/c09_1/02.png)

我们会发现，链路仍然有两条存在，但是，/trace/local这个http请求中打印出来多了两个Span，一个是异步线程，一个是添加了@Trace的自定义的方法。而且，**这两条链路的链路流水号是同一个**。

### 实现方式

- @TraceCrossThread
- @Async
- apm-jdk-threading-plugin

三种方式都可以实现，稍微做点比较。

#### `@TraceCrossThread`

`@TraceCrossThread`为skywalking提供的工具包中的注解，你可以认为其有一定的侵入性。使用方式则是在那个线程类上加上该注解，在类加载时，其构造方法会被代理agent做一次增强，具体的源码下一节说。

```java
@TraceCrossThread
public class MyRunnable implements Runnable{

    @Override
    public void run() {
        System.out.println(TraceContext.traceId());
        doNothing();

    }
}
```

#### `@Async`

@Async为spring-context包下的注解，对于java服务端来说，这个注解基本可以算无侵入了，毕竟是spring体系。不过使用方式和平时需要实现一个Runnable接口或继承Thread类差别挺大，个人觉得方便，但是在古董项目中、亦或是说服那些一年学习经验十年使用经验的老古董来说，推广起来还是比较头疼的。当然，如果是一个巨石的老项目，不推荐这样来改造了。

```java
@Service
public class TraceService implements InitializingBean {

    @Trace
    @Async
    public void traceAsync() throws InterruptedException {
        Thread.sleep(10);
        doNothing();

    }

    @Trace
    public void trace() throws InterruptedException {
        Thread.sleep(10);
        doNothing();
        executor.submit(new MyRunnable());
    }
}
```



关于这个注解的详细使用可以见：[JAVA多线程以及Spring异步注解@Async](https://blog.csdn.net/a17816876003/article/details/106448309)。效果图如下，其会多一个`SpringAsync`的链路，当然其链路流水号也是相同的：

![image-20211120183517423](/home/wanglh/Documents/Blogs/skywalking/pic/c09_1/03.png)

#### `apm-jdk-threading-plugin`

`apm-jdk-threading-plugin`这是官方提供的一个插件，是真正的无侵入了，使用方式也很简单，这个插件位于`${skywalking_dir}/agent/bootstrap-plugins`目录下，我们需要做的就是将其复制到`${skywalking_dir}/agent/plugins`目录下即可。

另外，还需要对代理的配置进行修改`${skywalking_dir}/agent/config/agent.config`，告诉代理对那些包下的线程池进行增强,在其底部添加这样的配置：

```properties
jdkthreading.threading_class_prefixes=com.aaa.bbb,com.bbb.ccc
```

当有多个包的时候，可以用英文的逗号进行分隔。



## 总结

多线程可能会导致链路断开，我们的链路续接不上，很多问题就没有办法查，要根据合适的方式，将链路接上。
