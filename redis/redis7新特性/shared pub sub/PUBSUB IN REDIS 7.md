# Sharded Pub/Sub 共享订阅

> From Redis 7.0, sharded Pub/Sub is introduced in which shard channels are assigned to slots by the same algorithm used to assign keys to slots. A shard message must be sent to a node that owns the slot the shard channel is hashed to. The cluster makes sure the published shard messages are forwarded to all nodes in the shard, so clients can subscribe to a shard channel by connecting to either the master responsible for the slot, or to any of its replicas. SSUBSCRIBE, SUNSUBSCRIBE and SPUBLISH are used to implement sharded Pub/Sub.


从 Redis 7.0 开始，引入了分片 Pub/Sub，其中分片通道通过用于将键分配给槽的相同算法分配给槽。分片消息必须发送到拥有分片通道散列到的槽的节点。集群确保已发布的分片消息被转发到分片中的所有节点，因此客户端可以通过连接到负责插槽的主节点或其任何副本来订阅分片通道。 SSUBSCRIBE 、 SUNSUBSCRIBE 和 SPUBLISH 用于实现分片 Pub/Sub。


> Sharded Pub/Sub helps to scale the usage of Pub/Sub in cluster mode. It restricts the propagation of messages to be within the shard of a cluster. Hence, the amount of data passing through the cluster bus is limited in comparison to global Pub/Sub where each message propagates to each node in the cluster. This allows users to horizontally scale the Pub/Sub usage by adding more shards.


Sharded Pub/Sub 有助于在集群模式下扩展 Pub/Sub 的使用。它将消息的传播限制在集群的分片内。因此，与每条消息传播到集群中每个节点的全局发布/订阅相比，通过集群总线的数据量是有限的。这允许用户通过添加更多分片来水平扩展 Pub/Sub 使用。