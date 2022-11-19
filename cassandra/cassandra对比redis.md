# Cassandra对比Redis暨分析中心SQL级缓存选型分析



## 速度

​		Cassandra更重视稳定性，提供类SQL（hive sql 语法）查询，可以存储大量数据。但是它查询比Redis慢。

​		Redis比Cassandra快得多，但如果将其用于大型数据集，速度会变慢，非常适合快速变化的小数据集。大value会导致后续的请求一定的堵塞。

## 系统架构

​		Cassandra选型为CP。为磁盘绑定型的内存数据库（disk-bound in-memory）。Cassandra类似于大表，它为数据列表保存列或列族。

​		Redis选型为AP。为磁盘备份型的内存数据库（disk backed in-memory）。Redis没有列概念，以键值对的形式存储数据。

## 分析中心缓存选型分析

​		当前分析中心使用Redis进行SQL级缓存，而Redis是纯内存数据库，且会随着使用内存变大，性能会有所降低，故当前仅将管理岗查询机构级的数据进行缓存，而无理财经理级，且可能由于量大导致内存置换，需要重新查询Kylin。

​		Cassandra的数据会存在磁盘中，且拥有横向扩展能力，能够承接大量数据，可以作为分析中心全SQL级缓存的KV数据库。对于同一天的SQL查询结果，被内存替换出后，其可以在自己的磁盘中查询出，而非Kylin。

​		综合考虑，建议管理岗的查询结果继续使用Redis，理财经理的查询使用Cassandra进行缓存。



## 引用

[Cassandra Vs. Redis: 7 Key Differentiating Factors](https://www.knowledgenile.com/blogs/cassandra-vs-redis/)

[Cassandra代替Redis?](https://timyang.net/data/cassandra-vs-redis/)

[Cassandra vs Redis](https://www.educba.com/cassandra-vs-redis/)

