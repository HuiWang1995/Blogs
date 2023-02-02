# redis 6.0的客户端缓存学习总结

> 官网原文: https://redis.io/docs/manual/client-side-caching/

客户端缓存是为了创建高性能的服务。有了客户端缓存，同样的key查询请求就可以在应用本地内存中获取到，无需网络请求。

    +-------------+                                +----------+
    |             |                                |          |
    | Application |       ( No chat needed )       | Database |
    |             |                                |          |
    +-------------+                                +----------+
    | Local cache |
    |             |
    | user:1234 = |
    | username    |
    | Alice       |
    +-------------+

而在客户端缓存上，计算机科学面临两个难题：

1. 如何使过时信息无效，避免其向用户呈现
2. 通过PUB/SUB的形式向客户端发送过期信息，有很大的带宽压力，即使那个客户端可能没有缓存该key。

所以redis6直接对其进行支持，提供了更可靠、更高效的实现。

redis的客户端缓存主要有两种模式：

1. redis服务端记录客户端缓存的key（默认）
2. redis客户端向服务端提供会缓存的key的前缀（广播模式）

## 服务端记录客户端缓存key的模式（默认）

- 服务端记录每个客户端缓存的key，在一个名为**Invalidation Table**的全局表里。
- 当客户端断开连接的时候，该客户端缓存的key会渐进式的进行垃圾回收。
  （若因网络情况发生这种情况的redis客户端，也应该清理自己的本地缓存，因为它很可能会有过期）
- 服务端不会为每个database提供一个缓存key的命名空间，因此，如果客户端在database 2 修改了key, 
  则该key在database 3 中没有被修改，也会被发送过期信息到客户端。这样的设计是为了减少内存开销，以及实现复杂度。
  _但是在我看来，我们应用系统往往都是在database 0 上进行读写的。_

在这种模式下，如果使用RESP2 协议（兼容老版本），则使用PUB/SUB命令进行实现。并且通过两个连接来维护。
一个连接进行正常的命令操作，一个连接专门用来接收无效信息（invalidation messages）。

但是如果使用RESP3 协议，无效信息将会使用push命令进行发送，无论是一个还是两个连接。

### 参数

- OPTIN
  
    CLIENT TRACKING on REDIRECT 1234 OPTIN

这个参数在开启TRACKING后，使得redis服务端不会记录redis客户端缓存了哪个key，除非客户端通知服务端进行缓存

    CLIENT CACHING YES
    +OK
    GET foo
    "bar"

**CACHING** 命令只会将紧跟其后的命令进行缓存，使用**MULTI** 命令会将一整个交易的key进行缓存，例如lua脚本的。

## 服务端广播模式

- 客户端开启客户端缓存时携带**BCAST**参数，并使用一个或多个**PREFIX** 参数声明需要缓存的key的前缀。
  并且前缀之间不可冲突。
- 服务端此时不再需要维护**Invalidation Table**，而是维护一张与每个客户端关联的**Prefixes Table**表。
- 所有符合前缀的key被修改后，都会通知对应的客户端进行无效。
- 这种模式服务端CPU的消耗和客户端注册的key的前缀数量成正比。
  如果是少量的，基本无影响。但是如果数量众多，就会十分影响CPU性能。
- 这种模式下，服务端可以优化，将过期指令合并成单个回复。

## 参数

- NOLOOP
  这个参数对两种模式都生效。
  默认情况下，服务端会将key失效信息发送给所有客户端，包括对这个key进行修改的客户端。
  但是对于那些实现先进的客户端，它们已经将自身的本地缓存做了修改，无需接收自己发起的修改的过期信息。
  使用这个参数，就可以来避免自己刚写的本地缓存被服务端通知清除。

## 避免紊乱情况

针对通过**两个连接**进行实现的客户端缓存，应当要通过占位符来避免因为两个连接接收服务端返回的响应顺序不一致而导致的问题。单连接的顺序有保证，不会有此问题。
例如下面这种情况，对旧版本请求的key的响应将出现问题，并缓存到了本地：

    [D] client -> server: GET foo
    [I] server -> client: Invalidate foo (somebody else touched it)
    [D] server -> client: "bar" (the reply of "GET foo")

那么通过一个占位符，来使得旧版本的请求即使返回，也不会被缓存。

    Client cache: set the local copy of "foo" to "caching-in-progress"
    [D] client-> server: GET foo.
    [I] server -> client: Invalidate foo (somebody else touched it)
    Client cache: delete "foo" from the local cache.
    [D] server -> client: "bar" (the reply of "GET foo")
    Client cache: don't set "bar" since the entry for "foo" is missing.

## 连接断开的情况

1. 本地缓存必须被清除
2. 不管是使用RESP2还是RESP3，都需要定期通过**PING**命令进行心跳检查，保持通道正常。
   
   > 所以在连接通道断开，且未被检测出来的时间段内，本地缓存可能存在过期情况！

## 什么应该被缓存

1. 不应该缓存那些会被连续性更新的key
2. 不应该缓存那些请求次数稀少的key
3. 应该缓存那些频繁被请求并且更新频率可接受的key

然而，简单的客户端实现往往只是随机的清除key，或使用LRU进行清除。

## 其他几个对客户端库的提示

1. 处理TTL（过期时间）
   在本地缓存页实现TTL，如果你想支持这个特性
2. 设置最大过期时间
   即使没有过期时间，但也建议本地有最大过期时间。防止产生bug或者连接问题而导致本地副本一直保留该数据。
3. 限制本地缓存的内存用量
   限制本地内存用量，并且在添加新的缓存key时，淘汰旧的缓存

## 学习总结

1. redis的客户端缓存有两种模式，默认模式会占用redis服务端的内存资源，广播模式会占用redis服务端的CPU资源。

2. 默认模式需要注意缓存的key的数量不能太大。

3. 广播模式需要注意缓存的key的前缀数量不能太大。（具体性能转折点在哪里需要自行测试）

4. 客户端缓存在网络问题导致连接通道断开且未被检测出来的时间段内，应用可能读到过期数据，实时性要求特别高的业务需要注意避免使用或者有兜底方案。