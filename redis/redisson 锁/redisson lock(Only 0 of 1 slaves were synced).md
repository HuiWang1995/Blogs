# redisson: Only 0 of 1 slaves were synced

近日部门微服务在生产上获取锁(rLock.tryLock())时报了如题的异常，这在redisson github issues 中有出现：
[issue-4222](https://github.com/redisson/redisson/issues/4222)

因此从服务端和客户端的双边实现进行了分析。
redis server版本为4.0.14，客户端redisson 3.17.4。

首先分析客户端到底发送了什么命令，如何判断slave 同步不满足条件

## 客户端

首先看抛出异常的位置，通过 res.getSyncedSlaves() < availableSlaves 判断已同步的从节点与总的可用的从节点的数量来判断是否满足条件。

其旨在让所有从节点均同步完设置锁的写操作，预防redis master 在发生故障后主从切换导致锁丢失的问题。

```java
// org.redisson.RedissonBaseLock#evalWriteAsync

    protected <T> RFuture<T> evalWriteAsync(String key, Codec codec, RedisCommand<T> evalCommandType, String script, List<Object> keys, Object... params) {
        MasterSlaveEntry entry = commandExecutor.getConnectionManager().getEntry(getRawName());
        int availableSlaves = entry.getAvailableSlaves();
        RFuture<BatchResult<?>> future = executorService.executeAsync();
        CompletionStage<T> f = future.handle((res, ex) -> {

            if (commandExecutor.getConnectionManager().getCfg().isCheckLockSyncedSlaves()
                    && res.getSyncedSlaves() < availableSlaves) {
                throw new CompletionException(
                        new IllegalStateException("Only " + res.getSyncedSlaves() + " of " + availableSlaves + " slaves were synced"));
            }
            return commandExecutor.getNow(result.toCompletableFuture());
        });
        return new CompletableFutureWrapper<>(f);
    }
```

接下来看看如何获取SyncedSlaves数量，是通过redis 提供的wait 命令，阻塞一定时间来等待主从同步完成。
> WAIT numreplicas timeout
>
> 此命令会阻塞当前客户端，直到所有先前的写命令都成功传输并至少被指定数量的副本确认。如果达到以毫秒为单位指定的超时，即使尚未达到指定的副本数，命令也会返回。
```java
// org.redisson.RedissonBaseLock#createCommandBatchService
// 等待完成主从同步的时间为1秒，在源码中是写死的。
BatchOptions options = BatchOptions.defaults()
        .syncSlaves(availableSlaves, 1, TimeUnit.SECONDS);


// org.redisson.command.CommandBatchService#executeAsync
        // ...        
        if (this.options.getSyncSlaves() > 0) {
            for (Entry entry : commands.values()) {
                    BatchCommandData<?, ?> waitCommand = new BatchCommandData(RedisCommands.WAIT,
            new Object[] { this.options.getSyncSlaves(), this.options.getSyncTimeout() }, index.incrementAndGet());
            entry.getCommands().add(waitCommand);
            }
        }
        // ...

        int syncedSlaves = 0;
        for (BatchCommandData<?, ?> commandEntry : entries){
            // 将wait 阻塞1秒后完成同步的从节点的数量返回
            if(isWaitCommand(commandEntry)){
                syncedSlaves=(Integer)commandEntry.getPromise().getNow(null);
            }
        }
        BatchResult<Object> result = new BatchResult<Object>(responses, syncedSlaves);
```
也就是说，客户端在主节点执行获取锁成功后，还会等待1秒的所有从节点同步完成，若没有完成同步则抛出异常。

如此一来，就需要清楚redis服务器的主从同步机制，是否能够在1秒内完成同步，若没有在1秒内完成同步是何问题？

## 服务端

具体的[主从同步](https://juejin.cn/post/7230769010908004389)机制不进行赘述,我们跟着问题来看1秒内能否完成同步。

redis 中的定时任务默认以1秒的频率进行执行，从节点通过发送偏移量到主节点，由主节点判断需要进行全量同步还是增量复制。
若是全量同步，那么当前一个节点接近10GB的内存使用，同步需要数分钟，那必然是会在客户端抛出异常；

若是增量复制，由于定时任务的间隔也是1秒，若在刚同步后立马获取了锁，且接下来1秒内写操作较多，也可能发生超出1秒多一些才能够同步完成（小概率）；
当然，若是遇上keys 等耗时命令（复杂度超过O(n)），亦或是bigKey等情况阻塞了master主线程，未完成同步也是正常的（这种情况需要排查一下慢日志）；


## 忽略主从同步

通过redisson配置，在获取锁后忽略主从同步等待，这异常就不会再抛出。
```properties
checkLockSyncedSlaves = false
```
优点在于不等待主从同步完成，获取锁的速度会变快。

缺陷在于，master服务端发生故障且未完成同步时，发生主从切换后，可能会被新的客户端获取该锁。这种情况下，业务的走向就需要有一定的考虑。


## 定制同步超时时间
在我们获取lock时，发现Redisson将自身的成员变量commandExecutor传入给RLock对象。
```java
// org.redisson.Redisson#getLock
    @Override
    public RLock getLock(String name) {
        return new RedissonLock(commandExecutor, name);
    }
```
并且在 tryLock() 时，RedissonBaseLock 必然会调用createCommandBatchService()创建一个批量命令执行服务
```java
// org.redisson.RedissonBaseLock#createCommandBatchService
 if (commandExecutor instanceof CommandBatchService) {
         return (CommandBatchService) commandExecutor;
         }
 BatchOptions options = BatchOptions.defaults().syncSlaves(availableSlaves, 1, TimeUnit.SECONDS);
 return new CommandBatchService(commandExecutor, options);
```
而commandExecutor若已经是CommandBatchService类型，则直接返回，因为其已有成员变量BatchOptions options，无需再额外配置。

那么问题就是，我们的Redisson对象（即RedissonClient）中的commandExecutor究竟是哪一种类型。继续通过Debug其注入过程，并查阅默认的源码：
```java
// org.redisson.Redisson#Redisson
protected Redisson(Config config) {
    this.config = config;
    Config configCopy = new Config(config);
    connectionManager = ConfigSupport.createConnectionManager(configCopy);
    RedissonObjectBuilder objectBuilder = null;
    if (config.isReferenceEnabled()) {
    objectBuilder = new RedissonObjectBuilder(this);
    }
    // 已经明确为 CommandSyncService
    commandExecutor = new CommandSyncService(connectionManager, objectBuilder);
    evictionScheduler = new EvictionScheduler(commandExecutor);
    writeBehindService = new WriteBehindService(commandExecutor);
    }
```
那么我们只要稍作修改，注入一个新的Redisson，其commandExecutor使用CommandBatchService 并配置好(BatchOptions)options 中的syncTimeout，
就可以增加获取锁后等待主从同步完成的超时时间。
```java
// org.redisson.Redisson#Redisson
protected Redisson(Config config) {
    this.config = config;
    // ...
    // 修改为 CommandBatchService，自行定义超时时间
    BatchOptions options = BatchOptions.defaults().syncSlaves(availableSlaves, 2, TimeUnit.SECONDS);
    commandExecutor = CommandBatchService(new CommandSyncService(connectionManager, objectBuilder), options);
    // ...
}
```
之后，需要获取锁的RedissonClient注入时选定为此对象即可。


## 新版本变化

查阅目前最新版本(redisson-3.21.0)中的源码，其判断逻辑发生了一些修改，在不忽略主从同步的情况下，只要有一台从节点同步成功则认为获取锁成功。
```java
// org.redisson.command.CommandAsyncService#syncedEval

if (getServiceManager().getCfg().isCheckLockSyncedSlaves()
        && res.getSyncedSlaves() == 0 && availableSlaves > 0) {
    throw new CompletionException(
            new IllegalStateException("None of slaves were synced"));
}
```

另外，对于availableSlaves 属性的获取，老版本是从配置中获取，而新版本会通过请求服务端`info replication`，获取连接着的slave数量。
应是解决了旧版本从节点宕机、网络不通情况下，获取锁会抛出异常的情况。