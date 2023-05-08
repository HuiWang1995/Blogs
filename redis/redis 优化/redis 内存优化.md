## info memory

## memory status

MEMORY STATS 命令返回一个关于服务器内存使用情况的数组回复。

The information about memory usage is provided as metrics and their respective values. The following metrics are reported:
有关内存使用情况的信息以指标及其各自的值的形式提供。报告了以下指标：

- peak.allocated: Peak memory consumed by Redis in bytes (see INFO's used_memory_peak)
- peak.allocated ：Redis 消耗的峰值内存（以字节为单位）（参见 INFO 的 used_memory_peak ）
- total.allocated: Total number of bytes allocated by Redis using its allocator (see INFO's used_memory)
- total.allocated ：Redis 使用其分配器分配的总字节数（请参阅 INFO 的 used_memory ）
- startup.allocated: Initial amount of memory consumed by Redis at startup in bytes (see INFO's used_memory_startup)
- startup.allocated ：Redis 在启动时消耗的初始内存量（以字节为单位）（请参阅 INFO 的 used_memory_startup ）
- replication.backlog: Size in bytes of the replication backlog (see INFO's repl_backlog_active)
- replication.backlog ：复制积压的大小（以字节为单位）（参见 INFO 的 repl_backlog_active ）
- clients.slaves: The total size in bytes of all replicas overheads (output and query buffers, connection contexts)
- clients.slaves ：所有副本开销（输出和查询缓冲区、连接上下文）的总大小（以字节为单位）
- clients.normal: The total size in bytes of all clients overheads (output and query buffers, connection contexts)
- clients.normal ：所有客户端开销（输出和查询缓冲区、连接上下文）的总大小（以字节为单位）
- cluster.links: Memory usage by cluster links (Added in Redis 7.0, see INFO's mem_cluster_links).
- cluster.links ：集群链接的内存使用情况（在 Redis 7.0 中添加，参见 INFO 的 mem_cluster_links ）。
- aof.buffer: The summed size in bytes of AOF related buffers.
- aof.buffer ：AOF 相关缓冲区的总大小（以字节为单位）。
- lua.caches: the summed size in bytes of the overheads of the Lua scripts' caches
- lua.caches : Lua 脚本缓存开销的总大小（以字节为单位）
- dbXXX: For each of the server's databases, the overheads of the main and expiry dictionaries (overhead.hashtable.main and overhead.hashtable.expires, respectively) are reported in bytes
- dbXXX ：对于服务器的每个数据库，主字典和过期字典（分别为 overhead.hashtable.main 和 overhead.hashtable.expires ）的开销以字节为单位报告
- overhead.total: The sum of all overheads, i.e. startup.allocated, replication.backlog, clients.slaves, clients.normal, aof.buffer and those of the internal data structures that are used in managing the Redis keyspace (see INFO's used_memory_overhead)
- overhead.total ：所有开销的总和，即 startup.allocated 、 replication.backlog 、 clients.slaves 、 clients.normal 、 aof.buffer 和用于管理 Redis 键空间的内部数据结构的总和（参见 INFO 的 used_memory_overhead ）
- keys.count: The total number of keys stored across all databases in the server
- keys.count ：服务器中所有数据库中存储的密钥总数
- keys.bytes-per-key: The ratio between net memory usage (total.allocated minus startup.allocated) and keys.count
- keys.bytes-per-key ：净内存使用量（ total.allocated 减去 startup.allocated ）与 keys.count 之间的比率
- dataset.bytes: The size in bytes of the dataset, i.e. overhead.total subtracted from total.allocated (see INFO's used_memory_dataset)
- dataset.bytes ：数据集的大小（以字节为单位），即从 total.allocated 中减去 overhead.total （参见 INFO 的 used_memory_dataset ）
- dataset.percentage: The percentage of dataset.bytes out of the net memory usage
- dataset.percentage : dataset.bytes 占净内存使用量的百分比
- peak.percentage: The percentage of peak.allocated out of total.allocated
- peak.percentage ： peak.allocated 占 total.allocated 的百分比
- fragmentation: See INFO's mem_fragmentation_ratio
- fragmentation ：见 INFO 的 mem_fragmentation_ratio