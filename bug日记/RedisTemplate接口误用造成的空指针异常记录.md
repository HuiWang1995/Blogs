# RedisTemplate 接口误用造成的空指针异常记录

> redis读写在现阶段，除了原生的调用接口，例如jedis、lettuce等,许多都使用了redisTemplate，当然，更多的使用了@Cacheable、@CaachePut之类的注解。
>
> redisTemplate的封装避免了底层api的不同。而注解@Cacheable等则更多的符合了旁路设计，避免了更多人为try、catch,代码更加优雅、不容易出错。

## BUG级别

低级

## BUG描述

典型的空指针异常。据产生BUG的童鞋在一定的排查后描述：向redis进行查询数据，redis返回给他一个数组，但是数组里面的那个对象为null，导致了他空指针、redis难道查不到不应该直接返回给他一个null对象吗？

## BUG相关代码

```java
        List<String> values = null;
        values = redisTemplate.opsForHash().multiGet(HASH, keys);

// 以下是multiGet源码定义
List<HV> multiGet(H var1, Collection<HK> var2);
```

## 分析

这个接口的作用是，传入一个HASH类型的KEY，以及需要去这个HASH对象中查询VALUE的KEY列表。

**那么当查多个KEY的时候，如何知道自己对应的结果是什么呢？自然是和KEY下标对应的结果列表下标位置的值了。那么自然，其中存在可能查询不到结果的情况，那下标肯定是不能乱的，那就在对应位置放一个null，不是合情合理吗？**

那使用这个借口的同学就是因为并不了解这个接口，没有查阅过文档，随意使用。所以没有进行非空判断，最终导致了空指针。（*PS：不过该童鞋作为该团队的核心开发人员，犯这样的低级错误实在不应该*）。

## 接口文档

1. RedisTemplate

https://docs.spring.io/spring-data/redis/docs/current/api/org/springframework/data/redis/core/RedisTemplate.html

2. HashOperations

https://docs.spring.io/spring-data/redis/docs/current/api/org/springframework/data/redis/core/HashOperations.html

> ####  multiGet
>
> ```
> List<HV> multiGet(H key,
>                   Collection<HK> hashKeys)
> ```
>
> Get values for given `hashKeys` from hash at `key`.
>
> - **Parameters:**
>
>   `key` - must not be null.
>
>   `hashKeys` - must not be null.
>
> - **Returns:**
>
>   null when used in pipeline / transaction.

根据接口文档描述，该接口传入的key, hashKeys不可为null。 以及在管道中与事务中使用时会返回为null。

倒是没有具体描述返回的内容内部可能为null。

## 源码

既然文档不能给出对应位置查询不到时会放入null的解答，就来查看一下源码。

1. 首先看看opsForHash的返回结果：

```java
// org.springframework.data.redis.core.HashOperations
    public <HK, HV> HashOperations<K, HK, HV> opsForHash() {
        return new DefaultHashOperations(this);
    }
//  org.springframework.data.redis.core.DefaultHashOperations#DefaultHashOperations
	DefaultHashOperations(RedisTemplate<K, ?> template) {
        super(template);
    }
// org.springframework.data.redis.core.AbstractOperations#AbstractOperations
    final RedisTemplate<K, V> template;

    AbstractOperations(RedisTemplate<K, V> template) {
        this.template = template;
    }
```

这个方法就是new了一个DefaultHashOperations对象，将redisTemplate对象传递了进去赋值。

2. 再看看multiGet方法

```java
// org.springframework.data.redis.core.DefaultHashOperations#multiGet
	public List<HV> multiGet(K key, Collection<HK> fields) {
        if (fields.isEmpty()) {
            return Collections.emptyList();
        } else {
            byte[] rawKey = this.rawKey(key);
            byte[][] rawHashKeys = new byte[fields.size()][];
            int counter = 0;

            Object hashKey;
            for(Iterator var6 = fields.iterator(); var6.hasNext(); rawHashKeys[counter++] = this.rawHashKey(hashKey)) {
                hashKey = var6.next();
            }

            List<byte[]> rawValues = (List)this.execute((connection) -> {
                return connection.hMGet(rawKey, rawHashKeys);
            }, true);
            return this.deserializeHashValues(rawValues);
        }
    }
```

