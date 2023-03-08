# Lettuce 客户端缓存 ClientSideCaching

> Lettuce 已经在2020年就支持了这个特性，详见[issue-1281](https://github.com/lettuce-io/lettuce-core/issues/1281)

## Example

官方示例
```java
// the client-side cache
Map<String, String> clientCache = new ConcurrentHashMap<>();

// prepare our connection and another party
StatefulRedisConnection<String, String> otherParty = redisClient.connect();
RedisCommands<String, String> commands = otherParty.sync();

StatefulRedisConnection<String, String> connection = redisClient.connect();

// Create cache-frontend through which we're going to access the cache
CacheFrontend<String, String> frontend = ClientSideCaching.enable(CacheAccessor.forMap(clientCache), connection,
        TrackingArgs.Builder.enabled());

// make sure value exists in Redis
// client-side cache is empty
commands.set(key, value);

// Read-through into Redis
String cachedValue = frontend.get(key);
assertThat(cachedValue).isNotNull();

// client-side cache holds the same value
assertThat(clientCache).hasSize(1);

// now, the key expires
commands.pexpire(key, 1);

// a while later
Thread.sleep(200);

// the expiration reflects in the client-side cache
assertThat(clientCache).isEmpty();
```
ClientSideCaching 将一个本地Map 与一个Redis连接StatefulRedisConnection 进行组合，通过enable 方法
返回一个CacheFrontend<K, V> 接口对象，实际实现为ClientSideCaching 。

## 实现方式

1. 通过对ClientSideCaching进行get/put操作，会通过成员变量RedisCache<K, V> redisCache 对Redis进行同步的操作。
2. 通过对connection添加PushListener监听，对“invalidate”消息，将其要删除的KEY，从本地的Map进行删除。
3. 从实现的方式上来说，走的是默认模式，而非BCAST模式。不过Lettuce也提供了参数可以进行更改，详见`io.lettuce.core.TrackingArgs`

## 存在的问题

1. 虽然连接会自动重连，但是在连接断开时，本地Map中数据不会清除（无监听断连）。即使重连后，也不会更新。所以一致性问题巨大。
2. 稍微有点离谱的是，在get为null时，通过valueLoader get实际数据后，要对redis 进行 set、get两次操作。
虽然get无可避免，需要进行一个get操作，redis服务器才会记录当前客户端存下了这个key，后续有更新才会通知，但是这个get大可走异步。
3. valueLoader.call() 获取实际数据值仍为null 时，将会抛出异常，即缓存中不能存null， 这点就需要开发注意。
4. 没有数量限制、没有清除策略、没有超时时间，也没有重连策略。

## 意外的发现

从前没有关注过Lettuce的具体实现方式，不过今天在看到`io.lettuce.core.api.sync.RedisServerCommands#clientTracking` 
这一个接口实际并无实现类。第一个可笑的想法是，Lettuce不会根本没实现就在忽悠咱吧。
然后DEBUG可以看到redisClient.connect()的返回结果属性`sync`实际是个代理对象，也就可以想到是动态代理了。
从下面代码可以看到代理实际去执行的方式：
```java
//io.lettuce.core.FutureSyncInvocationHandler#handleInvocation

@Override
protected Object handleInvocation(Object proxy, Method method, Object[] args) throws Throwable {
    try {
        Method targetMethod = this.translator.get(method);
        Object result = targetMethod.invoke(asyncApi, args);
        return result;
    } catch (InvocationTargetException e) {
        throw e.getTargetException();
    }
}
```
通过translator 这个Map映射到具体的方法，并将参数传递过去执行，具体的就不细讲了。