---
layout: post
title: "Mybatis-plus 源码解析一：@MapperScan、@Mapper、基础增删改查"
subtitle: ""
author: ""
header-style: text
tags:
  - mybatis
---



# @MapperScan

使用场景：打在配置类或启动类上，从指定路径或当前包所在路径开始，扫描mapper接口，动态生成bean注册到容器中。



## MapperScannerRegistrar

`@MapperScan` 包含了一个元注解 `@Import(MapperScannerRegistrar.class)`。

`@Import` 可以和 `ImportBeanDefinitionRegistrar` 实现类配合使用，`ImportBeanDefinitionRegistrar` 可以动态注册BeanDefinition：

```java
void registerBeanDefinitions(AnnotationMetadata importingClassMetadata, BeanDefinitionRegistry registry)
```

`importingClassMetadata` 为导入类的注解元信息，比如在Application类上打了 `@MapperScan` 注解，因为 `@MapperScan` 中包含了 `@Import`，所以 `importingClassMetadata` 就代表 `Application` 类的注解元信息。

`MapperScannerRegistrar` 中实现如下：

```java
@Override
  public void registerBeanDefinitions(AnnotationMetadata importingClassMetadata, BeanDefinitionRegistry registry) {
      // 从导入类注解元信息中获取@MapperScan中的属性值
    AnnotationAttributes mapperScanAttrs = AnnotationAttributes
        .fromMap(importingClassMetadata.getAnnotationAttributes(MapperScan.class.getName()));
    if (mapperScanAttrs != null) {
        // 接着往下看
      registerBeanDefinitions(importingClassMetadata, mapperScanAttrs, registry,
          generateBaseBeanName(importingClassMetadata, 0));
    }
  }
```

```java
void registerBeanDefinitions(AnnotationMetadata annoMeta, AnnotationAttributes annoAttrs,
      BeanDefinitionRegistry registry, String beanName) {

    // 构造一个 MapperScannerConfigurer 的 BeanDefinitionBuilder
    BeanDefinitionBuilder builder = BeanDefinitionBuilder.genericBeanDefinition(MapperScannerConfigurer.class);
    
    ...省略...
    
    // 从 @MapperScan 中的 value、basePackages、basePackageClasses 读取要扫描的路径
    List<String> basePackages = new ArrayList<>();
    basePackages.addAll( Arrays.stream(annoAttrs.getStringArray("value")).filter(StringUtils::hasText).collect(Collectors.toList());

    basePackages.addAll(Arrays.stream(annoAttrs.getStringArray("basePackages")).filter(StringUtils::hasText)
        .collect(Collectors.toList()));

                        				                                      basePackages.addAll(Arrays.stream(annoAttrs.getClassArray("basePackageClasses")).map(ClassUtils::getPackageName)
        .collect(Collectors.toList()));

    // 如果都没指定，取当前包名
    if (basePackages.isEmpty()) {
      basePackages.add(getDefaultBasePackage(annoMeta));
    }

    ...省略...

    // 将扫描路径设置到 MapperScannerConfigurer 的 basePackage属性                  
    builder.addPropertyValue("basePackage", StringUtils.collectionToCommaDelimitedString(basePackages));

    // 注册BeanDefinition
    registry.registerBeanDefinition(beanName, builder.getBeanDefinition());

  }
```



所以 `MapperScannerRegistrar` 的作用就是注册 `MapperScannerConfigurer` 的 BeanDefinition，**将 `@MapperScan` 的属性值设置进去**，然后由 `MapperScannerConfigurer`来完成 mapper 接口的扫描注册工作。



------

<br/><br/>

## MapperScannerConfigurer

`MapperScannerConfigurer` 实现了 `BeanDefinitionRegistryPostProcessor` 接口，`postProcessBeanDefinitionRegistry` 方法实现如下：

```java
@Override
  public void postProcessBeanDefinitionRegistry(BeanDefinitionRegistry registry) {
    ...省略...

    // class路径扫描器    
    ClassPathMapperScanner scanner = new ClassPathMapperScanner(registry);
    ...省略...
    scanner.registerFilters();
    scanner.scan(
        StringUtils.tokenizeToStringArray(this.basePackage, ConfigurableApplicationContext.CONFIG_LOCATION_DELIMITERS));
  }
```

