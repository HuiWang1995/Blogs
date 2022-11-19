# Mybatis Mapper Bean 生成源码分析三--自定义Mapper.md

## 运行中添加Mapper

先上代码：

```java
@Autowired
public SqlSessionTemplate sqlSession;

    @GetMapping("/list")
    @ResponseBody
    public String getAll() {
        try {
            // 获取失败
            BaseMapper mapper2 = sqlSession.getMapper(User2Mapper.class);
        } catch (Exception e) {
            e.printStackTrace();
        }
        // 添加
        sqlSession.getConfiguration().addMapper(User2Mapper.class);
        // 获取
        BaseMapper mapper3 = sqlSession.getMapper(User2Mapper.class);
        mapper1.selectCount(new LambdaQueryWrapper());
        mapper3.selectCount(new LambdaQueryWrapper());

    }
```

```java
//@Mapper
public interface User2Mapper extends BaseMapper<UserEntity> {

}
```

代码中，`User2Mapper`特意将@Mapper注解注释掉，那么在Spring启动过程中，该Mapper不会被加载，所以获取时就会获取不到。而`com.baomidou.mybatisplus.core.MybatisConfiguration`中提供了addMapper方法。通过这么一个方法，就可以将未进入容器的Mapper加进容器中了。



## 实际添加Mapper还是一个动态代理

通过debug可以发现，addMapper最终调用到的是`com.baomidou.mybatisplus.core.MybatisMapperRegistry#addMapper`:

```java
    @Override
    public <T> void addMapper(Class<T> type) {
        if (type.isInterface()) {
            if (hasMapper(type)) {
                return;
            }
            boolean loadCompleted = false;
            try {
                // 构建代理工厂MybatisMapperProxyFactory
                knownMappers.put(type, new MybatisMapperProxyFactory<>(type));
                // mapper解析器
                MybatisMapperAnnotationBuilder parser = new MybatisMapperAnnotationBuilder(config, type);
                // 解析器解析
                // 核心逻辑，读取xml，解析Mapper接口方法、缓存、parseStatement等等
                parser.parse();
                loadCompleted = true;
            } finally {
                if (!loadCompleted) {
                    knownMappers.remove(type);
                }
            }
        }
    }
```

## 总结

addMapper实现逻辑与spring初始化过程中getMapper调用的代码很相似。不过再添加的Mapper注册在MybatisConfiguration中，并无Spring Bean，所以在Spring上下文中获取该Bean也将会空指针，不能自动注入了。
