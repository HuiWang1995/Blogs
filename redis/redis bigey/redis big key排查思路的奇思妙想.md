# redis big key排查思路的奇思妙想

<img src = "https://community-cdn-digitalocean-com.global.ssl.fastly.net/variants/9ZubAf8gaQpgHYcCnySh6jbw/035575f2985fe451d86e717d73691e533a1a00545d7230900ed786341dc3c882" alt="redis"/>

今天在秦晓辉的运维系统监控专栏交流群中，看到了几位朋友在讨论redis big key 扫描的方案。不自觉的来了兴致，参与了讨论。
并且有一些比较奇特的思路。

## 定义big key
为了让对redis较为陌生的朋友不清楚big key的含义有一定的认知。我们先来定义一下Big Key。
一切因为大，而导致redis去执行命令，网络传输而导致慢的key，都可以称为Big Key。
- 一个String的值特别大
- 一个List的元素特别多
- 一个Hash的元素特别多
- List、Hash中某个元素特别大

都可以称为Big Key。

那具体，多大才算大呢？那其实要看你具体业务的容忍度了，并不是一个很严格的限制。

这是我在知乎上看到一个博主对Big Key 大小的定义：

> - 一个STRING类型的Key，它的值为5MB（数据过大）
> 
> - 一个LIST类型的Key，它的列表数量为20000个（列表数量过多）
> 
> - 一个ZSET类型的Key，它的成员数量为10000个（成员数量过多）
> 
> - 一个HASH格式的Key，它的成员数量虽然只有1000个但这些成员的value总大小为100MB（成员体积过大）
 
我个人认为他对这个值度量定义的门槛颇低了，我目前开发维护的系统中，对一个String的Key，认为超过100KB就开始算大，超过1MB是严禁发生的。

## 如何排查Big Key
那如何排查Big Key呢？何时排查Big Key呢？

一般情况下，我们应该在第一次上生产之前，对系统进行全面的各项测试，其中就应该包括Big Key 的排查。
其次，就是在生产运行中，也许我们测试不够全面、也许多次迭代下来会有新的Big Key，我们应该由监控系统进行扫描排查。

对于Big Key的排查来说，那应该就是把所有的Key，按照我们的阈值进行比对其占用的内存大小，判断其是否为Big Key。
而我们知道，对于Redis 这种高性能内存缓存来说，我们都尽量使用一个O(1)算法复杂度的命令来调用，性能最佳。
而全部Key进行扫描，显然是一个O(n)的复杂度，将会阻塞Redis 相当长的时间。

而群里的讨论点在于：**在压测的过程中**，对redis big key进行扫描，并且尽量不影响性能。

那让我们来看看传统的方案，以及个人的奇思妙想。

### 官方解决方案

