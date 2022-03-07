---
layout: post
title: "Mybatis-plus 源码解析二：一级缓存和二级缓存"
subtitle: ""
author: ""
header-style: text
tags:
  - mybatis

---





mybatis-plus本身不提供缓存功能，一级缓存和二级缓存是mybatis中的实现



# 一级缓存

又称本地缓存。在同一个 SqlSession 内可以对相同sql的执行结果进行缓存。

```java
@Test
    public void testFirstLevelCacheOnlyQueryOnce() {
        SqlSession session = sqlSessionFactory.openSession(true);
        UserMapper userMapper = session.getMapper(UserMapper.class);
        System.out.println(userMapper.selectById(1));
        System.out.println(userMapper.selectById(1));
    }
```

从日志中可看到，两次相同的查询只会查询一次数据库。

```java
@Test
    public void testFirstLevelCacheQueryAfterUpdate() {
        SqlSession session = sqlSessionFactory.openSession(true);
        UserMapper userMapper = session.getMapper(UserMapper.class);
        System.out.println(userMapper.selectById(1));
        userMapper.updateById(new User().setId(10086L).setName("foo").setAge(100).setEmail("foo@foo.com"));
        System.out.println(userMapper.selectById(1));
    }
```

更新后，会重新查询一次数据库。

```java
@Test
    public void testFirstLevelCacheInTwoSession() {
        SqlSession session1 = sqlSessionFactory.openSession(true);
        SqlSession session2 = sqlSessionFactory.openSession(true);
        UserMapper userMapper1 = session1.getMapper(UserMapper.class);
        UserMapper userMapper2 = session2.getMapper(UserMapper.class);

        System.out.println("userMapper1.selectById(1): " + userMapper1.selectById(1));
        System.out.println("userMapper1.selectById(1): " + userMapper1.selectById(1));
        // 会重新查询一次数据库
        System.out.println("userMapper2.selectById(1): " + userMapper2.selectById(1));

        // session2 更新了数据
        userMapper2.updateById(new User().setId(1L).setName("foo").setAge(100).setEmail("foo@foo.com"));

        // session1查到了脏数据
        System.out.println("userMapper1.selectById(1): " + userMapper1.selectById(1));
        // session2会重新查询一次，读到的是最新的数据
        System.out.println("userMapper2.selectById(1): " + userMapper2.selectById(1));
    }
```

可看出，一级缓存的范围是同一个SqlSession内。



### 源码分析

先认识几个顶级接口：

`SqlSession`：mybatis的主要接口。通过这个接口来执行增删改查、获取mapper、管理事务等。

`Executor`：顾名思义，执行器。具体干活的接口，包括update、query、commit、rollback等。

每个 SqlSession 有一个 Executor，Executor 有个抽象实现类 BaseExecutor，BaseExecutor 中有个cache：

```java
protected PerpetualCache localCache;
```

PerpetualCache 内部就是个 HashMap

```java
public class PerpetualCache implements Cache {

  private final String id;

  private final Map<Object, Object> cache = new HashMap<>();

  public PerpetualCache(String id) {
    this.id = id;
  }
```

所以一级缓存的范围就是SqlSession，同一个session内共享同一个cache，不同session之间缓存互不影响。



#### 缓存key

查询会走到 BaseExecutor 的这个方法：

```java
@Override
  public <E> List<E> query(MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler) throws SQLException {
    BoundSql boundSql = ms.getBoundSql(parameter);
    CacheKey key = createCacheKey(ms, parameter, rowBounds, boundSql);
    return query(ms, parameter, rowBounds, resultHandler, key, boundSql);
  }
```

`createCacheKey` 会生成一个cache key，实现如下：

```java
@Override
  public CacheKey createCacheKey(MappedStatement ms, Object parameterObject, RowBounds rowBounds, BoundSql boundSql) {
    ...省略...
    CacheKey cacheKey = new CacheKey();
    cacheKey.update(ms.getId());
    cacheKey.update(rowBounds.getOffset());
    cacheKey.update(rowBounds.getLimit());
    cacheKey.update(boundSql.getSql());
    List<ParameterMapping> parameterMappings = boundSql.getParameterMappings();
    TypeHandlerRegistry typeHandlerRegistry = ms.getConfiguration().getTypeHandlerRegistry();
    // mimic DefaultParameterHandler logic
    for (ParameterMapping parameterMapping : parameterMappings) {
      if (parameterMapping.getMode() != ParameterMode.OUT) {
        Object value;
        String propertyName = parameterMapping.getProperty();
        if (boundSql.hasAdditionalParameter(propertyName)) {
          value = boundSql.getAdditionalParameter(propertyName);
        } else if (parameterObject == null) {
          value = null;
        } else if (typeHandlerRegistry.hasTypeHandler(parameterObject.getClass())) {
          value = parameterObject;
        } else {
          MetaObject metaObject = configuration.newMetaObject(parameterObject);
          value = metaObject.getValue(propertyName);
        }
        cacheKey.update(value);
      }
    }
    if (configuration.getEnvironment() != null) {
      // issue #176
      cacheKey.update(configuration.getEnvironment().getId());
    }
    return cacheKey;
  }
```

