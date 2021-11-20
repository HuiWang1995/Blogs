# Mybatis Mapper Bean 生成源码分析二

## 问题

在java程序中，我们往往一个`@Autowired`注解就可以获取到一个Mapper接口实例，一个未经我们实现的接口实例，那么这个Bean实例是怎么在Spring初始化的过程中注入的呢？ 让我们来找找答案。

探寻内容偏长，没有DEBUG过且不耐烦可直接跳过去看总结~



## MapperFactoryBean创建工厂

```
public T getObject() throws Exception {
	// getSqlSession() 即 SqlSessionTemplate;
    return this.getSqlSession().getMapper(this.mapperInterface);
}
```

## 关键类`org.mybatis.spring.SqlSessionTemplate`

在`org.mybatis.spring.SqlSessionTemplate`中有一个getMapper方法，如下：

```java
    public <T> T getMapper(Class<T> type) {
        return this.getConfiguration().getMapper(type, this);
    }
```

这个SqlSessionTemplate会被注入到容器中，而且能够在我们代码中使用，获取到一个Mapper。使用如下：

```java
    @Autowired
    public SqlSessionTemplate sqlSession;

	public void demo() {
        sqlSession.getMapper(UserMapper.class);
    }
```

显然这个SqlSessionTemplate类可以做到根据接口取到实例，那么Spring启动时其实也是根据它来获取的。



## spring启动过程注入一个动态代理对象

对于如下一个需要Mapper的服务，启动过程中Mapper必然会被注入进来：

```java
@Component
@RestController
public class UserController {

    @Autowired
    UserMapper userMapper;
}

```

那么其会走spring的如下方法来获取Bean:

```java
// org.springframework.beans.factory.support.AbstractBeanFactory#doGetBean
// 方法很长，将会省略很多
protected <T> T doGetBean(
			String name, @Nullable Class<T> requiredType, @Nullable Object[] args, boolean typeCheckOnly)
			throws BeansException {
    // beanName即userMapper
    String beanName = transformedBeanName(name);
	Object bean;
    
	// Eagerly check singleton cache for manually registered singletons.
    // 获取的sharedInstance即为工厂
	Object sharedInstance = getSingleton(beanName);
    // 启动时，第一次实际产生工厂代码为下方代码，生成了MapperFactoryBean
    sharedInstance = getSingleton(beanName, () -> {
						try {
                            // 作为懒加载的方法，最终创建工厂对象，值得一看
							return createBean(beanName, mbd, args);
						}
						catch (BeansException ex) {

							destroySingleton(beanName);
							throw ex;
						}
					});
    // 通过工厂来生成Bean实例
    bean = getObjectForBeanInstance(sharedInstance, name, beanName, mbd);
    return (T) bean;
}
```



```java
// org.springframework.beans.factory.support.FactoryBeanRegistrySupport#doGetObjectFromFactoryBean
	private Object doGetObjectFromFactoryBean(FactoryBean<?> factory, String beanName) throws BeanCreationException {
		Object object;
		try {
			if (System.getSecurityManager() != null) {
				// 省略
			}
			else {
                // 重点，通过工厂进行获取Bean
				object = factory.getObject();
			}
		}
		return object;
	}

```

其堆栈最终如下图：

![image-20210911211048537](/home/wanglh/Documents/Blogs/mybatis/pic/image-20210911211048537.png)

所以可以看到在下方代码中，返回了一个JDK动态代理的对象，就是我们服务内会注入的名为“userMapper”的Bean了：

```java
// com.baomidou.mybatisplus.core.MybatisMapperRegistry#getMapper
    @SuppressWarnings("unchecked")
    @Override
    public <T> T getMapper(Class<T> type, SqlSession sqlSession) {
        // 代理工厂
        final MybatisMapperProxyFactory<T> mapperProxyFactory = (MybatisMapperProxyFactory<T>) knownMappers.get(type);
        if (mapperProxyFactory == null) {
            throw new BindingException("Type " + type + " is not known to the MybatisPlusMapperRegistry.");
        }
        try {
            // 通过代理工厂，来生成对应的代理实例
            return mapperProxyFactory.newInstance(sqlSession);
        } catch (Exception e) {
            throw new BindingException("Error getting mapper instance. Cause: " + e, e);
        }
    }
```

## 代理工厂生成一个怎样的实例

来看看MybatisMapperProxyFactory，究竟是怎么样一个工厂