`registerFilters`  注册扫描器的过滤器，可根据注解、接口等进行过滤。

`ClassPathMapperScanner`  是 `mybatis-spring` 的类，继承自 spring-context 中的 `ClassPathBeanDefinitionScanner`。重写了 `doScan` 方法和  `isCandidateComponent` 方法。

`isCandidateComponent`  指定了只要接口且不是内部类。

`doScan` 方法调用了父类的 doScan 后(**这一步已经将扫描到的BeanDefinition注册到容器中了** )，对扫描到的 beanDefinition 做了进一步处理：

```java
@Override
  public Set<BeanDefinitionHolder> doScan(String... basePackages) {
    // 这一步已经将扫描到的BeanDefinition注册到容器中了  
    Set<BeanDefinitionHolder> beanDefinitions = super.doScan(basePackages);

    if (beanDefinitions.isEmpty()) {
      LOGGER.warn(() -> "No MyBatis mapper was found in '" + Arrays.toString(basePackages)
          + "' package. Please check your configuration.");
    } else {
        // 往下看
      processBeanDefinitions(beanDefinitions);
    }

    return beanDefinitions;
  }
```



```java
private void processBeanDefinitions(Set<BeanDefinitionHolder> beanDefinitions) {
    AbstractBeanDefinition definition;
    BeanDefinitionRegistry registry = getRegistry();
    // 遍历扫描到的BeanDefinition
    for (BeanDefinitionHolder holder : beanDefinitions) {
      definition = (AbstractBeanDefinition) holder.getBeanDefinition();
      ...省略...
      // 得到beanClassName,比如 a.b.c.UserMapper
      String beanClassName = definition.getBeanClassName();
      LOGGER.debug(() -> "Creating MapperFactoryBean with name '" + holder.getBeanName() + "' and '" + beanClassName
          + "' mapperInterface");
  
      // the mapper interface is the original class of the bean
      // but, the actual class of the bean is MapperFactoryBean
      // 设置构造函数值为beanClassName，MapperFactoryBean的构造函数为：public MapperFactoryBean(Class<T> mapperInterface) ，需要传递mapper的class
      definition.getConstructorArgumentValues().addGenericArgumentValue(beanClassName); // issue #59
      // 将mapper的 BeanDefinition 的class属性设为 MapperFactoryBean
      definition.setBeanClass(this.mapperFactoryBeanClass);

      ...省略...

    }
  }
```

所以这时每个 mapper 对应的 BeanDefinition 实际上都是 `MapperFactoryBean` 类型了，是一个工厂bean，并持有一个 `mapperInterface`  属性，对应mapper的class。

> ps：`FactoryBean`  是懒加载，只有在第一次使用到对应类型的对象时，才会调用 `getObject`  获取对象。如果要急加载，可实现 `SmartFactoryBean` 接口，有一个 `isEagerInit`  方法。

------

<br/><br/>



# @Mapper

使用场景：单独定义在mapper接口上



## MybatisPlusAutoConfiguration#AutoConfiguredMapperScannerRegistrar

在 `MybatisPlusAutoConfiguration` 中有一个内部配置类 `MapperScannerRegistrarNotFoundConfiguration`：

```java
	@Configuration(proxyBeanMethods = false)
    @Import(AutoConfiguredMapperScannerRegistrar.class)
    @ConditionalOnMissingBean({MapperFactoryBean.class, MapperScannerConfigurer.class})
    public static class MapperScannerRegistrarNotFoundConfiguration 
```

当没有 `MapperFactoryBean` 和 `MapperScannerConfigurer` 类型的bean时，该配置类会生效。而这两种类型的bean只有当使用了 `@MapperScan` 注解时才会被动态注入，详见上一节。所以，换言之，没使用 `@MapperScan` 时会使用这个配置。该配置类import了另一个配置类 `AutoConfiguredMapperScannerRegistrar`：

```java
public static class AutoConfiguredMapperScannerRegistrar implements BeanFactoryAware, ImportBeanDefinitionRegistrar 
```

可发现也是一个 `ImportBeanDefinitionRegistrar` 实现类，基本套路也和 `MapperScannerRegistrar` 一致，看它的 `registerBeanDefinitions` 方法：

