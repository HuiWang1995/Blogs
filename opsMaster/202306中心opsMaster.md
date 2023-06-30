# 202306中心级opsMaster运维比赛总结

<img src="https://img2.baidu.com/it/u=1535717794,1585386024&fm=253&fmt=auto&app=138&f=JPEG?w=500&h=544">

## 考察能力
一共出了7道题，大致考察了java反编译、oracle分区索引、redis、nginx、网络命令、日志分析、内存、磁盘、抓包分析
我用还记得住的点记录一下。

## 内存超出
> 本题考查了 JVM 参数，需要对JVM 有一定的了解。

java 进程A 访问java 进程B，若正常页面会返回答案，异常则返回内存空间不足。

进入服务器发现 进程B 占用内存3g， 立即重启，重启报错，空间不足。显然它还是占用了过大的空间。

JVM启动参数添加 -Xmx512m 。限制内存使用512m, 启动解决问题。
```shell
java -Xmx512m processB.jar
```


## oracle分区索引失效
> 本题考查了oracle分区索引失效的解决方案。

访问页面，报错 ORA-XXXX，某分区索引失效。

反编译，获取数据库连接串，通过plsql 连接， 重建该分析索引，或者直接删除分区索引均可。

可以先看看失效的索引有哪些：
```sql
SELECT INDEX_OWNER, INDEX_NAME, PARTITION_NAME, STATUS FROM DBA_IND_PARTITIONS WHERE STATUS='UNUSABLE';
```

因为我们知道了索引名、表名，也可以直接查ALL_INDEXES
```sql
SELECT * FROM ALL_INDEXES WHERE owner = '用户名' AND TABLE_NAME = '表名';
```
下一步删除索引即可，当然，如果该索引在主键约束中，则还需加一步。
```sql
-- 去除主键索引
ALTER TABLE 表名 DROP CONSTRAINT 主键约束名;

-- 删除索引
DROP INDEX 索引名;
```
解决问题后，则获得答案。

> ORACLE 分区索引为啥会失效呢？ 一般都是DDL操作导致的，但是DML也是有可能的。


## REDIS GET 命令报错
> 本题考查了redis 运维相关的知识，如何禁用一个命令。

这次在页面上显示报错 redis get 命令错误。
而get命令是redis基础命令，不应有错。通过反编译jar包，在linux上通过redis-cli 执行get 发现的确报错。
通过 ps -ef | grep redis 查看到redis进程，但貌似没指定配置文件。
直接kill，再去/usr/local/redis/bin重启redis-server，发现的确没报错了，但是也没数据。

显然，这步操作十分的冒失了，相当于重启之后数据文件位置指定不对，它其实是有配置文件启动的。
通过find / -name redis.conf 查找到了3个配置文件，通过路径上的6379猜测应该是它，则用它重启redis，重现了该问题。

然后在配置文件中，发现了`rename get ''` 将get 命令禁用了，将其注释并重启则解决问题。

其实，还有一种方法，进到服务器时，不将redis进程kill掉，用lsof找到它的数据路径，直接用新的配置文件，将数据路径指向其去启动即可。

> 生产上，像keys、 flushdb、 flushall、 config 等命令都是被运维要求禁用的，都是通过rename来实现。当然，redis 7 之后的acl控制可以有其他方法。

## nginx 403 forbidden
> 考察nginx 配置文件

当页面访问403后，首先想到的是将项目文件的权限改正，让nginx有权限读。

后发现仍403，打开配置文件未发现异常，但是它有include conf.d文件夹中的配置文件，在其中找到了 deny all 配置，删除其即可完成访问。

## 网络命令、磁盘
> 本题综合考察

提干说X端口的python服务用于接收另一台机器的请求，备份日志。但是不知其IP，但是给出了服务器账号和密码。
通过 netstat -ano|grep 端口号， 则可以发现和本机建立连接的IP。 登录后发现其启动的python脚本中提示是磁盘空间不足。

通过df -h 查看到该日志目录挂载的磁盘的确空间不足，通过 `rm -rf *` 完成删除即可。 

## 抓包分析
> wireshark的应用，网络知识

题目说某位老师（xxxxx）忘记了内管账号的密码，但是有正常登录时的网络包，找出他的密码。
打开题目给定的网络包，直接先过滤一手 http 协议。

在备注中可以看到有些POST 访问xxx/inner/xxx的报文，打开一看的确是请求登录的报文。

通过TCP流追踪可以发现，登录报错都是400返回，找到数个成功的是200返回码，当然，还需要注意该老师的工号要对上。

## 日志分析
> 考察awk,sort等一系列命令的运用。

在给定的logs目录下有 system.out.log.1 - system.out.log.100 这100个文件，让我们找出访问接口次数最多的接口，它在哪个文件中被访问的最多。

所以其实一共有两步，先找到该接口，再找到该接口在哪个文件中出现次数最多。

```shell
# $3 是每行分割后接口所在的列
awk '{print $3}' logs/system.out.log.* | sort | uniq -c | sort -rn | head -n 10

1111 /api/login
231  /api/test
1    /actuator/info
...(demo)
```

接下来，我们再找出它在哪个文件中个数最多
```shell
cd logs

# $1 是grep 输出的第一列，文件名
grep "/api/login" system.out.log.* | awk -F ':' '{print $1}' | sort | uniq -c | sort -nr | head -n 10

653  system.out.log.21
251  system.out.log.2
37   system.out.log.56
...(demo)
```
那么最多的就是system.out.log.21。

> 对于不长时间运维的的确挺难的，我们知道命令也没在时限内做出来。