CacheKey 的 update 方法就是往一个List里add

```java
public void update(Object object) {
    int baseHashCode = object == null ? 1 : ArrayUtil.hashCode(object);

    count++;
    checksum += baseHashCode;
    baseHashCode *= count;

    hashcode = multiplier * hashcode + baseHashCode;

    updateList.add(object);
  }
```

所以一个CacheKey是由以下部分组成的：MappedStatement的id(即mapper方法全路径，如a.b.c.UserMapper.selectById)、offset和limit、sql、参数。当两个mapper方法以上部分都匹配时，会直接从缓存中取出对应的值。





#### 刷新缓存

除查询以外的方法，包括增删改，都会进入 BaseExecutor 的 update 方法：

```java
@Override
  public int update(MappedStatement ms, Object parameter) throws SQLException {
    ErrorContext.instance().resource(ms.getResource()).activity("executing an update").object(ms.getId());
    if (closed) {
      throw new ExecutorException("Executor was closed.");
    }
    clearLocalCache();
    return doUpdate(ms, parameter);
  }
```

调用了 `clearLocalCache`：

```java
@Override
  public void clearLocalCache() {
    if (!closed) {
      localCache.clear();
      localOutputParameterCache.clear();
    }
  }
```

所以，增删改会简单粗暴的清空所有缓存，之后的查询会再次查询数据库。



#### 缓存范围

一级缓存可选范围有两个：SESSION、STATEMENT。

配置项为：

```yaml
mybatis-plus:
  configuration:
    local-cache-scope: session(或statement)
```

默认为session，之前的分析都是在该配置下。

在 BaseExecutor 的 query 方法中有个判断：

```java
@Override
public <E> List<E> query(MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler, CacheKey key, BoundSql boundSql) throws SQLException {
    ...省略...
    try {
        queryStack++;
        // 先从缓存中获取数据
        list = resultHandler == null ? (List<E>) localCache.getObject(key) : null;
        if (list != null) {
            handleLocallyCachedOutputParameters(ms, key, parameter, boundSql);
        } else {
            // 未从缓存中获取到数据时直接从数据库中查询
            list = queryFromDatabase(ms, parameter, rowBounds, resultHandler, key, boundSql);
        }
    } finally {
        queryStack--;
    }
    if (queryStack == 0) {
        for (BaseExecutor.DeferredLoad deferredLoad : deferredLoads) {
            deferredLoad.load();
        }
        // issue #601
        deferredLoads.clear();
        // 如果参数localCacheScope值为STATEMENT，则每次查询之后都清空缓存
        if (configuration.getLocalCacheScope() == LocalCacheScope.STATEMENT) {
            // issue #482
            clearLocalCache();
        }
    }
    return list;
}
```

当范围为statement时，每次查询时会先查询缓存或直接从数据库查询，之后都会清空缓存。所以个人理解，statement范围的一级缓存其实相当于是关闭了一级缓存，因为每次查询后都会清空缓存，增删改操作也会清空缓存(如上文所述)，那就用不上缓存了嘛。

------

<br/><br/><br/>





# 二级缓存

二级缓存就是为了解决一级缓存不能跨session的问题。

既然一级缓存是存在于每个SqlSession内，那就弄一个全局的cache不就行了。

使用方法：

- 开启二级缓存：

  ```yaml
  mybatis-plus:
    configuration:
      cache-enabled: true
  ```

  默认即为true

- 在要开启二级缓存的mapper上加上注解：`@CacheNamespace` 即可

`cache-enabled` 设为true后，在生成Executor实例时会用 `CachingExecutor` 包装一下(装饰器模式)，以提供全局缓存功能。

二级缓存是以 `namespace` 作为一个分组，同一个namespace内的缓存都可以共享，不再局限于单个SqlSession内。`@CacheNamespace` 即表示为该mapper开启一个namespace，该mapper内的方法共享一个缓存，当然，底层还是CacheKey，即key的生成和匹配还是按照上一节讲的逻辑。



### commit后二级缓存才会生效

