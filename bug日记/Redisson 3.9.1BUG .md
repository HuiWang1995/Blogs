# Redisson BUG Failed to submit a listener notification task. Event loop shut down?



## 堆栈

```shell
ERROR 12561 --- [ Thread-3] i.n.u.c.D.rejectedExecution : Failed to submit a listener notification task. Event loop shut down?

java.util.concurrent.RejectedExecutionException: event executor terminated
at io.netty.util.concurrent.SingleThreadEventExecutor.reject(SingleThreadEventExecutor.java:842) ~[netty-common-4.1.29.Final.jar:4.1.29.Final]
at io.netty.util.concurrent.SingleThreadEventExecutor.offerTask(SingleThreadEventExecutor.java:328) ~[netty-common-4.1.29.Final.jar:4.1.29.Final]
at io.netty.util.concurrent.SingleThreadEventExecutor.addTask(SingleThreadEventExecutor.java:321) ~[netty-common-4.1.29.Final.jar:4.1.29.Final]
at io.netty.util.concurrent.SingleThreadEventExecutor.execute(SingleThreadEventExecutor.java:765) ~[netty-common-4.1.29.Final.jar:4.1.29.Final]
at io.netty.util.concurrent.DefaultPromise.safeExecute(DefaultPromise.java:768) [netty-common-4.1.29.Final.jar:4.1.29.Final]
at io.netty.util.concurrent.DefaultPromise.notifyListeners(DefaultPromise.java:432) [netty-common-4.1.29.Final.jar:4.1.29.Final]
at io.netty.util.concurrent.DefaultPromise.setSuccess(DefaultPromise.java:94) [netty-common-4.1.29.Final.jar:4.1.29.Final]
at io.netty.channel.group.DefaultChannelGroupFuture.setSuccess0(DefaultChannelGroupFuture.java:205) [netty-transport-4.1.29.Final.jar:4.1.29.Final]
at io.netty.channel.group.DefaultChannelGroupFuture.(DefaultChannelGroupFuture.java:121) [netty-transport-4.1.29.Final.jar:4.1.29.Final]
at io.netty.channel.group.DefaultChannelGroup.newCloseFuture(DefaultChannelGroup.java:449) [netty-transport-4.1.29.Final.jar:4.1.29.Final]
at io.netty.channel.group.DefaultChannelGroup.newCloseFuture(DefaultChannelGroup.java:430) [netty-transport-4.1.29.Final.jar:4.1.29.Final]
at org.redisson.client.RedisClient.shutdownAsync(RedisClient.java:334) [redisson-3.9.0.jar:na]
at org.redisson.connection.MasterSlaveConnectionManager.closeNodeConnections(MasterSlaveConnectionManager.java:215) [redisson-3.9.0.jar:na]
at org.redisson.cluster.ClusterConnectionManager.shutdown(ClusterConnectionManager.java:802) [redisson-3.9.0.jar:na]
at org.redisson.Redisson.shutdown(Redisson.java:661) [redisson-3.9.0.jar:na]
```

## 发生位置

spring boot 在启动时，创建Bean时，调用了RedissonClient的shutdown方法，抛出的异常。

## 版本

3.9.0 - 3.9.1 均有出现

## issue

经查为已知issue https://github.com/redisson/redisson/issues/1736

## 解决

将原先依赖的redisson-spring-boot-starter

```xml
        <dependency>
            <groupId>org.redisson</groupId>
            <artifactId>redisson-spring-boot-starter</artifactId>
            <version>3.9.1</version>
        </dependency>
```

改为

```xml
        <dependency>
            <groupId>org.redisson</groupId>
            <artifactId>redisson-spring-boot-starter</artifactId>
            <version>3.16.1</version>
            <exclusions>
                <exclusion>
                    <artifactId>redisson-spring-data-25</artifactId>
                    <groupId>org.redisson</groupId>
                </exclusion>
            </exclusions>
        </dependency>

        <dependency>
            <groupId>org.redisson</groupId>
            <artifactId>redisson-spring-data-20</artifactId>
            <version>3.16.1</version>
        </dependency>
```

即升高了redisson-spring-boot-starter版本，但仍选择redisson-spring-data-20对springboot 2.0.x进行兼容。