```java
		@Override
        public void registerBeanDefinitions(AnnotationMetadata importingClassMetadata, BeanDefinitionRegistry registry) {

            ...省略...

            logger.debug("Searching for mappers annotated with @Mapper");

            // 获取自动配置类所在包路径，即打了 @SpringbootApplication 的启动类，因为 @Maper 不需要指定扫描路径，那么就从			// 根路径开始扫描
            List<String> packages = AutoConfigurationPackages.get(this.beanFactory);
            if (logger.isDebugEnabled()) {
                packages.forEach(pkg -> logger.debug("Using auto-configuration base package '{}'", pkg));
            }

            // 同样，构建一个 MapperScannerConfigurer 的 BeanDefinition
            BeanDefinitionBuilder builder = BeanDefinitionBuilder.genericBeanDefinition(MapperScannerConfigurer.class);
            ...省略...
            // 给MapperScannerConfigurer设置annotationClass属性，相当于 @MapperScan 中的annotationClass属性     
            builder.addPropertyValue("annotationClass", Mapper.class);
            // 设置basePackage
            builder.addPropertyValue("basePackage", StringUtils.collectionToCommaDelimitedString(packages));
            ...省略...
            // 注册 MapperScannerConfigurer 的 BeanDefinition   
            registry.registerBeanDefinition(MapperScannerConfigurer.class.getName(), builder.getBeanDefinition());
        }
```

后面的处理逻辑就是 `MapperScannerConfigurer` 的处理逻辑了，和 `@MapperScan` 一样。



------

<br/><br/>



# mybatis-plus 增删改查

上文说到，mapper 都是 `MapperFactoryBean`，所以入口就在  `getObject`：

```java
@Override
  public T getObject() throws Exception {
    return getSqlSession().getMapper(this.mapperInterface);
  }
```

对应 `SqlSession` 接口的 `<T> T getMapper(Class<T> type)` 方法。`SqlSession` 有两个主要实现类：`DefaultSqlSession(mybatis)` 和 `SqlSessionTemplate(mybatis-spring)`。此时的实现类是 `SqlSessionTemplate`。

 `MapperFactoryBean` 除了 `mapperInterface` 外，还有两个属性：`sqlSessionFactory` 和 `sqlSessionTemplate`。这两个属性是在何时设置进去的？

回到 `MybatisPlusAutoConfiguration`，该配置类还生成了两个bean：

```java
	@Bean
    @ConditionalOnMissingBean
    public SqlSessionFactory sqlSessionFactory(DataSource dataSource) 
        
    @Bean
    @ConditionalOnMissingBean
    public SqlSessionTemplate sqlSessionTemplate(SqlSessionFactory sqlSessionFactory) 
```

上文讲到 `ClassPathMapperScanner` 的  `processBeanDefinitions` 方法，用于给每个扫描到的 mapper BeanDefinition 设置属性，这一步会将 mapper 的 class 都设为 `MapperFactoryBean`，除此之外，还设置了 `autowireMode`：

```java
definition.setAutowireMode(AbstractBeanDefinition.AUTOWIRE_BY_TYPE);
```

表示对 `MapperFactoryBean` 启用属性按类型自动注入，在spring容器初始化过程中，会初始化每个 `MapperFactoryBean`，会把 `MybatisPlusAutoConfiguration` 中配置的bean设置进去。

回到 `getMapper` 方法，经过层层嵌套，最终到了 `com.baomidou.mybatisplus.core.MybatisMapperRegistry#getMapper`：

```java
	@Override
    public <T> T getMapper(Class<T> type, SqlSession sqlSession) {
        // TODO 这里换成 MybatisMapperProxyFactory 而不是 MapperProxyFactory
        final MybatisMapperProxyFactory<T> mapperProxyFactory = (MybatisMapperProxyFactory<T>) knownMappers.get(type);
        if (mapperProxyFactory == null) {
            throw new BindingException("Type " + type + " is not known to the MybatisPlusMapperRegistry.");
        }
        try {
            // 往下看
            return mapperProxyFactory.newInstance(sqlSession);
        } catch (Exception e) {
            throw new BindingException("Error getting mapper instance. Cause: " + e, e);
        }
    }
```