```java
@Test
    public void testSecondaryCache() {
        SqlSession session1 = sqlSessionFactory.openSession(true);
        SqlSession session2 = sqlSessionFactory.openSession(true);
        UserWithCacheMapper userMapper1 = session1.getMapper(UserWithCacheMapper.class);
        UserWithCacheMapper userMapper2 = session2.getMapper(UserWithCacheMapper.class);

        System.out.println("userMapper1.selectById(1): " + userMapper1.selectById(1));
        System.out.println("userMapper1.selectById(1): " + userMapper1.selectById(1));
        System.out.println("userMapper2.selectById(1): " + userMapper2.selectById(1));
    }
```

从日志观察发现，上面3次 selectById 查询了2次数据库(第一个和第三个查询了数据库，第二个使用了一级缓存)，二级缓存并没有生效。

在第二次查询后加入 `session1.commit()`，3次查询只查询了一次数据库，二级缓存生效。



#### 源码分析

直接从CachingExecutor看起，查询会走以下方法：

```java
@Override
  public <E> List<E> query(MappedStatement ms, Object parameterObject, RowBounds rowBounds, ResultHandler resultHandler, CacheKey key, BoundSql boundSql)
      throws SQLException {
    Cache cache = ms.getCache();
    if (cache != null) {
      flushCacheIfRequired(ms);
      if (ms.isUseCache() && resultHandler == null) {
        ensureNoOutParams(ms, boundSql);
        @SuppressWarnings("unchecked")
        List<E> list = (List<E>) tcm.getObject(cache, key);
        if (list == null) {
          list = delegate.query(ms, parameterObject, rowBounds, resultHandler, key, boundSql);
          tcm.putObject(cache, key, list); // issue #578 and #116
        }
        return list;
      }
    }
    return delegate.query(ms, parameterObject, rowBounds, resultHandler, key, boundSql);
  }
```

当开启了二级缓存并使用`@CaheNamespace`标注mapper后，mapper中的每个 `MappedStatement` 会有一个cache属性，会进入到 `if(cache!=null)`  的代码逻辑。

首先尝试从缓存中查询：`tcm.getObject(cache, key)`，tcm 是 TransactionCacheManager，即带有事务性质的缓存管理器，管理的是 TransactionCache，TransactionCache 最底层仍是上节提到的PerpetualCache，采用HashMap实现。这些都可以在 `@CaheNamespace` 中自定义。

缓存中没查询到的话，会走数据库查询，然后将结果放到缓存中 `tcm.putObject(cache, key, list)`，最终实现如下：

```java
@Override
  public void putObject(Object key, Object object) {
    entriesToAddOnCommit.put(key, object);
  }
```

```java
private final Map<Object, Object> entriesToAddOnCommit;
```

从名字可以看出来：当commit时要提交的项。

可以看到，`tcm.putObject` 只是将结果放到了 `TransactionCache` 中的 `entriesToAddOnCommit` 里面，而查询逻辑 `tcm.getObject(cache, key)` 是 

```java
@Override
public Object getObject(Object key) {
  // issue #116
  Object object = delegate.getObject(key);
  if (object == null) {
    entriesMissedInCache.add(key);
  }
  // issue #146
  if (clearOnCommit) {
    return null;
  } else {
    return object;
  }
}
```

可看到 getObject 是委托到给了 delegate 去做，不用管 delegate 具体是什么，从最底层的PerpetualCache到 TransactionCache 经历了层层包装(装饰器模式)。上文讲到 putObject 只是把结果放到了 entriesToAddOnCommit 里，而 getObject 根本就没有查询 entriesToAddOnCommit，那这个缓存到底是怎么用的呢？

从  sqlSession.commit 看起，最终会进入到 TransactionCache 的 commit 方法：

```java
public void commit() {
  if (clearOnCommit) {
    delegate.clear();
  }
  flushPendingEntries();
  reset();
}
```

接着看 `flushPendingEntries`：

```java
private void flushPendingEntries() {
  for (Map.Entry<Object, Object> entry : entriesToAddOnCommit.entrySet()) {
    delegate.putObject(entry.getKey(), entry.getValue());
  }
  for (Object entry : entriesMissedInCache) {
    if (!entriesToAddOnCommit.containsKey(entry)) {
      delegate.putObject(entry, null);
    }
  }
}
```

从名字也可以看出来，flush pending entries，这里的pendingEntries 就是 entriesToAddOnCommit。

`delegate.putObject(entry.getKey(), entry.getValue())` ：将entriesToAddOnCommit 添加到 delegate 中去。

**所以，只有当 commit 时，才会将查询结果缓存起来，后续才能利用到缓存。**

增删改操作同样会刷新缓存，也就是 `delegate.clear()`，将delegate中的缓存清空。