这个方法会将key序列化为二进制数组byte[],同样的对`hashKeys`，也就是此处的入参`fields`进行序列化，成为`rawHashKeys`。然后，它声明了`rawValues`对象，用于存放查询redis的结果，并且通过传入回调的方式，将`rawKey`,`rawHashKeys`往下传递。

再来看看核心去查询的方法，execute.

```java
// org.springframework.data.redis.core.RedisTemplate#execute(org.springframework.data.redis.core.RedisCallback<T>, boolean) 
 @Nullable
    public <T> T execute(RedisCallback<T> action, boolean exposeConnection) {
        return this.execute(action, exposeConnection, false);
    }

// 	org.springframework.data.redis.core.RedisTemplate#execute(org.springframework.data.redis.core.RedisCallback<T>, boolean)
	@Nullable
	public <T> T execute(RedisCallback<T> action, boolean exposeConnection) {
		return execute(action, exposeConnection, false);
	}

// org.springframework.data.redis.core.RedisTemplate#execute(org.springframework.data.redis.core.RedisCallback<T>, boolean, boolean)
@Nullable
	public <T> T execute(RedisCallback<T> action, boolean exposeConnection, boolean pipeline) {

		Assert.isTrue(initialized, "template not initialized; call afterPropertiesSet() before using it");
		Assert.notNull(action, "Callback object must not be null");

		RedisConnectionFactory factory = getRequiredConnectionFactory();
		RedisConnection conn = RedisConnectionUtils.getConnection(factory, enableTransactionSupport);

		try {

			boolean existingConnection = TransactionSynchronizationManager.hasResource(factory);
			RedisConnection connToUse = preProcessConnection(conn, existingConnection);

			boolean pipelineStatus = connToUse.isPipelined();
			if (pipeline && !pipelineStatus) {
				connToUse.openPipeline();
			}

			RedisConnection connToExpose = (exposeConnection ? connToUse : createRedisConnectionProxy(connToUse));
			T result = action.doInRedis(connToExpose);

			// close pipeline
			if (pipeline && !pipelineStatus) {
				connToUse.closePipeline();
			}

			return postProcessResult(result, connToUse, existingConnection);
		} finally {
			RedisConnectionUtils.releaseConnection(conn, factory, enableTransactionSupport);
		}
	}
```

可以看到，最终的`T result`是回调`action`执行 doInRedis进行获取的。通过debug进去可以看到, 调用了一开始的hGet：

```java
// org.springframework.data.redis.connection.DefaultedRedisConnection#hMGet
	@Override
	@Deprecated
	default List<byte[]> hMGet(byte[] key, byte[]... fields) {
		return hashCommands().hMGet(key, fields);
	}

//org.springframework.data.redis.connection.lettuce.LettuceHashCommands#hMGet
	@Override
	public List<byte[]> hMGet(byte[] key, byte[]... fields) {

		Assert.notNull(key, "Key must not be null!");
		Assert.notNull(fields, "Fields must not be null!");

		return connection.invoke().fromMany(RedisHashAsyncCommands::hmget, key, fields)
				.toList(source -> source.getValueOrElse(null));
	}
```

这里我们需要关注到这样的代码：*.toList(source -> source.getValueOrElse(null));*

这行代码正是将每条查询的结果合并成List返回的结果，入参是一个`Converter<S, T> converter`接口（及回调）。语意就是将结果进行转换，如果getValue失败，则Else为null。这就是最终返回的List中含有null对象的原因了。

当然，我们可以通过对toList方法进行debug，看到更详细的内容：

![image-20210703150355480](/home/wanglh/.config/Typora/typora-user-images/image-20210703150355480.png)

其中查询的结果是一个KeyValue对象，其中还有一个empty对象，其值为null。

## 总结

1. 这次空指针事件，原因为不清楚RedisTemplate提供的接口，仅看语意相近则使用，没有考虑边界事件，以及为空的事件，开发人员自测不足都占一定的成分。

2. 追查列表中null对象如何产生，光看接口文档仍不足，需要通过阅读源码才能看到原因。 Spring Data Redis 工程的源码抽象程度高，使用回调、函数式编程较多，还需仔细阅读。