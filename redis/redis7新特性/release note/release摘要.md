Introduction to the Redis 7.0 release
=====================================

Redis 7.0 includes several new user-facing features, significant performance
optimizations, and many other improvements. It also includes changes that
potentially break backwards compatibility with older versions. We urge users to
review the release notes carefully before upgrading.

In particular, users should be aware of the following changes:

1. Redis 7 stores AOF as multiple files in a folder; see Multi-Part AOF below.
2. Redis 7 uses a new version 10 format for RDB files, which is incompatible
   with older versions.
3. Redis 7 converts ziplist encoded keys to listpacks on the fly when loading
   an older RDB format. Conversion applies to loading a file from disk or
   replicating from a Redis master and will slightly increase loading time.
4. See sections about breaking changes mentioned below.

Here is a comprehensive list of changes in this release compared to 6.2.6.
Each one includes the PR number that added it so that you can get more details
at https://github.com/redis/redis/pull/<number>

New Features
============

* Redis Functions: A new way to extend Redis with server-side scripts (#8693)
  see https://redis.io/topics/functions-intro
* ACL: Fine-grained key-based permissions and allow users to support multiple
  sets of command rules with selectors (#9974)
  see https://redis.io/topics/acl#key-permissions and https://redis.io/topics/acl#selectors.
* Cluster: Sharded (node-specific) Pub/Sub support (#8621)
  see https://redis.io/topics/pubsub#sharded-pubsub
* First-class handling of sub-commands in most contexts (affecting ACL
  categories, INFO commandstats, etc.) (#9504, #10147)
* Command metadata and documentation (#10104)
  see https://redis.io/commands/command-docs, https://redis.io/topics/command-tips
* Command key-specs. A better way for clients to locate key arguments and their
  read/write purpose (#8324, #10122, #10167)
  see https://redis.io/topics/key-specs
* Multi-Part AOF mechanism to avoid AOF rewrite overheads (#9788)
* Cluster: Support for hostnames, instead of IP addresses only (#9530)
* Improved management of memory consumed by network buffers, and an option to
  drop clients when total memory exceeds a limit  (#8687)
* Cluster: A mechanism for disconnecting cluster bus connections to prevent
  uncontrolled buffer growth (#9774)
* AOF: Timestamp annotations and support for point-in-time recovery (#9326)
* Lua: support Function flags in EVAL scripts (#10126)
  see https://redis.io/topics/eval-intro#eval-flags
* Lua: Support RESP3 reply for Verbatim and Big-Number types (#9202)
* Lua: Get Redis version via redis.REDIS_VERSION, redis.REDIS_VERSION_NUM (#10066)

New user commands or command arguments
--------------------------------------

* ZMPOP, BZMPOP commands (#9484)
* LMPOP, BLMPOP commands (#9373)
* SINTERCARD, ZINTERCARD commands (#8946, #9425)
* SPUBLISH, SSUBSCRIBE, SUNSUBSCRIBE, PUBSUB SHARDCHANNELS/SHARDNUMSUB (#8621)
* EXPIRETIME and PEXPIRETIME commands (#8474)
* EXPIRE command group supports NX/XX/GT/LT options (#2795)
* SET command supports combining NX and GET flags (#8906)
* BITPOS, BITCOUNT accepts BIT index (#9324)
* EVAL_RO, EVALSHA_RO command variants, to run on read-only replicas (#8820)
* SORT_RO command, to run on read-only replicas (#9299)
* SHUTDOWN arguments: NOW, FORCE, ABORT (#9872)
* FUNCTION *, FCALL, FCALL_RO - https://redis.io/commands/function-load
* CONFIG SET/GET can handle multiple configs atomically, in one call (#9748, #9914)
* QUIT promoted to be a proper command, HOST: and POST demoted (#9798)
* XADD supports auto sequence number via <ms>-* (#9217)

New administrative and introspection commands and command arguments
-------------------------------------------------------------------

* COMMAND DOCS (#9656, #10056, #10104)
* COMMAND LIST (#9504)
* COMMAND INFO accepts sub-commands as args, and no args too (#9504, #10056)
* LATENCY HISTOGRAM (#9462)
* CLUSTER LINKS (#9774)
* CLUSTER DELSLOTSRANGE and CLUSTER ADDSLOTSRANGE (#9445)
* CLIENT NO-EVICT (#8687)
* ACL DRYRUN (#9974)
* SLOWLOG GET supports passing in -1 to get all entries (#9018)

Command replies that have been extended
---------------------------------------

* COMMAND and COMMAND INFO extended with tips, key-specs and sub-commands
  see https://redis.io/commands/command
* ACL CAT, COMMAND LIST list sub-commands (#10127)
* MODULE LIST reply includes path and args (#4848)
* OBJECT ENCODING returns listpack instead of ziplist (#8887, #9366)
* CLUSTER SLOTS hostname support (#9530)
* COMMAND command: Added the `blocking` and `module` flags (#10104, #9656)


Potentially Breaking Changes
============================

* Modifying the bind parameter to a non-default value will no longer implicitly
  disable protected-mode (#9034)
* Remove EVAL script verbatim replication, propagation, and deterministic
  execution logic (#9812)
  This has been deprecated and off by default since Redis 6 and is no longer
  supported.
* ACL: pub/sub channels are blocked by default (acl-pubsub-default=resetchannels) (#10181)
* SCRIPT LOAD and SCRIPT FLUSH are no longer propagated to replicas / AOF (#9812)
* ACL: Declarations of duplicate ACL users in startup files and command line
  arguments will result in an error, whereas previously the last declaration
  would overwrite the others. (#9330)
* Replication: TTLs are always replicated as absolute (not relative) millisecond
  timestamps (#8474)
* Fixes in handling multi-key commands with expired keys on writable replicas (#9572)
* CONFIG SET maxmemory returns before starting eviction (#10019)
* AOF: The new Multi-Part mechanism stores data as a set of multiple files in a
  designated folder (#9788)
* Remove STRALGO command, preserve LCS a standalone command which only works on
  keys (#9799)
* Remove gopher protocol support (#9057)
* MODULE and DEBUG commands disabled (protected) by default, for better security (#9920)
* Snapshot-creating and other admin commands in MULTI/EXEC transactions are now
  rejected (#10015)
* PING is now rejected with -MASTERDOWN when replica-serve-stale-data=no (#9757)
* ACL GETUSER reply now uses ACL syntax for `keys` and `channels` (#9974)
* COMMAND reply drops `random` and `sort-for-scripts` flags, which are now part
  of command tips (#10104)
* LPOP/RPOP with count against non-existing list return null array (#10095)
* INFO commandstats now shows the stats per sub-command (#9504)
* ZPOPMIN/ZPOPMAX used to produce wrong replies when count is 0 with non-zset (#9711)
* LPOP/RPOP used to produce wrong replies when count is 0 (#9692)
* CONFIG GET bind now returns the current value in effect, even if the implicit
  default is in use (#9034)
* CONFIG REWRITE now rewrites the list of modules to load (#4848)
* Config: repl-diskless-sync is now set to yes by default (#10092)
* When shutting down, Redis can optionally wait for replicas to catch up on the
  replication link (#9872)
* Most CONFIG SET, REWRITE, RESETSTAT commands are now allowed during loading (#9878)
* READONLY and READWRITE commands are now allowed when loading and on stale
  replicas (#7425)
* Fix ACL category for SELECT, WAIT, ROLE, LASTSAVE, READONLY, READWRITE, ASKING (#9208)
* RESET is now allowed even when on unauthenticated connections (#9798)
* SCRIPT LOAD is now allowed on stale replicas (#10126)


Security improvements
=====================

* Sensitive configs and commands blocked (protected) by default (#9920)
* Improve bind and protected-mode config handling (#9034)
* Sentinel: avoid logging auth-pass value (#9652)
* redis-cli: sensitive commands bypass the history file (#8895)


Performance and resource utilization improvements
=================================================

* Significant memory saving and latency improvements in cluster mode (#9356)
* Significant memory savings in case of many hash or zset keys (#9228)
* Replication backlog and replicas use one global shared replication buffer (#9166)
* Significant reduction of copy-on-write memory overheads (#8974)
* Free unused capacity in the cluster send buffer (#9255)
* Memory efficiency, make full use of client struct memory for reply buffers (#8968)
* Replace ziplist with listpack in Hash, List, Zset (#8887, #9366, #9740)
* Add support for list type to store elements larger than 4GB (#9357)
* Reuse temporary client objects for blocked clients by module (#9940)
* Remove command argument count limit, dynamically grow argv buffer (#9528)
* Optimize list type operations to seek from the nearest end (#9454)
* Improvements in fsync to avoid large writes to disk (#9409)
* BITSET and BITFIELD SET only propagated when the value actually changed (#9403)
* Improve latency when a client is unblocked by module timer (#9593)


Other General Improvements
==========================

* Make partial sync possible after master reboot (#8015)
* Always create a base AOF file when redis starts from empty (#10102)
* Replica keep serving data during repl-diskless-load=swapdb for better
  availability (#9323)


Changes in CLI tools
====================
* redis-cli --json, and -2 options (#9954)
* redis-cli --scan, add sleep interval option (#3751)
* redis-cli --replica optimization, skip RDB generation (#10044)
* redis-cli --functions-rdb, generate RDB with Functions only (#9968)
* redis-cli -X, take an arbitrary arg from stdin, extend --cluster call take -x (#9980)
* redis-benchmark -x takes an argument from stdin (#9130)
* redis-benchmark, Added URI support (#9314)
* redis-cli monitor and pubsub can be aborted with Ctrl+C, keeping the cli alive (#9347)


Platform / toolchain support related improvements
=================================================

* Upgrade jemalloc 5.2.1 (#9623)
* Fix RSS metrics on NetBSD and OpenBSD (#10116, #10149)
* Check somaxconn system settings on macOS, FreeBSD and OpenBSD (#9972)
* Better fsync on MacOS, improve power failure safety (#9545)


New configuration options
=========================

* CONFIG SET/GET can handle multiple configs in one call (#9748, #9914)
* Support glob pattern matching for config include files (#8980)
* appenddirname, folder where multi-part AOF files are stored (#9788)
* shutdown-timeout, default 10 seconds (#9872)
* maxmemory-clients, allows limiting the total memory usage by all clients (#8687)
* cluster-port, can control the bind port of cluster bus (#9389)
* bind-source-addr, configuration argument control IP of outgoing connections (#9142)
* busy-reply-threshold, alias for the old lua-time-limit (#9963)
* repl-diskless-sync-max-replicas, allows faster replication in some cases (#10092)
* latency-tracking, enabled by default, and latency-tracking-info-percentiles (#9462)
* cluster-announce-hostnameand cluster-preferred-endpoint-type (#9530)
* cluster-allow-pubsubshard-when-down (#8621)
* cluster-link-sendbuf-limit (#9774)
* list-max-listpack-*, hash-max-listpack-*, zset-max-listpack-* as aliases for
  the old ziplist configs (#8887, #9366, #9740)


INFO fields and introspection changes
=====================================

* INFO: latencystats section (#9462)
* INFO: total_active_defrag_time and current_active_defrag_time (#9377)
* INFO: total_eviction_exceeded_time and current_eviction_exceeded_time (#9031)
* INFO: evicted_clients (#8687)
* INFO: mem_cluster_links, total_cluster_links_buffer_limit_exceeded (#9774)
* INFO: current_cow_peak (#8974)
* INFO: Remove aof_rewrite_buffer_length (#9788)
* MEMORY STATS: Report slot to keys map size in in cluster mode (#10017)
* INFO MEMORY: changes to separate memory usage of Functions and EVAL (#9780)
* INFO MEMORY: Add mem_total_replication_buffers, change meaning of
  mem_clients_slaves (#9166)
* CLIENT LIST: tot-mem, multi-mem (#8687)
* CLIENT LIST, INFO: Show RESP version (#9508)
* SENTINEL INFO: tilt_mode_since (#9000)
* LATENCY: Track module-acquire-GIL latency (#9608)


Module API changes
==================

* Add API for replying with RESP3 types (#8521, #9639, #9632)
* Add API for parsing RESP3 replies from RM_Call (#9202)
* Add RM_Call '0' and '3' flags to control RESP version to be used (#9202)
* Add Support for validating ACL explicitly (#9309, #9974)
* Add missing list type functionality APIs (#8439)
* Add API for yielding to Redis events during long busy jobs (#9963)
* Add API for registering other file descriptors to the Redis event loop (#10001)
* Enhance mem_usage/free_effort/unlink/copy and IO callbacks to have key name
  and DB index (#8999)
* Enhance mem_usage callback to get the requested sample size (#9612)
* RM_GetContextFlags: CTX_FLAGS_ASYNC_LOADING, CTX_FLAGS_RESP3 (#9323, #9202)
* Mark APIs as non-experimental (#9983)
* RM_CreateSubcommand (#9504)
* RM_KeyExists (#9600)
* RM_TrimStringAllocation (#9540)
* RM_LoadDataTypeFromStringEncver (#9537)
* RM_MonotonicMicroseconds (#10101)
* Add ReplAsyncLoad event and deprecate the ReplBackup event (#9323)
* Add RM_SetModuleOptions OPTIONS_HANDLE_REPL_ASYNC_LOAD flag (#9323)


Bug Fixes
=========

* Fix COMMAND GETKEYS on EVAL without keys (#9733)
* Improve MEMORY USAGE with allocator overheads (#9095)
* Unpause clients after manual failover ends instead of waiting for timed (#9676)
* Lua: fix crash on a script call with many arguments, a regression in v6.2.6 (#9809)
* Lua: Use all characters to calculate string hash to prevent hash collisions (#9449)
* Prevent LCS from allocating temp memory over proto-max-bulk-len (#9817)
* Tracking: Make invalidation messages always after command's reply (#9422)
* Cluster: Hide empty replicas from CLUSTER SLOTS responses (#9287)
* CLIENT KILL killed all clients when used with ID of 0 (#9853)
* Fix bugs around lists with list-compress-depth (#9849, #9779)
* Fix one in a blue moon LRU bug in RESTORE, RDB loading, and module API (#9279)
* Reset lazyfreed_objects info field with RESETSTAT, test for stream lazyfree (#8934)
* Fix RDB and list node compression for handling values larger than 4GB (#9776)
* Fix a crash when adding elements larger than 2GB to a Set or Hash (#9916)
* Diskless replication could not count as a change and skip next database SAVE (#9323)
* Fix excessive stream trimming due to an overflow (#10068)
* Safe and organized exit when receiving SIGTERM while loading (#10003)
* Improve EXPIRE TTL overflow detection (#9839)
* Add missed error counting for INFO errorstats (#9646)
* DECRBY LLONG_MIN caused negation overflow (#9577)
* Delay discarding cached master when full synchronization (#9398)
* Fix Stream keyspace notification and persistence triggers in consumer
  creation and deletion (#9263)
* Fix rank overflow in zset with more than 2B entries (#9249)
* Avoid starting in check-aof / check-rdb / sentinel modes if only the folder
  name contains that name (#9215, #9176)
* create the log file only after done parsing the entire config file (#6741)
* redis-cli: Fix SCAN sleep interval for --bigkeys, --memkeys, --hotkeys (#9624)
* redis-cli: Fix prompt to show the right DB num and transaction state after
  RESET (#9096)
* Module API: fix possible propagation bugs in case a module calls CONFIG SET
  maxmemory outside a command (#10019, #9890)
* Module API: carry through client RESP version to module blocked clients (#9634)
* Module API: release clients blocked on module commands in cluster resharding
  and down state (#9483)
* Sentinel: Fix availability after master reboot (#9438)
* Sentinel: Fix memory leak with TLS (#9753)
* Sentinel: Fix possible failover due to duplicate zero-port (#9240)
* Sentinel: Fix issues with hostname support (#10146)
* Sentinel: Fix election failures on certain container environments (#10197)



--- 以下是机器翻译 ---


Redis 7.0版本介绍
=====================================

Redis 7.0 包括几个面向用户的新功能、显着的性能
优化和许多其他改进。 它还包括以下更改
可能破坏与旧版本的向后兼容性。 我们敦促用户
升级前仔细阅读发行说明。

用户尤其应注意以下变化：

1、Redis 7将AOF存储为一个文件夹中的多个文件； 请参阅下面的多部分 AOF。
2. Redis 7使用新的version 10格式的RDB文件，不兼容
   与旧版本。
3. Redis 7 在加载时将 ziplist 编码的键动态转换为列表包
   较旧的 RDB 格式。 转换适用于从磁盘加载文件或
   从 Redis 主服务器复制，会稍微增加加载时间。
4. 请参阅下面提到的有关重大更改的部分。

以下是此版本与 6.2.6 相比的完整更改列表。
每一个都包含添加它的 PR 编号，以便您可以获得更多详细信息
在 https://github.com/redis/redis/pull/<number>

新功能
============

* Redis 函数：一种使用服务器端脚本扩展 Redis 的新方法 (#8693)
  请参阅 https://redis.io/topics/functions-intro
* ACL：基于密钥的细粒度权限，允许用户支持多个
  带选择器的命令规则集 (#9974)
  请参阅 https://redis.io/topics/acl#key-permissions 和 https://redis.io/topics/acl#selectors。
  *集群：分片（节点特定）发布/订阅支持（#8621）
  请参阅 https://redis.io/topics/pubsub#sharded-pubsub
* 在大多数情况下对子命令的一流处理（影响 ACL
  类别、信息命令统计等）（#9504、#10147）
* 命令元数据和文档 (#10104)
  请参阅 https://redis.io/commands/command-docs，https://redis.io/topics/command-tips
* 命令键规范。 一种更好的方式让客户找到关键论点和他们的意见
  读/写用途（#8324、#10122、#10167）
  请参阅 https://redis.io/topics/key-specs
* 避免 AOF 重写开销的多部分 AOF 机制 (#9788)
* 集群：支持主机名，而不仅仅是 IP 地址 (#9530)
  *改进了对网络缓冲区消耗的内存的管理，以及一个选项
  当总内存超过限制时删除客户端 (#8687)
* Cluster：一种断开集群总线连接的机制，以防止
  不受控制的缓冲区增长 (#9774)
* AOF：时间戳注释和对时间点恢复的支持（#9326）
* Lua：支持 EVAL 脚本中的函数标志 (#10126)
  请参阅 https://redis.io/topics/eval-intro#eval-flags
* Lua: 支持 Verbatim 和 Big-Number 类型的 RESP3 回复 (#9202)
* Lua: 通过 redis.REDIS_VERSION, redis.REDIS_VERSION_NUM 获取 Redis 版本 (#10066)

新的用户命令或命令参数
--------------------------------------

* ZMPOP、BZMPOP 命令 (#9484)
* LMPOP, BLMPOP 命令 (#9373)
* SINTERCARD、ZINTERCARD 命令（#8946、#9425）
* 发布、SSUBSCRIBE、SUNSUBSCRIBE、PUBSUB SHARDCHANNELS/SHARDNUMSUB (#8621)
* EXPIRETIME 和 PEXPIRETIME 命令（#8474）
* EXPIRE 命令组支持 NX/XX/GT/LT 选项 (#2795)
* SET 命令支持组合 NX 和 GET 标志 (#8906)
* BITPOS, BITCOUNT 接受 BIT 索引 (#9324)
* EVAL_RO、EVALSHA_RO 命令变体，在只读副本上运行 (#8820)
* SORT_RO 命令，在只读副本上运行（#9299）
  *关机参数：现在，强制，中止（#9872）
* 功能 *、FCALL、FCALL_RO - https://redis.io/commands/function-load
* CONFIG SET/GET 可以在一次调用中自动处理多个配置（#9748、#9914）
* QUIT 提升为正确的命令，HOST: 和 POST 降级 (#9798)
* XADD 通过 <ms>-* 支持自动序列号 (#9217)

新的管理和内省命令以及命令参数
---------------------------------------------- ------------------

* 命令文档（#9656、#10056、#10104）
* 命令列表 (#9504)
* COMMAND INFO 接受子命令作为参数，也没有参数（#9504，#10056）
* 延迟直方图 (#9462)
* 集群链接 (#9774)
  *集群DELSLOTSRANGE和集群ADDSLOTSRANGE（#9445）
* 客户端不退出 (#8687)
* ACL 空运行 (#9974)
* SLOWLOG GET 支持传入 -1 获取所有条目 (#9018)

已扩展的命令回复
--------------------------------------

* COMMAND 和 COMMAND INFO 扩展了提示、关键规范和子命令
  请参阅 https://redis.io/commands/command
* ACL CAT, COMMAND LIST 列出子命令 (#10127)
* MODULE LIST 回复包括路径和参数 (#4848)
* OBJECT ENCODING 返回 listpack 而不是 ziplist (#8887, #9366)
* CLUSTER SLOTS 主机名支持 (#9530)
* COMMAND 命令：添加了`blocking` 和`module` 标志（#10104，#9656）


潜在的重大变化
============================

* 将绑定参数修改为非默认值将不再隐式
  禁用保护模式 (#9034)
* 删除 EVAL 脚本逐字复制、传播和确定性
  执行逻辑 (#9812)
  自 Redis 6 以来，这已被弃用并默认关闭，不再是
  支持