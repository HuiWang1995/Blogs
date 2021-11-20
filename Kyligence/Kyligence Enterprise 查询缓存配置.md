# Kyligence Enterprise 查询缓存配置

SQL级缓存，SQL增加空白符号都会不命中。

为了提升执行相同查询的效率，Kyligence Enterprise 系统自带查询缓存并默认开启。

本文部分摘自Kyligence官方文档。

## 查询缓存的选型

- Ehcache

  java内存级缓存，在Kyligence实例默认为8G JVM时， 默认的缓存大小为1G。

  增大缓存方式为先增大JVM内存，再配置增加缓存大小。

  缺点：1. 在SQL命中率并不高的场景下，过小的缓存作用不大；2. 过大的缓存可能加快GC频率，在FGC下也会导致延迟增加。

- Redis

  缓存存储在Redis中，需要一套Redis集群。在v4.3中引入。

  增大缓存的方式为提升Redis集群的大小。

  缺点： 1. 增加运维难度;

  优点： 1. 提升SQL级缓存实用性。

## 缓存大小的预估方式

在kylin元数据库中根据查询历史进行SQL统计，例如统计24小时内查询延迟大于2000毫秒或扫描行数大于10000或查询扫描数据量字节数大于1048676,且查询结果集小于10000单元格的SQL历史，其一共需要的结果集的大小，以此来预估预存T+1日慢SQL需要的缓存大小。（kylin 查询历史的页面无法如此精细）

## 参数记录

### $KYLIN_HOME/conf/kylin.properties

Kyligence Enterprise 自带查询缓存功能并默认开启

**注意**：所有下述配置需要重启后方能生效。

| 配置项                    | 描述                                             | 默认值 | 可选值 |
| :------------------------ | :----------------------------------------------- | :----- | :----- |
| kylin.query.cache-enabled | 是否开启查询缓存，当该参数开启，下述参数才生效。 | true   | false  |

### 查询被缓存的条件

由于内存资源可能是有限的，Kyligence Enterprise 不会默认缓存每条查询的结果。目前产品会有选择性的对性能较慢且结果集不是特别大的查询进行缓存。一条查询是否会被缓存，由以下几个参数影响：满足序号为 1、2、3 配置项中任意一项，且同时满足序号 4 配置项时，查询会被缓存。

| 序号 | 配置项                                 | 描述                     | 默认值  | 默认值单位 |
| :--- | :------------------------------------- | :----------------------- | :------ | :--------- |
| 1    | kylin.query.cache-threshold-duration   | 查询延迟大于该值         | 2000    | 毫秒       |
| 2    | kylin.query.cache-threshold-scan-count | 查询扫描的行数大于该值   | 10240   | 行         |
| 3    | kylin.query.cache-threshold-scan-bytes | 查询扫描的数据量大于该值 | 1048576 | 字节       |
| 4    | kylin.query.large-query-threshold      | 查询结果集小于该值       | 1000000 | 单元格     |

### Ehcache 缓存配置

Kyligence Enterprise 默认使用 Ehcache 作为查询缓存，您可以通过配置 Ehcache 来控制查询缓存的大小和策略。您可以通过修改以下配置项来替换默认的查询缓存配置文件，更多的 Ehcache 配置项请参考官网 [ehcache文档](https://www.ehcache.org/generated/2.9.0/html/ehc-all/#page/Ehcache_Documentation_Set%2Fehcache_all.1.017.html%23)。

| 配置项             | 描述                                                         | 默认值                |
| :----------------- | :----------------------------------------------------------- | :-------------------- |
| kylin.cache.config | ehcache.xml 文件的路径。您可以在 `${KYLIN_HOME}/conf/` 下新建格式为 `xml` 的文件来替换默认的查询缓存配置文件，如 `ehcache2.xml`，并且修改配置项的值为： `file://${KYLIN_HOME}/conf/ehcache2.xml`。 | classpath:ehcache.xml |

### Redis 缓存配置 （v4.2不支持，v4.3引入）

由于 Ehcache 查询缓存是进程级的，在不同进程或不同节点之间并不共享，因此在集群部署模式下，当后续相同查询路由至不同 Kyligence Enterprise 节点时，一个进程查询执行结果的缓存无法被另一个进程使用。因此，我们支持使用 Redis 集群作为分布式查询缓存，在所有 Kyligence Enterprise 节点间共享。具体的参数和配置方法如下：

| 配置项                             | 描述                                                         | 默认值         | 可选值 |
| :--------------------------------- | :----------------------------------------------------------- | :------------- | :----- |
| kylin.cache.redis.enabled          | 是否开启 Redis 集群用于查询缓存                              | false          | true   |
| kylin.cache.redis.cluster-enabled  | 是否开启 Redis 集群模式                                      | false          | true   |
| kylin.cache.redis.hosts            | Redis 主机地址，当您需要连接 Redis 集群时，请使用逗号进行分割。如 kylin.cache.redis.hosts=localhost:6379,localhost:6380 | localhost:6379 |        |
| kylin.cache.redis.expire-time-unit | Redis 缓存保留单位，EX为秒，PX为毫秒                         | EX             | PX     |
| kylin.cache.redis.expire-time      | Redis 缓存保留时间                                           | 86400          |        |
| kylin.cache.redis.password         | Redis密码                                                    |                |        |

#### 已知限制

由于当前 Query 节点与 All/Job 节点存在元数据存在同步差异，redis 缓存开关 `kylin.cache.redis.enabled=true` 需要和 `kylin.server.store-type=jdbc` 一起配置。

> **注意**：Redis密码可以明文配置，也可以加密。加密方法请参考： [使用 MySQL 作为元数据存储](https://docs.kyligence.io/books/v4.3/zh-cn/installation/rdbms_metastore/mysql_metastore.cn.html)