在[官方文档](https://redis.io/docs/ui/cli/) Scanning for big keys中如下描述：

> In this special mode, redis-cli works as a key space analyzer. It scans the dataset for big keys, but also provides information about the data types that the data set consists of. This mode is enabled with the --bigkeys option, and produces verbose output.

```bash
$ redis-cli --bigkeys

# Scanning the entire keyspace to find biggest keys as well as
# average sizes per key type.  You can use -i 0.01 to sleep 0.01 sec
# per SCAN command (not usually needed).

[00.00%] Biggest string found so far 'key-419' with 3 bytes
[05.14%] Biggest list   found so far 'mylist' with 100004 items
[35.77%] Biggest string found so far 'counter:__rand_int__' with 6 bytes
[73.91%] Biggest hash   found so far 'myobject' with 3 fields

-------- summary -------

Sampled 506 keys in the keyspace!
Total key length in bytes is 3452 (avg len 6.82)

Biggest string found 'counter:__rand_int__' has 6 bytes
Biggest   list found 'mylist' has 100004 items
Biggest   hash found 'myobject' has 3 fields

504 strings with 1403 bytes (99.60% of keys, avg size 2.78)
1 lists with 100004 items (00.20% of keys, avg size 100004.00)
0 sets with 0 members (00.00% of keys, avg size 0.00)
1 hashs with 3 fields (00.20% of keys, avg size 3.00)
0 zsets with 0 members (00.00% of keys, avg size 0.00)

```

也就是使用`redis-cli --bigkeys` 命令进行扫描，并会按照strings、lists、sets、hashs、zsets统计。

> The program uses the SCAN command, so it can be executed against a busy server without impacting the operations, however the -i option can be used in order to throttle the scanning process of the specified fraction of second for each SCAN command.

这个命令是通过`SCAN`指令进行实现的，并不是一次性直接对redis完成扫描，对于已经繁忙(处于)压测的服务器不会完全影响业务进行。
但是实际上，也会有一定的性能影响。这个方案，也是这位测开朋友在压测时采用的方案。

不过，他忽略了(当然，早期的客户端不一定支持这一参数)可以使用 `-i 0.01`参数可以更好的降低对redis服务器处理业务请求的影响。
`-i 0.01`代表着redis-cli这一客户端在每执行一次`SCAN`指令后，会暂停0.01秒的时间。
这一参数会导致big key 扫描本身耗时有一定增加，但是对redis服务器的压力就是降低许多，毕竟0.01秒对redis这种高性能的中间件来说，已经是一段不断的时间了。

所以，就目前来说，最方便也最稳妥的方式就是`redis-cli --bigkeys -i 0.01`。

### 解析RDB文件并统计
RDB文件作为redis的一个全量持久化文件。通过下载并对他的解析统计Big Key，这对Redis服务器的资源则没有任何消耗，是十分合理的。
但是若是为了扫描Big Key，在压测环境下执行`BGSAVE`这样的持久化指令，其fork进程的过程也会产生一定的阻塞，在Redis对他的标记上，也是@slow的。
所以扫描一个已经产生的RDB文件是可取的，特意去持久化一次，理论上对Redis产生的阻塞也是不小的。那具体其耗时如何，也未进行实验验证。

而这样的一个工具：[redis-rdb-tools](https://github.com/sripathikrishnan/redis-rdb-tools) 在github上也拥有4.8K的star，想必使用的用户也是不少的。
但是，我注意到了一点，这个工程最后一次提交在2020年。
而截止目前，redis在其后已经相继发布了redis 6、redis 7等更高的版本，并且对于RDB文件的格式，在redis 7.0 已经更新到了format 10。
这一“年久失修”的redis-rdb-tools 未必能够解析新版本的RDB文件。

### 网络统计
个人突发奇想，若是在网络层面上，通过抓包进行分析，对redis的TCP报文进行复制，然后使用Redis对应版本的RESP2、RESP3报文解析，不就可以分析这段时间内，客户端获取过的Big Key了吗？
这样对redis服务器的CPU等资源就没有什么消耗了。
当然，这个方案也有很明显的缺点，除了需要自行编写工具去分析以外，还存在很多分析不到的位置。
例如一个[`LRANGE`](https://redis.io/commands/lrange/)指令，对指定key的list进行范围扫描并返回:
```bash
LRANGE key start stop
```
它的复杂度是O(S+N)，其中S的复杂度与列表的HEAD或者TAIL的距离有关，而N则是范围内元素的个数。
所以当S很大，而N很小的时候，返回给客户端的数据量，其实还是小的，而它可能是一个Big Key，但是我们这个方案是没有机会发现它了。


### 新增从节点
个人觉得这是一个很妙的方式，具有可行性，但是也比较浪费，意义不是十分大的方案。

我们可以在压测开始前，通过slave of 命令，将我们新起的一个redis节点作为压测节点的从节点。并且这个节点对应用不可见。
那么我们在这个节点上进行big key的统计，就对业务没有任何影响了。

## 总结
官方提供的方式：`redis-cli --bigkeys -i 0.01` 应该是处理运行中的 redis big key 扫描的最佳方案了，当然，我们尽量避免高峰期去执行。

不知道是否还有其他的方式进行Big Key 的扫描呢~