# Redis客户端配置优化建议

## Redis客户端配置项

> 参考spring-boot docs:[data-properties](https://docs.spring.io/spring-boot/docs/2.3.1.RELEASE/reference/html/appendix-application-properties.html#data-properties).本篇以lettuce为例.着重讲连接池的配置.



| 配置项                                                 | 默认值  | 描述                                                         | 中文描述                                                     |
| ------------------------------------------------------ | ------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| `spring.redis.lettuce.cluster.refresh.adaptive`        | `false` | Whether adaptive topology refreshing using all available refresh triggers should be used. | redis集群拓扑自动刷新                                        |
| `spring.redis.lettuce.cluster.refresh.period`          |         | Cluster topology refresh period.                             | 集群拓扑刷新周期。                                           |
| `spring.redis.lettuce.pool.max-active`                 | `8.0`   | Maximum number of connections that can be allocated by the pool at a given time. Use a negative value for no limit. | 连接池可以分配的最大连接数。使用负值表示无限制。             |
| `spring.redis.lettuce.pool.max-idle`                   | `8.0`   | Maximum number of "idle" connections in the pool. Use a negative value to indicate an unlimited number of idle connections. | 连接池可以分配的最大连接数。使用负值表示无限制。             |
| `spring.redis.lettuce.pool.max-wait`                   | `-1ms`  | Maximum amount of time a connection allocation should block before throwing an exception when the pool is exhausted. Use a negative value to block indefinitely. | 连接池资源耗尽时，连接尝试分配阻塞时间,超时即抛出异常。使用负值无限期阻塞。 |
| `spring.redis.lettuce.pool.min-idle`                   | `0.0`   | Target for the minimum number of idle connections to maintain in the pool. This setting only has an effect if both it and time between eviction runs are positive. | 连接池最小空闲连接数.仅在它和`time-between-eviction-runs`都为正时有效 |
| `spring.redis.lettuce.pool.time-between-eviction-runs` |         | Time between runs of the idle object evictor thread. When positive, the idle object evictor thread starts, otherwise no idle object eviction is performed. | 空闲对象逐出器线程的运行间隔时间。当为正值时，空闲对象逐出器线程启动，否则不执行空闲对象逐出。 |
| `spring.redis.lettuce.shutdown-timeout`                | `100ms` | Shutdown timeout.                                            | 关闭超时                                                     |



## 配置详解

### `spring.redis.lettuce.pool.max-active`

连接池最大的连接数.过少会导致竞争\阻塞.过多会浪费资源.

- 配置数量过少

  往往在开发环境时配置会比较低,在压测时会导致竞争激烈,多数线程被阻塞,导致TPS上不去.

  可以通过打印redis查询接口耗时发现,接口耗时不稳定.有些快的在1ms完成,慢的在40ms以上,甚至超时.**[建议查询耗时在1-3ms之间]**

  解决方法: 加大配置值.

  建议默认值: CPU*2

> [一次redis连接池连接数配置过少引起的性能问题](https://www.cnblogs.com/huantianxing/p/8632890.html)

- 配置数量过大

  配置数量过大不仅浪费资源,甚至可能抢占不到与redis的连接,这与redis服务器可连接的最大数有关.

> [处理redis连接数过多](https://blog.csdn.net/u013050593/article/details/56480358?utm_medium=distribute.pc_relevant_t0.none-task-blog-BlogCommendFromMachineLearnPai2-1.baidujs&dist_request_id=e2bf0fd8-5b3e-4d6a-a2df-2cb5c901ac14&depth_1-utm_source=distribute.pc_relevant_t0.none-task-blog-BlogCommendFromMachineLearnPai2-1.baidujs)

### `spring.redis.lettuce.pool.max-idle`

连接池最大的空闲数.过少会导致频繁释放\建立链接,十分耗时(**建立连接是耗时操作**).过多会浪费资源.

- 配置数量过少

  导致并发高时,需要新建与redis的连接.

  通过监控查看redis每秒新建连接数与当前连接数,逐步提高配置数量,以致达到预期.

> [一次redis调优——连接池优化](https://blog.csdn.net/asdasd3418/article/details/84639782)

### `spring.redis.lettuce.pool.max-wait`

连接尝试分配阻塞时间.过短会频繁抛出异常,在有旁路设计的系统中,压力就会宣泄到数据库中.过长或者无限制会导致接口响应时间过长.

### `spring.redis.lettuce.pool.min-idle`

连接池最小空闲连接数.

### `spring.redis.lettuce.pool.time-between-eviction-runs`

空闲对象逐出器线程的运行间隔时间.空闲连接线程释放周期时间.



## 配置推荐

### 开发环境

```properties
spring.redis.lettuce.pool.max-active = 2
spring.redis.lettuce.pool.max-idle = 2
```

### 生产环境

```properties
spring.redis.lettuce.pool.max-active = 大于cpu*2
spring.redis.lettuce.pool.max-idle = cpu*2
spring.redis.lettuce.pool.max-wait = 5s
spring.redis.lettuce.pool.min-idle = 0
spring.redis.lettuce.pool.time-between-eviction-runs = 1s
```



## 其他

> lettuce连接池源码解析: [lettuce连接池很香，撸撸它的源代码](https://blog.csdn.net/zjj2006/article/details/106876235/)