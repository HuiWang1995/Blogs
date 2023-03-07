# Redisson的本地缓存--RLocalCachedMap

> 在咨询Redisson官方后(https://github.com/redisson/redisson/issues/4837)，
> mrniko 表示RESP3 协议并不保证本地缓存的一致性，当客户端实例与redis服务端连接断开时。
> 然而他指出Redisson使用RLocalCachedMap 可以支持所有redis版本。

## RLocalCachedMap 使用DEMO

```java
// 由redisson客户端创建本地缓存
RLocalCachedMap<String, Integer> map = redisson.getLocalCachedMap("test", LocalCachedMapOptions.defaults());

String prevObject = map.put("123", 1);
String currentObject = map.putIfAbsent("323", 2);
String obj = map.remove("123");

// use fast* methods when previous value is not required
map.fastPut("a", 1);
map.fastPutIfAbsent("d", 32);
map.fastRemove("b");

RFuture<String> putAsyncFuture = map.putAsync("321");
RFuture<Void> fastPutAsyncFuture = map.fastPutAsync("321");

map.fastPutAsync("321", new SomeObject());
map.fastRemoveAsync("321");

// 销毁缓存
map.destroy();
```
## RLocalCachedMap 实现原理

实现主要在RedissonLocalCacheMap 中，不同于Redis 6 中的客户端缓存使用RESP3协议中提供的新命令来实现。
这里的实现用的是PUB/SUB。

RLocalCachedMap 将与Redis的交互封装在其中，对使用的开发者无感。
通过LocalCacheListener 对invalidationTopic 进行发送，监听信息。
1. 当put/remove发生时，通过topic发送事件;
2. 当topic监听到key发生update/remove时，对本地map进行update/remove;
3. 当重新订阅Topic时，可以重连后清空本地缓存或重新加载

## 对比Redis 6 的客户端缓存

两者的主角（用法）并不相同，在redis 6 的客户端缓存中，对开发者来说，自己一直在操作redis，客户端本地缓存是无感知的。
而RLocalCachedMap对开发者来说就是一个Map，借助redis完成一致性的实现是无感知的。

从通用性来看，RLocalCachedMap仅在Redisson中有，通用性低于redis 6 的客户端缓存。
只是目前来说，支持RESP3的客户端并不是十分完善。

在Redis服务断开后，RLocalCachedMap仍然可用，但将会出现集群中不同节点**数据不一致**问题。
而Redis 6 的客户端缓存在Redis服务断开后，将会清空本地缓存（在心跳检测到断开前，本地缓存仍然是可用的）。

在用途上，Redis 6 的客户端缓存配合Spring Data Cache的 @Cacheable等注解，能较大提升Redis缓存的性能，减少请求Redis的次数。
RLocalCachedMap则是借助Redis来完成一致性。

## 重要参数

```java

// LocalCachedMapOptions

    private ReconnectionStrategy reconnectionStrategy;
    private SyncStrategy syncStrategy;
    private EvictionPolicy evictionPolicy;
    private int cacheSize;
    private long timeToLiveInMillis;
    private long maxIdleInMillis;
    private CacheProvider cacheProvider;
    private StoreMode storeMode;
    private boolean storeCacheMiss;
    
```

### 重连策略
```java
    /**
     * Various strategies to avoid stale objects in local cache.
     * Handle cases when map instance has been disconnected for a while.
     *
     */
    public enum ReconnectionStrategy {
        
        /**
         * No reconnect handling.
         */
        NONE,
        
        /**
         * Clear local cache if map instance disconnected.
         */
        CLEAR,
        
        /**
         * Store invalidated entry hash in invalidation log for 10 minutes.
         * Cache keys for stored invalidated entry hashes will be removed 
         * if LocalCachedMap instance has been disconnected less than 10 minutes 
         * or whole local cache will be cleaned otherwise.
         */
        LOAD
        
    }
```
在关注一致性的情况下，`ReconnectionStrategy.CLEAR`会是更好的选择，在断开连接时就清空本地缓存。
**但是在Redisson 3.15.0 版本中实测，即使Redis服务都已经宕掉的情况下，本地缓存没有失效，且仍然可用。**
`LOAD`则是将10分钟内的缓存项从Redis进行读取更新。

### 同步策略
```java
    public enum SyncStrategy {
        
        /**
         * No synchronizations on map changes.
         */
        NONE,
        
        /**
         * Invalidate local cache entry across all LocalCachedMap instances on map entry change. Broadcasts map entry hash (16 bytes) to all instances.
         */
        INVALIDATE,
        
        /**
         * Update local cache entry across all LocalCachedMap instances on map entry change. Broadcasts full map entry state (Key and Value objects) to all instances.
         */
        UPDATE
        
    }
```
`INVALIDATE`将更新的HASH KEY 发布出去，通知订阅者更新。
`UPDATE`将**整个Map缓存对象发布**出去，这时候需要注意会不会成为一个Big Key, 阻塞Redis过久。

### 清理策略
```java
    public enum EvictionPolicy {
        
        /**
         * Local cache without eviction. 
         */
        NONE, 
        
        /**
         * Least Recently Used local cache.
         */
        LRU, 
        
        /**
         * Least Frequently Used local cache.
         */
        LFU, 
        
        /**
         * Local cache with Soft Reference used for values.
         * All references will be collected by GC
         */
        SOFT, 

        /**
         * Local cache with Weak Reference used for values. 
         * All references will be collected by GC
         */
        WEAK
    };
```
清理策略默认是NONE，是十分危险的，如果在不限制数量的情况下，个人建议配置为SOFT，至少不会造成OOM问题。

### 缓存数量
`cacheSize`定义缓存键值对的数量，建议配置个上限。

### TTL
`timeToLiveInMillis` 存活时间，根据业务设置，尽量不要永久。
### 最大空闲时间
`maxIdleInMillis` 可以配置与`timeToLiveInMillis`一致。

### 存储模式
```java
    public enum StoreMode {

        /**
         * Store data only in local cache.
         */
        LOCALCACHE,

        /**
         * Store data only in both Redis and local cache.
         */
        LOCALCACHE_REDIS

    }
```
默认为`LOCALCACHE_REDIS`