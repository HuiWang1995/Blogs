# Mybatis Mapper Bean 生成源码分析一

## 问题

我们的Mapper接口，用的是@Mapper，而非@Service等会被Spring扫描的注解，那么它最后怎么会生成一个Bean呢。

## 重点方法一：`ClassPathMapperScanner#doScan`

`org.mybatis.spring.mapper.ClassPathMapperScanner`继承自`org.springframework.context.annotation.ClassPathBeanDefinitionScanner`,重载了其`#doScan`方法，调用其父类进行包下的扫描，再对扫描出来的`BeanDefinitionHolder beanDefinitions` 进行处理，代码如下：

```java

  /**
   * Calls the parent search that will search and register all the candidates. Then the registered objects are post
   * processed to set them as MapperFactoryBeans
   */
  @Override
  public Set<BeanDefinitionHolder> doScan(String... basePackages) {
    // 让Spring的父类负责扫描
    Set<BeanDefinitionHolder> beanDefinitions = super.doScan(basePackages);

    if (beanDefinitions.isEmpty()) {
      LOGGER.warn(() -> "No MyBatis mapper was found in '" + Arrays.toString(basePackages)
          + "' package. Please check your configuration.");
    } else {
      // mybatis 定制，修改了bean 定义
      processBeanDefinitions(beanDefinitions);
    }

    return beanDefinitions;
  }

```

## 重点方法二：`ClassPathMapperScanner#processBeanDefinitions`

```java
private Class<? extends MapperFactoryBean> mapperFactoryBeanClass = MapperFactoryBean.class;

private void processBeanDefinitions(Set<BeanDefinitionHolder> beanDefinitions) {
    GenericBeanDefinition definition;
    for (BeanDefinitionHolder holder : beanDefinitions) {
      definition = (GenericBeanDefinition) holder.getBeanDefinition();
      String beanClassName = definition.getBeanClassName();
        
      // the mapper interface is the original class of the bean
      // but, the actual class of the bean is MapperFactoryBean
      definition.getConstructorArgumentValues().addGenericArgumentValue(beanClassName); 
      // 替换成了MapperFactoryBean，一个FactoryBean，在Bean真正产生时用来创建Bean的工厂
      definition.setBeanClass(this.mapperFactoryBeanClass);
    }
}
```

## 总结

mybatis通过扩展doScan方法，将扫描到的Mapper的BeanDefinitionHolder中的beanClass从接口，改成了MapperFactoryBean，完成了这波偷天换日的第一步，之后Bean的初始化时，最终将通过MapperFactoryBean扩展的FactoryBean 的 getObject方法返回真正的Mapper(JDK 代理)。