> mybatis-plus 继承了 mybatis 很多类，由于这些类mybatis在设计时并没有打算可扩展，所以mybatis-plus的做法是直接全部copy过来，然后进行扩展。



接着点进去：

```java
	protected T newInstance(MybatisMapperProxy<T> mapperProxy) {
        return (T) Proxy.newProxyInstance(mapperInterface.getClassLoader(), new Class[]{mapperInterface}, mapperProxy);
    }

    public T newInstance(SqlSession sqlSession) {
        final MybatisMapperProxy<T> mapperProxy = new MybatisMapperProxy<>(sqlSession, mapperInterface, methodCache);
        return newInstance(mapperProxy);
    }
```

可看到最终是使用了Jdk的动态代理，mapper最终对应的是 mapperInterface 的一个动态代理类。

------

<br/><br/>





## 注入基本增删改查

那么为何只需要继承 `BaseMapper` 接口就能实现基本的增删改查呢？

mapper基本调用逻辑上文已讲过，撇去层层嵌套后，最终是归功于 `com.baomidou.mybatisplus.core.MybatisConfiguration#getMappedStatement(java.lang.String)`：

```java
public MappedStatement getMappedStatement(String id) {
    return this.getMappedStatement(id, true);
  }
```

`id` 为 mapper 的方法全路径，比如：`a.b.c.UserMapper.insert`，`MappedStatement` 里包含了对应要执行的sql(SqlSource类->DynamicSqlSource)。`MybatisConfiguration` 中维护了一个map，存储的就是这样的键值对类型，每个mapper的每个方法都对应了一个MappedStatement。

所以执行的逻辑基本上就是，根据方法找到对应的MappedStatement，结合传递进来的参数，生成最终的sql，生成 PreparedStatement 对象，后面就是jdbc的流程了。

那么这个map是如何生成的？

`MapperFactoryBean` 继承了 `DaoSupport` 接口，`DaoSupport` 是 Spring-tx 中 DAO 的基类，实现类有 JdbcDaoSupport 等。`DaoSupport` 实现了 `InitializingBean` 接口，`afterPropertiesSet` 实现如下：

```java
	@Override
	public final void afterPropertiesSet() throws IllegalArgumentException, BeanInitializationException {
		// Let abstract subclasses check their configuration.
		checkDaoConfig();

		// Let concrete implementations initialize themselves.
		try {
			initDao();
		}
		catch (Exception ex) {
			throw new BeanInitializationException("Initialization of DAO failed", ex);
		}
	}
```

`MapperFactoryBean` 重写了 `checkDaoConfig`  方法

```java
  @Override
  protected void checkDaoConfig() {
    super.checkDaoConfig();

    notNull(this.mapperInterface, "Property 'mapperInterface' is required");

    Configuration configuration = getSqlSession().getConfiguration();
    if (this.addToConfig && !configuration.hasMapper(this.mapperInterface)) {
      try {
        // 在这一步添加mapper，最终会生成该mapperInterface中所有方法的键值对，放到map中去
        configuration.addMapper(this.mapperInterface);
      } catch (Exception e) {
        logger.error("Error while adding the mapper '" + this.mapperInterface + "' to configuration.", e);
        throw new IllegalArgumentException(e);
      } finally {
        ErrorContext.instance().reset();
      }
    }
  }
```

`configuration.addMapper(this.mapperInterface)` 详细流程很复杂，简单来说就是会利用反射获取到 `BaseMapper<T>` 中的泛型参数，以此为表名，获取到表相关信息，封装为 `TableInfo` 对象。Mybatis-plus为基本增删改查操作定义了一系列的类，如`Insert`、`Delete`、`Update` 等等，以 `Insert`类为例：

```java
public class Insert extends AbstractMethod {

    public Insert() {
        super(SqlMethod.INSERT_ONE.getMethod());
    }
```

 `SqlMethod` 是一个枚举：

```java
public enum SqlMethod {
    /**
     * 插入
     */
    INSERT_ONE("insert", "插入一条数据（选择字段插入）", "<script>\nINSERT INTO %s %s VALUES %s\n</script>"),
```

相当于定义了一个sql模板字符串，第一个占位符是表名，第二个占位符是列名字符串，第三个占位符是值字符串。

------

<br/><br/><br/>





# 未完待续！！！