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
3. 当通道断开时，可以重连后清空本地缓存或重新加载;(需要测试一下，若真如此，则问题在于失联后且还未重连时数据可能会不新鲜)

## 对比Redis 6 的客户端缓存

两者的主角（用法）并不相同，在redis 6 的客户端缓存中，对开发者来说，自己一直在操作redis，客户端本地缓存是无感知的。
而RLocalCachedMap对开发者来说就是一个Map，借助redis完成一致性的实现是无感知的。

从通用性来看，RLocalCachedMap仅在Redisson中有，通用性低于redis 6 的客户端缓存。
只是目前来说，支持RESP3的客户端并不是十分完善。