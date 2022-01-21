# skywalking09 - 异步线程路线续接（中）

> 在上一篇中介绍了三种链路续接的方式，但是都需要我们按照其姿势去使用异步，对于我们项目里已经写好的lambda这样的匿名内部类的方式，改造起来将会改造量十分大，造出很多类。而官网中已经给出了更多的解决方案，将改造量进一步减少：https://skywalking.apache.org/docs/main/v8.4.0/en/setup/service-agent/java-agent/application-toolkit-trace-cross-thread/

## 无返回异步
RunnableWrapper.of()将lambda包裹即可。

```java
    ExecutorService executorService = Executors.newFixedThreadPool(1);
    executorService.execute(RunnableWrapper.of(new Runnable() {
        @Override public void run() {
            //your code
        }
    }));
```
## 带返回异步
CallableWrapper.of()将lambda包裹即可
```java
    ExecutorService executorService = Executors.newFixedThreadPool(1);
    executorService.submit(CallableWrapper.of(new Callable<String>() {
        @Override public String call() throws Exception {
            return null;
        }
    }));
```

## Supplier异步
SupplierWrapper.of()将lambda包裹即可
```java
    CompletableFuture.supplyAsync(SupplierWrapper.of(()->{
            return "SupplierWrapper";
    })).thenAccept(System.out::println);
```