```java
// com.baomidou.mybatisplus.core.override.MybatisMapperProxyFactory
public class MybatisMapperProxyFactory<T> {
    
    // 接口名称
    @Getter
    private final Class<T> mapperInterface;
    @Getter
    private final Map<Method, MybatisMapperProxy.MapperMethodInvoker> methodCache = new ConcurrentHashMap<>();
    
    public MybatisMapperProxyFactory(Class<T> mapperInterface) {
        this.mapperInterface = mapperInterface;
    }

    @SuppressWarnings("unchecked")
    protected T newInstance(MybatisMapperProxy<T> mapperProxy) {
        // 最终包装成JDK动态代理的对象
        return (T) Proxy.newProxyInstance(mapperInterface.getClassLoader(), new Class[]{mapperInterface}, mapperProxy);
    }

    // debug可查看实际走这一个方法来产生
    public T newInstance(SqlSession sqlSession) {
        final MybatisMapperProxy<T> mapperProxy = new MybatisMapperProxy<>(sqlSession, mapperInterface, methodCache);
        return newInstance(mapperProxy);
    }
}
```

```java
// com.baomidou.mybatisplus.core.override.MybatisMapperProxy
public class MybatisMapperProxy<T> implements InvocationHandler, Serializable {
    // bean最后实际会调用到这个代理的invoke方法，由此方法做路由到具体方法
    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        try {
            if (Object.class.equals(method.getDeclaringClass())) {
                return method.invoke(this, args);
            } else {
                return cachedInvoker(method).invoke(proxy, method, args, sqlSession);
            }
        } catch (Throwable t) {
            throw ExceptionUtil.unwrapThrowable(t);
        }
    }
    
    private MapperMethodInvoker cachedInvoker(Method method) throws Throwable {
        try {
            return CollectionUtils.computeIfAbsent(methodCache, method, m -> {
                if (m.isDefault()) {
                    try {
                        if (privateLookupInMethod == null) {
                            return new DefaultMethodInvoker(getMethodHandleJava8(method));
                        } else {
                            return new DefaultMethodInvoker(getMethodHandleJava9(method));
                        }
                    } catch (IllegalAccessException | InstantiationException | InvocationTargetException
                        | NoSuchMethodException e) {
                        throw new RuntimeException(e);
                    }
                } else {
                    // 缓存一个Mapper方法
                    return new PlainMethodInvoker(new MybatisMapperMethod(mapperInterface, method, sqlSession.getConfiguration()));
                }
            });
        } catch (RuntimeException re) {
            Throwable cause = re.getCause();
            throw cause == null ? re : cause;
        }
    }
    
}
```

- 最终使用的方法：通过这个方法最终决定走查询、插入、更新等等命令。

```java
// com.baomidou.mybatisplus.core.override.MybatisMapperMethod#execute

    public Object execute(SqlSession sqlSession, Object[] args) {
        Object result;
        switch (command.getType()) {
            case INSERT: {
                Object param = method.convertArgsToSqlCommandParam(args);
                result = rowCountResult(sqlSession.insert(command.getName(), param));
                break;
            }
            case UPDATE: {
                Object param = method.convertArgsToSqlCommandParam(args);
                result = rowCountResult(sqlSession.update(command.getName(), param));
                break;
            }
            case DELETE: {
                Object param = method.convertArgsToSqlCommandParam(args);
                result = rowCountResult(sqlSession.delete(command.getName(), param));
                break;
            }
            case SELECT:
                if (method.returnsVoid() && method.hasResultHandler()) {
                    executeWithResultHandler(sqlSession, args);
                    result = null;
                } else if (method.returnsMany()) {
                    result = executeForMany(sqlSession, args);
                } else if (method.returnsMap()) {
                    result = executeForMap(sqlSession, args);
                } else if (method.returnsCursor()) {
                    result = executeForCursor(sqlSession, args);
                } else {
                    // TODO 这里下面改了
                    if (IPage.class.isAssignableFrom(method.getReturnType())) {
                        result = executeForIPage(sqlSession, args);
                        // TODO 这里上面改了
                    } else {
                        Object param = method.convertArgsToSqlCommandParam(args);
                        result = sqlSession.selectOne(command.getName(), param);
                        if (method.returnsOptional()
                            && (result == null || !method.getReturnType().equals(result.getClass()))) {
                            result = Optional.ofNullable(result);
                        }
                    }
                }
                break;
            case FLUSH:
                result = sqlSession.flushStatements();
                break;
            default:
                throw new BindingException("Unknown execution method for: " + command.getName());
        }
        if (result == null && method.getReturnType().isPrimitive() && !method.returnsVoid()) {
            throw new BindingException("Mapper method '" + command.getName()
                + " attempted to return null from a method with a primitive return type (" + method.getReturnType() + ").");
        }
        return result;
    }

```

## 总结

有如下几步，来进行Mapper接口Bean的注入

1. spring doGetBean 方法获取到bean工厂
2. bean工厂通过SqlSessionTemplate来进行获取（创建）Mapper实例
3. MybatisMapperProxyFactory工厂通过newInstance构建了一个通过JDK代理过的MybatisMapperProxy对象。
4. 该代理对象的invoke方法完成了Mapper接口方法的路由，到具体SQL命令中去。

