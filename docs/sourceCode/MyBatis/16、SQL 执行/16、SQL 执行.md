# 1、Executor

## 1. 概述

从本文开始，我们来分享 SQL **执行**的流程。在 [《精尽 MyBatis 源码分析 —— 项目结构一览》](http://svip.iocoder.cn/MyBatis/intro) 中，我们简单介绍这个流程如下：

> 对应 `executor` 和 `cursor` 模块。前者对应**执行器**，后者对应执行**结果的游标**。
>
> SQL 语句的执行涉及多个组件 ，其中比较重要的是 Executor、StatementHandler、ParameterHandler 和 ResultSetHandler 。
>
> - **Executor** 主要负责维护一级缓存和二级缓存，并提供事务管理的相关操作，它会将数据库相关操作委托给 StatementHandler完成。
> - **StatementHandler** 首先通过 **ParameterHandler** 完成 SQL 语句的实参绑定，然后通过 `java.sql.Statement` 对象执行 SQL 语句并得到结果集，最后通过 **ResultSetHandler** 完成结果集的映射，得到结果对象并返回。
>
> 整体过程如下图：
>
> [![整体过程](http://static.iocoder.cn/images/MyBatis/2020_01_04/05.png)](http://static.iocoder.cn/images/MyBatis/2020_01_04/05.png)整体过程

下面，我们在看看 `executor` 包下的列情况，如下图所示：[![`executor` 包](http://static.iocoder.cn/images/MyBatis/2020_02_28/01.png)](http://static.iocoder.cn/images/MyBatis/2020_02_28/01.png)`executor` 包

- 正如该包下的分包情况，每个包对应一个功能。

- `statement` 包，实现向数据库发起 SQL 命令。

- ```
  parameter
  ```

   

  包，实现设置 PreparedStatement 的占位符参数。

  - 目前只有一个 ParameterHandler 接口，在 [《精尽 MyBatis 源码分析 —— SQL 初始化（下）之 SqlSource》](http://svip.iocoder.cn/MyBatis/scripting-2) 已经详细解析。

- `keygen` 包，实现数据库主键生成( 获得 )的功能。

- `resultset` 包，实现 ResultSet 结果集的处理，将其映射成对应的结果对象。

- `result` 包，结果的处理，被 `resultset` 包所调用。可能胖友会好奇为啥会有 `resultset` 和 `result` 两个“重叠”的包。答案见 [《精尽 MyBatis 源码分析 —— SQL 执行（四）之 ResultSetHandler》](http://svip.iocoder.cn/MyBatis/executor-4) 。

- `loader` 包，实现延迟加载的功能。

- 根目录，Executor 接口及其实现类，作为 SQL 执行的核心入口。

考虑到整个 `executor` 包的代码量近 5000 行，所以我们将每一个子包，作为一篇文章，逐包解析。所以，本文我们先来分享 根目录，也就是 Executor 接口及其实现类。

## 2. Executor

`org.apache.ibatis.executor.Executor` ，执行器接口。代码如下：

```
// Executor.java

public interface Executor {

    // 空 ResultHandler 对象的枚举
    ResultHandler NO_RESULT_HANDLER = null;

    // 更新 or 插入 or 删除，由传入的 MappedStatement 的 SQL 所决定
    int update(MappedStatement ms, Object parameter) throws SQLException;

    // 查询，带 ResultHandler + CacheKey + BoundSql
    <E> List<E> query(MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler, CacheKey cacheKey, BoundSql boundSql) throws SQLException;
    // 查询，带 ResultHandler
    <E> List<E> query(MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler) throws SQLException;
    // 查询，返回值为 Cursor
    <E> Cursor<E> queryCursor(MappedStatement ms, Object parameter, RowBounds rowBounds) throws SQLException;

    // 刷入批处理语句
    List<BatchResult> flushStatements() throws SQLException;

    // 提交事务
    void commit(boolean required) throws SQLException;
    // 回滚事务
    void rollback(boolean required) throws SQLException;

    // 创建 CacheKey 对象
    CacheKey createCacheKey(MappedStatement ms, Object parameterObject, RowBounds rowBounds, BoundSql boundSql);
    // 判断是否缓存
    boolean isCached(MappedStatement ms, CacheKey key);
    // 清除本地缓存
    void clearLocalCache();

    // 延迟加载
    void deferLoad(MappedStatement ms, MetaObject resultObject, String property, CacheKey key, Class<?> targetType);

    // 获得事务
    Transaction getTransaction();
    // 关闭事务
    void close(boolean forceRollback);
    // 判断事务是否关闭
    boolean isClosed();

    // 设置包装的 Executor 对象
    void setExecutorWrapper(Executor executor);

}
```

- 读和写操作相关的方法
- 事务相关的方法
- 缓存相关的方法
- 设置延迟加载的方法
- 设置包装的 Executor 对象的方法

------

Executor 的实现类如下图所示：[![Executor 类图](http://static.iocoder.cn/images/MyBatis/2020_02_28/02.png)](http://static.iocoder.cn/images/MyBatis/2020_02_28/02.png)Executor 类图

- 我们可以看到，Executor 的直接子类有 BaseExecutor 和 CachingExecutor 两个。
- 实际上，CachingExecutor 在 BaseExecutor 的基础上，实现**二级缓存**功能。
- 在下文中，BaseExecutor 的**本地**缓存，就是**一级**缓存。

下面，我们按照先看 BaseExecutor 侧的实现类的源码解析，再看 CachingExecutor 的。

## 3. BaseExecutor

`org.apache.ibatis.executor.BaseExecutor` ，实现 Executor 接口，提供骨架方法，从而使子类只要实现指定的几个抽象方法即可。

#### 3.1 构造方法

```
// BaseExecutor.java

/**
 * 事务对象
 */
protected Transaction transaction;
/**
 * 包装的 Executor 对象
 */
protected Executor wrapper;

/**
 * DeferredLoad( 延迟加载 ) 队列
 */
protected ConcurrentLinkedQueue<DeferredLoad> deferredLoads;
/**
 * 本地缓存，即一级缓存
 */
protected PerpetualCache localCache;
/**
 * 本地输出类型的参数的缓存
 */
protected PerpetualCache localOutputParameterCache;
protected Configuration configuration;

/**
 * 记录嵌套查询的层级
 */
protected int queryStack;
/**
 * 是否关闭
 */
private boolean closed;

protected BaseExecutor(Configuration configuration, Transaction transaction) {
    this.transaction = transaction;
    this.deferredLoads = new ConcurrentLinkedQueue<>();
    this.localCache = new PerpetualCache("LocalCache");
    this.localOutputParameterCache = new PerpetualCache("LocalOutputParameterCache");
    this.closed = false;
    this.configuration = configuration;
    this.wrapper = this; // 自己
}
```

- 和延迟加载相关，后续文章，详细解析。详细解析，见

   

  《精尽 MyBatis 源码分析 —— SQL 执行（五）之延迟加载》

   

  。

  - `queryStack` 属性，记录**递归**嵌套查询的层级。
  - `deferredLoads` 属性，DeferredLoad( 延迟加载 ) 队列。

- `wrapper` 属性，在构造方法中，初始化为 `this` ，即自己。

- `localCache` 属性，本地缓存，即**一级缓存**。那什么是一级缓存呢？

  > 基于 [《MyBatis 的一级缓存实现详解及使用注意事项》](https://blog.csdn.net/luanlouis/article/details/41280959) 进行修改
  >
  > 每当我们使用 MyBatis 开启一次和数据库的会话，MyBatis 会创建出一个 SqlSession 对象表示一次数据库会话，**而每个 SqlSession 都会创建一个 Executor 对象**。
  >
  > 在对数据库的一次会话中，我们有可能会反复地执行完全相同的查询语句，如果不采取一些措施的话，每一次查询都会查询一次数据库，而我们在极短的时间内做了完全相同的查询，那么它们的结果极有可能完全相同，由于查询一次数据库的代价很大，这有可能造成很大的资源浪费。
  >
  > 为了解决这一问题，减少资源的浪费，MyBatis 会在表示会话的SqlSession 对象中建立一个简单的缓存，将每次查询到的结果结果缓存起来，当下次查询的时候，如果判断先前有个完全一样的查询，会直接从缓存中直接将结果取出，返回给用户，不需要再进行一次数据库查询了。😈 **注意，这个“简单的缓存”就是一级缓存，且默认开启，无法关闭**。
  >
  > 如下图所示，MyBatis 会在一次会话的表示 —— 一个 SqlSession 对象中创建一个本地缓存( `localCache` )，对于每一次查询，都会尝试根据查询的条件去本地缓存中查找是否在缓存中，如果在缓存中，就直接从缓存中取出，然后返回给用户；否则，从数据库读取数据，将查询结果存入缓存并返回给用户。
  >
  > [![整体过程](http://static.iocoder.cn/images/MyBatis/2020_02_28/03.png)](http://static.iocoder.cn/images/MyBatis/2020_02_28/03.png)整体过程
  >
  > - 关于这段话，胖友要理解 SqlSession 和 Executor 和一级缓存的关系。
  > - 😈 另外，下文，我们还会介绍**二级缓存**是什么。

- `transaction` 属性，事务对象。该属性，是通过构造方法传入。为什么呢？待我们看 `org.apache.ibatis.session.session` 包。

#### 3.2 clearLocalCache

`#clearLocalCache()` 方法，清理一级（本地）缓存。代码如下：

```
// BaseExecutor.java

@Override
public void clearLocalCache() {
    if (!closed) {
        // 清理 localCache
        localCache.clear();
        // 清理 localOutputParameterCache
        localOutputParameterCache.clear();
    }
}
```

#### 3.3 createCacheKey

`#createCacheKey(MappedStatement ms, Object parameterObject, RowBounds rowBounds, BoundSql boundSql)` 方法，创建 CacheKey 对象。代码如下：

```
// BaseExecutor.java

@Override
public CacheKey createCacheKey(MappedStatement ms, Object parameterObject, RowBounds rowBounds, BoundSql boundSql) {
    if (closed) {
        throw new ExecutorException("Executor was closed.");
    }
    // <1> 创建 CacheKey 对象
    CacheKey cacheKey = new CacheKey();
    // <2> 设置 id、offset、limit、sql 到 CacheKey 对象中
    cacheKey.update(ms.getId());
    cacheKey.update(rowBounds.getOffset());
    cacheKey.update(rowBounds.getLimit());
    cacheKey.update(boundSql.getSql());
    // <3> 设置 ParameterMapping 数组的元素对应的每个 value 到 CacheKey 对象中
    List<ParameterMapping> parameterMappings = boundSql.getParameterMappings();
    TypeHandlerRegistry typeHandlerRegistry = ms.getConfiguration().getTypeHandlerRegistry();
    // mimic DefaultParameterHandler logic 这块逻辑，和 DefaultParameterHandler 获取 value 是一致的。
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
    // <4> 设置 Environment.id 到 CacheKey 对象中
    if (configuration.getEnvironment() != null) {
        // issue #176
        cacheKey.update(configuration.getEnvironment().getId());
    }
    return cacheKey;
}
```

- `<1>` 处，创建 CacheKey 对象。关于 CacheKey 类，在 [《精尽 MyBatis 源码分析 —— 缓存模块》](http://svip.iocoder.cn/MyBatis/cache-package) 已经详细解析。
- `<2>` 处，设置 `id`、`offset`、`limit`、`sql` 到 CacheKey 对象中。
- `<3>` 处，设置 ParameterMapping 数组的元素对应的每个 `value` 到 CacheKey 对象中。注意，这块逻辑，和 DefaultParameterHandler 获取 `value` 是一致的。
- `<4>` 处，设置 `Environment.id` 到 CacheKey 对象中。

#### 3.4 isCached

`#isCached(MappedStatement ms, CacheKey key)` 方法，判断一级缓存是否存在。代码如下：

```
// BaseExecutor.java

@Override
public boolean isCached(MappedStatement ms, CacheKey key) {
    return localCache.getObject(key) != null;
}
```

#### 3.5 query

① `#query(MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler)` 方法，读操作。代码如下：

```
// BaseExecutor.java

@Override
public <E> List<E> query(MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler) throws SQLException {
    // <1> 获得 BoundSql 对象
    BoundSql boundSql = ms.getBoundSql(parameter);
    // <2> 创建 CacheKey 对象
    CacheKey key = createCacheKey(ms, parameter, rowBounds, boundSql);
    // <3> 查询
    return query(ms, parameter, rowBounds, resultHandler, key, boundSql);
}
```

- `<1>` 处，调用 `MappedStatement#getBoundSql(Object parameterObject)` 方法，获得 BoundSql 对象。
- `<2>` 处，调用 `#createCacheKey(MappedStatement ms, Object parameterObject, RowBounds rowBounds, BoundSql boundSql)` 方法，创建 CacheKey 对象。
- `<3>` 处，调用 `#query(MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler, CacheKey key, BoundSql boundSql)` 方法，读操作。通过这样的方式，两个 `#query(...)` 方法，实际是统一的。

② `#query(MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler, CacheKey key, BoundSql boundSql)` 方法，读操作。代码如下：

```
// BaseExecutor.java

@Override
public <E> List<E> query(MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler, CacheKey key, BoundSql boundSql) throws SQLException {
    ErrorContext.instance().resource(ms.getResource()).activity("executing a query").object(ms.getId());
    // <1> 已经关闭，则抛出 ExecutorException 异常
    if (closed) {
        throw new ExecutorException("Executor was closed.");
    }
    // <2> 清空本地缓存，如果 queryStack 为零，并且要求清空本地缓存。
    if (queryStack == 0 && ms.isFlushCacheRequired()) {
        clearLocalCache();
    }
    List<E> list;
    try {
        // <3> queryStack + 1
        queryStack++;
        // <4.1> 从一级缓存中，获取查询结果
        list = resultHandler == null ? (List<E>) localCache.getObject(key) : null;
        // <4.2> 获取到，则进行处理
        if (list != null) {
            handleLocallyCachedOutputParameters(ms, key, parameter, boundSql);
        // <4.3> 获得不到，则从数据库中查询
        } else {
            list = queryFromDatabase(ms, parameter, rowBounds, resultHandler, key, boundSql);
        }
    } finally {
        // <5> queryStack - 1
        queryStack--;
    }
    if (queryStack == 0) {
        // <6.1> 执行延迟加载
        for (DeferredLoad deferredLoad : deferredLoads) {
            deferredLoad.load();
        }
        // issue #601
        // <6.2> 清空 deferredLoads
        deferredLoads.clear();
        // <7> 如果缓存级别是 LocalCacheScope.STATEMENT ，则进行清理
        if (configuration.getLocalCacheScope() == LocalCacheScope.STATEMENT) {
            // issue #482
            clearLocalCache();
        }
    }
    return list;
}
```

- `<1>` 处，已经关闭，则抛出 ExecutorException 异常。

- `<2>` 处，调用 `#clearLocalCache()` 方法，清空本地缓存，如果 `queryStack` 为零，并且要求清空本地缓存。例如：`<select flushCache="true"> ... </a>` 。

- `<3>` 处，`queryStack` + 1 。

- ```
  <4.1>
  ```

   

  处，从一级缓存

   

  ```
  localCache
  ```

   

  中，获取查询结果。

  - `<4.2>` 处，获取到，则进行处理。对于 `#handleLocallyCachedOutputParameters(MappedStatement ms, CacheKey key, Object parameter, BoundSql boundSql)` 方法，是处理存储过程的情况，所以我们就忽略。
  - `<4.3>` 处，获得不到，则调用 `#queryFromDatabase()` 方法，从数据库中查询。详细解析，见 [「3.5.1 queryFromDatabase」](http://svip.iocoder.cn/MyBatis/executor-1/#) 。

- `<5>` 处，`queryStack` - 1 。

- ```
  <6.1>
  ```

   

  处，遍历 DeferredLoad 队列，逐个调用

   

  ```
  DeferredLoad#load()
  ```

   

  方法，执行延迟加载。详细解析，见

   

  《精尽 MyBatis 源码分析 —— SQL 执行（五）之延迟加载》

   

  。

  - `<6.2>` 处，清空 DeferredLoad 队列。

- `<7>` 处，如果缓存级别是 `LocalCacheScope.STATEMENT` ，则调用 `#clearLocalCache()` 方法，清空本地缓存。默认情况下，缓存级别是 `LocalCacheScope.SESSION` 。代码如下：

  ```
  // Configuration.java
  
  /**
   * {@link BaseExecutor} 本地缓存范围
   */
  protected LocalCacheScope localCacheScope = LocalCacheScope.SESSION;
  
  // LocalCacheScope.java
  
  public enum LocalCacheScope {
  
      /**
       * 会话级
       */
      SESSION,
      /**
       * SQL 语句级
       */
      STATEMENT
  
  }
  ```

###### 3.5.1 queryFromDatabase

`#queryFromDatabase(MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler, CacheKey key, BoundSql boundSql)` 方法，从数据库中读取操作。代码如下：

```
// BaseExecutor.java

private <E> List<E> queryFromDatabase(MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler, CacheKey key, BoundSql boundSql) throws SQLException {
    List<E> list;
    // <1> 在缓存中，添加占位对象。此处的占位符，和延迟加载有关，可见 `DeferredLoad#canLoad()` 方法
    localCache.putObject(key, EXECUTION_PLACEHOLDER);
    try {
        // <2> 执行读操作
        list = doQuery(ms, parameter, rowBounds, resultHandler, boundSql);
    } finally {
        // <3> 从缓存中，移除占位对象
        localCache.removeObject(key);
    }
    // <4> 添加到缓存中
    localCache.putObject(key, list);
    // <5> 暂时忽略，存储过程相关
    if (ms.getStatementType() == StatementType.CALLABLE) {
        localOutputParameterCache.putObject(key, parameter);
    }
    return list;
}
```

- `<1>` 处，在缓存中，添加**占位对象**。此处的占位符，和延迟加载有关，后续可见 `DeferredLoad#canLoad()` 方法。

  ```
  // BaseExecutor.java
  
  public enum ExecutionPlaceholder {
  
      /**
       * 正在执行中的占位符
       */
      EXECUTION_PLACEHOLDER
  
  }
  ```

- `<2>` 处，调用 `#doQuery(MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler, BoundSql boundSql)` 执行读操作。这是个抽象方法，由子类实现。

- `<3>` 处，从缓存中，移除占位对象。

- `<4>` 处，添加**结果**到缓存中。

- `<5>` 处，暂时忽略，存储过程相关。

###### 3.5.2 doQuery

```
// BaseExecutor.java

protected abstract <E> List<E> doQuery(MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler, BoundSql boundSql)
        throws SQLException;
```

#### 3.6 queryCursor

`#queryCursor(MappedStatement ms, Object parameter, RowBounds rowBounds)` 方法，执行查询，返回的结果为 Cursor 游标对象。代码如下：

```
// BaseExecutor.java

@Override
public <E> Cursor<E> queryCursor(MappedStatement ms, Object parameter, RowBounds rowBounds) throws SQLException {
    // <1> 获得 BoundSql 对象
    BoundSql boundSql = ms.getBoundSql(parameter);
    // 执行查询
    return doQueryCursor(ms, parameter, rowBounds, boundSql);
}
```

- `<1>` 处，调用 `MappedStatement#getBoundSql(Object parameterObject)` 方法，获得 BoundSql 对象。
- `<2>` 处，调用 `#doQueryCursor(MappedStatement ms, Object parameter, RowBounds rowBounds, BoundSql boundSql)` 方法，执行读操作。这是个抽象方法，由子类实现。

###### 3.6.1 doQueryCursor

```
// BaseExecutor.java

protected abstract <E> Cursor<E> doQueryCursor(MappedStatement ms, Object parameter, RowBounds rowBounds, BoundSql boundSql)
        throws SQLException;
```

#### 3.7 update

`#update(MappedStatement ms, Object parameter)` 方法，执行写操作。代码如下：

```
// BaseExecutor.java

@Override
public int update(MappedStatement ms, Object parameter) throws SQLException {
    ErrorContext.instance().resource(ms.getResource()).activity("executing an update").object(ms.getId());
    // <1> 已经关闭，则抛出 ExecutorException 异常
    if (closed) {
        throw new ExecutorException("Executor was closed.");
    }
    // <2> 清空本地缓存
    clearLocalCache();
    // <3> 执行写操作
    return doUpdate(ms, parameter);
}
```

- `<1>` 处，已经关闭，则抛出 ExecutorException 异常。
- `<2>` 处，调用 `#clearLocalCache()` 方法，清空本地缓存。因为，更新后，可能缓存会失效。但是，又没很好的办法，判断哪一些失效。所以，最稳妥的做法，就是全部清空。
- `<3>` 处，调用 `#doUpdate(MappedStatement ms, Object parameter)` 方法，执行写操作。这是个抽象方法，由子类实现。

###### 3.7.1 doUpdate

```
// BaseExecutor.java

protected abstract int doUpdate(MappedStatement ms, Object parameter)
        throws SQLException;
```

#### 3.8 flushStatements

`#flushStatements()` 方法，刷入批处理语句。代码如下：

```
// BaseExecutor.java

@Override
public List<BatchResult> flushStatements() throws SQLException {
    return flushStatements(false);
}

public List<BatchResult> flushStatements(boolean isRollBack) throws SQLException {
    // <1> 已经关闭，则抛出 ExecutorException 异常
    if (closed) {
        throw new ExecutorException("Executor was closed.");
    }
    // <2> 执行刷入批处理语句
    return doFlushStatements(isRollBack);
}
```

- `isRollBack` 属性，目前看下来没什么逻辑。唯一看到在 BatchExecutor 中，如果 `isRollBack = true` ，则**不执行**刷入批处理语句。有点奇怪。
- `<1>` 处，已经关闭，则抛出 ExecutorException 异常。
- `<2>` 处，调用 `#doFlushStatements(boolean isRollback)` 方法，执行刷入批处理语句。这是个抽象方法，由子类实现。

###### 3.8.1 doFlushStatements

```
// BaseExecutor.java

protected abstract List<BatchResult> doFlushStatements(boolean isRollback)
        throws SQLException;
```

------

至此，我们已经看到了 BaseExecutor 所定义的**四个抽象方法**：

- 「3.5.2 doQuery」
- 「3.6.1 doQueryCursor」
- 「3.7.1 doUpdate」
- 「3.8.1 doFlushStatements」

#### 3.9 getTransaction

`#getTransaction()` 方法，获得事务对象。代码如下：

```
// BaseExecutor.java

@Override
public Transaction getTransaction() {
    if (closed) {
        throw new ExecutorException("Executor was closed.");
    }
    return transaction;
}
```

###### 3.9.1 commit

```
// BaseExecutor.java

@Override
public void commit(boolean required) throws SQLException {
    // 已经关闭，则抛出 ExecutorException 异常
    if (closed) {
        throw new ExecutorException("Cannot commit, transaction is already closed");
    }
    // 清空本地缓存
    clearLocalCache();
    // 刷入批处理语句
    flushStatements();
    // 是否要求提交事务。如果是，则提交事务。
    if (required) {
        transaction.commit();
    }
}
```

###### 3.9.2 rollback

```
// BaseExecutor.java

public void rollback(boolean required) throws SQLException {
    if (!closed) {
        try {
            // 清空本地缓存
            clearLocalCache();
            // 刷入批处理语句
            flushStatements(true);
        } finally {
            if (required) {
                // 是否要求回滚事务。如果是，则回滚事务。
                transaction.rollback();
            }
        }
    }
}
```

#### 3.10 close

`#close()` 方法，关闭执行器。代码如下：

```
// BaseExecutor.java

@Override
public void close(boolean forceRollback) {
    try {
        // 回滚事务
        try {
            rollback(forceRollback);
        } finally {
            // 关闭事务
            if (transaction != null) {
                transaction.close();
            }
        }
    } catch (SQLException e) {
        // Ignore.  There's nothing that can be done at this point.
        log.warn("Unexpected exception on closing transaction.  Cause: " + e);
    } finally {
        // 置空变量
        transaction = null;
        deferredLoads = null;
        localCache = null;
        localOutputParameterCache = null;
        closed = true;
    }
}
```

###### 3.10.1 isClosed

```
// BaseExecutor.java

@Override
public boolean isClosed() {
    return closed;
}
```

#### 3.11 deferLoad

详细解析，见 [《精尽 MyBatis 源码分析 —— SQL 执行（五）之延迟加载》](http://svip.iocoder.cn/MyBatis/executor-5) 。

#### 3.12 setExecutorWrapper

`#setExecutorWrapper(Executor wrapper)` 方法，设置包装器。代码如下：

```
// BaseExecutor.java

@Override
public void setExecutorWrapper(Executor wrapper) {
    this.wrapper = wrapper;
}
```

#### 3.13 其它方法

```
// BaseExecutor.java

// 获得 Connection 对象
protected Connection getConnection(Log statementLog) throws SQLException {
    // 获得 Connection 对象
    Connection connection = transaction.getConnection();
    // 如果 debug 日志级别，则创建 ConnectionLogger 对象，进行动态代理
    if (statementLog.isDebugEnabled()) {
        return ConnectionLogger.newInstance(connection, statementLog, queryStack);
    } else {
        return connection;
    }
}

// 设置事务超时时间
protected void applyTransactionTimeout(Statement statement) throws SQLException {
    StatementUtil.applyTransactionTimeout(statement, statement.getQueryTimeout(), transaction.getTimeout());
}

// 关闭 Statement 对象
protected void closeStatement(Statement statement) {
    if (statement != null) {
        try {
            if (!statement.isClosed()) {
                statement.close();
            }
        } catch (SQLException e) {
            // ignore
        }
    }
}
```

- 关于 StatementUtil 类，可以点击 [`org.apache.ibatis.executor.statement.StatementUtil`](https://github.com/YunaiV/mybatis-3/blob/master/src/main/java/org/apache/ibatis/executor/statement/StatementUtil.java) 查看。

## 4. SimpleExecutor

`org.apache.ibatis.executor.SimpleExecutor` ，继承 BaseExecutor 抽象类，简单的 Executor 实现类。

- 每次开始读或写操作，都创建对应的 Statement 对象。
- 执行完成后，关闭该 Statement 对象。

#### 4.1 构造方法

```
// SimpleExecutor.java

public SimpleExecutor(Configuration configuration, Transaction transaction) {
    super(configuration, transaction);
}
```

#### 4.2 doQuery

> 老艿艿：从此处开始，我们会看到 StatementHandler 相关的调用，胖友可以结合 [《精尽 MyBatis 源码分析 —— SQL 执行（二）之 StatementHandler》](http://svip.iocoder.cn/MyBatis/executor-2) 一起看。

```
// SimpleExecutor.java

@Override
public <E> List<E> doQuery(MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler, BoundSql boundSql) throws SQLException {
    Statement stmt = null;
    try {
        Configuration configuration = ms.getConfiguration();
        // <1> 创建 StatementHandler 对象
        StatementHandler handler = configuration.newStatementHandler(wrapper, ms, parameter, rowBounds, resultHandler, boundSql);
        // <2> 初始化 StatementHandler 对象
        stmt = prepareStatement(handler, ms.getStatementLog());
        // <3> 执行 StatementHandler  ，进行读操作
        return handler.query(stmt, resultHandler);
    } finally {
        // <4> 关闭 StatementHandler 对象
        closeStatement(stmt);
    }
}
```

- `<1>` 处，调用 `Configuration#newStatementHandler(Executor executor, MappedStatement mappedStatement, Object parameterObject, RowBounds rowBounds, ResultHandler resultHandler, BoundSql boundSql)` 方法，创建 StatementHandler 对象。

- `<2>` 处，调用 `#prepareStatement(StatementHandler handler, Log statementLog)` 方法，初始化 StatementHandler 对象。代码如下：

  ```
  // SimpleExecutor.java
  
  private Statement prepareStatement(StatementHandler handler, Log statementLog) throws SQLException {
      Statement stmt;
      // <2.1> 获得 Connection 对象
      Connection connection = getConnection(statementLog);
      // <2.2> 创建 Statement 或 PrepareStatement 对象
      stmt = handler.prepare(connection, transaction.getTimeout());
      // <2.3> 设置 SQL 上的参数，例如 PrepareStatement 对象上的占位符
      handler.parameterize(stmt);
      return stmt;
  }
  ```

  - `<2.1>` 处，调用 `#getConnection(Log statementLog)` 方法，获得 Connection 对象。
  - `<2.2>` 处，调用 `StatementHandler#prepare(Connection connection, Integer transactionTimeout)` 方法，创建 Statement 或 PrepareStatement 对象。
  - `<2.3>` 处，调用 `StatementHandler#prepare(Statement statement)` 方法，设置 SQL 上的参数，例如 PrepareStatement 对象上的占位符。

- `<3>` 处，调用 `StatementHandler#query(Statement statement, ResultHandler resultHandler)` 方法，进行**读操作**。

- `<4>` 处，调用 `#closeStatement(Statement stmt)` 方法，关闭 StatementHandler 对象。

#### 4.3 doQueryCursor

```
// SimpleExecutor.java

@Override
protected <E> Cursor<E> doQueryCursor(MappedStatement ms, Object parameter, RowBounds rowBounds, BoundSql boundSql) throws SQLException {
    Configuration configuration = ms.getConfiguration();
    // 创建 StatementHandler 对象
    StatementHandler handler = configuration.newStatementHandler(wrapper, ms, parameter, rowBounds, null, boundSql);
    // 初始化 StatementHandler 对象
    Statement stmt = prepareStatement(handler, ms.getStatementLog());
    // 设置 Statement ，如果执行完成，则进行自动关闭
    stmt.closeOnCompletion();
    // 执行 StatementHandler  ，进行读操作
    return handler.queryCursor(stmt);
}
```

- 和 `#doQuery(...)` 方法的思路是一致的，胖友自己看下。

#### 4.4 doUpdate

```
// SimpleExecutor.java

@Override
public int doUpdate(MappedStatement ms, Object parameter) throws SQLException {
    Statement stmt = null;
    try {
        Configuration configuration = ms.getConfiguration();
        // 创建 StatementHandler 对象
        StatementHandler handler = configuration.newStatementHandler(this, ms, parameter, RowBounds.DEFAULT, null, null);
        // 初始化 StatementHandler 对象
        stmt = prepareStatement(handler, ms.getStatementLog());
        // <3> 执行 StatementHandler ，进行写操作
        return handler.update(stmt);
    } finally {
        // 关闭 StatementHandler 对象
        closeStatement(stmt);
    }
}
```

- 相比 `#doQuery(...)` 方法，差异点在 `<3>` 处，换成了调用 `StatementHandler#update(Statement statement)` 方法，进行**写操作**。

#### 4.5 doFlushStatements

```
// SimpleExecutor.java

@Override
public List<BatchResult> doFlushStatements(boolean isRollback) throws SQLException {
    return Collections.emptyList();
}
```

- 不存在批量操作的情况，所以直接返回空数组。

## 5. ReuseExecutor

`org.apache.ibatis.executor.ReuseExecutor` ，继承 BaseExecutor 抽象类，可重用的 Executor 实现类。

- 每次开始读或写操作，优先从缓存中获取对应的 Statement 对象。如果不存在，才进行创建。
- 执行完成后，不关闭该 Statement 对象。
- 其它的，和 SimpleExecutor 是一致的。

#### 5.1 构造方法

```
// ReuseExecutor.java

/**
 * Statement 的缓存
 *
 * KEY ：SQL
 */
private final Map<String, Statement> statementMap = new HashMap<>();

public ReuseExecutor(Configuration configuration, Transaction transaction) {
    super(configuration, transaction);
}
```

#### 5.2 doQuery

```
// ReuseExecutor.java

@Override
public <E> List<E> doQuery(MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler, BoundSql boundSql) throws SQLException {
    Configuration configuration = ms.getConfiguration();
    // 创建 StatementHandler 对象
    StatementHandler handler = configuration.newStatementHandler(wrapper, ms, parameter, rowBounds, resultHandler, boundSql);
    // <1> 初始化 StatementHandler 对象
    Statement stmt = prepareStatement(handler, ms.getStatementLog());
    // 执行 StatementHandler  ，进行读操作
    return handler.query(stmt, resultHandler);
}
```

- 差异一，在于 `<1>` 处，调用 `#prepareStatement(StatementHandler handler, Log statementLog)` 方法，初始化 StatementHandler 对象。代码如下：

  ```
  // ReuseExecutor.java
  
  private Statement prepareStatement(StatementHandler handler, Log statementLog) throws SQLException {
      Statement stmt;
      BoundSql boundSql = handler.getBoundSql();
      String sql = boundSql.getSql();
      // 存在
      if (hasStatementFor(sql)) {
          // <1.1> 从缓存中获得 Statement 或 PrepareStatement 对象
          stmt = getStatement(sql);
          // <1.2> 设置事务超时时间
          applyTransactionTimeout(stmt);
      // 不存在
      } else {
          // <2.1> 获得 Connection 对象
          Connection connection = getConnection(statementLog);
          // <2.2> 创建 Statement 或 PrepareStatement 对象
          stmt = handler.prepare(connection, transaction.getTimeout());
          // <2.3> 添加到缓存中
          putStatement(sql, stmt);
      }
      // <2> 设置 SQL 上的参数，例如 PrepareStatement 对象上的占位符
      handler.parameterize(stmt);
      return stmt;
  }
  ```

  - 调用 `#hasStatementFor(String sql)` 方法，判断是否存在对应的 Statement 对象。代码如下：

    ```
    // ReuseExecutor.java
    
    private boolean hasStatementFor(String sql) {
        try {
            return statementMap.keySet().contains(sql) && !statementMap.get(sql).getConnection().isClosed();
        } catch (SQLException e) {
            return false;
        }
    }
    ```

    - 并且，要求连接**未关闭**。

  - 存在

    - `<1.1>` 处，调用 `#getStatement(String s)` 方法，获得 Statement 对象。代码如下：

      ```
      // ReuseExecutor.java
      
      private Statement getStatement(String s) {
          return statementMap.get(s);
      }
      ```

      - x

    - 【差异】`<1.2>` 处，调用 `#applyTransactionTimeout(Statement stmt)` 方法，设置事务超时时间。

  - 不存在

    - `<2.1>` 处，获得 Connection 对象。

    - `<2.2>` 处，调用 `StatementHandler#prepare(Connection connection, Integer transactionTimeout)` 方法，创建 Statement 或 PrepareStatement 对象。

    - 【差异】`<2.3>` 处，调用 `#putStatement(String sql, Statement stmt)` 方法，添加 Statement 对象到缓存中。代码如下：

      ```
      // ReuseExecutor.java
      
      private void putStatement(String sql, Statement stmt) {
          statementMap.put(sql, stmt);
      }
      ```

      - x

  - `<2>` 处，调用 `StatementHandler#prepare(Statement statement)` 方法，设置 SQL 上的参数，例如 PrepareStatement 对象上的占位符。

- 差异二，在执行完数据库操作后，不会关闭 Statement 。

#### 5.3 doQueryCursor

```
// ReuseExecutor.java

@Override
protected <E> Cursor<E> doQueryCursor(MappedStatement ms, Object parameter, RowBounds rowBounds, BoundSql boundSql) throws SQLException {
    Configuration configuration = ms.getConfiguration();
    // 创建 StatementHandler 对象
    StatementHandler handler = configuration.newStatementHandler(wrapper, ms, parameter, rowBounds, null, boundSql);
    // 初始化 StatementHandler 对象
    Statement stmt = prepareStatement(handler, ms.getStatementLog());
    // 执行 StatementHandler  ，进行读操作
    return handler.queryCursor(stmt);
}
```

#### 5.4 doUpdate

```
// ReuseExecutor.java

@Override
public int doUpdate(MappedStatement ms, Object parameter) throws SQLException {
    Configuration configuration = ms.getConfiguration();
    // 创建 StatementHandler 对象
    StatementHandler handler = configuration.newStatementHandler(this, ms, parameter, RowBounds.DEFAULT, null, null);
    // 初始化 StatementHandler 对象
    Statement stmt = prepareStatement(handler, ms.getStatementLog());
    // 执行 StatementHandler  ，进行写操作
    return handler.update(stmt);
}
```

#### 5.5 doFlushStatements

```
// ReuseExecutor.java

@Override
public List<BatchResult> doFlushStatements(boolean isRollback) throws SQLException {
    // 关闭缓存的 Statement 对象们
    for (Statement stmt : statementMap.values()) {
        closeStatement(stmt);
    }
    statementMap.clear();
    // 返回空集合
    return Collections.emptyList();
}
```

- ReuseExecutor 考虑到重用性，但是 Statement 最终还是需要有地方关闭。答案就在 `#doFlushStatements(boolean isRollback)` 方法中。而 BaseExecutor 在关闭 `#close()` 方法中，最终也会调用该方法，从而完成关闭缓存的 Statement 对象们
  。
- 另外，BaseExecutor 在提交或者回滚事务方法中，最终也会调用该方法，也能完成关闭缓存的 Statement 对象们。

## 6. BatchExecutor

`org.apache.ibatis.executor.BatchExecutor` ，继承 BaseExecutor 抽象类，批量执行的 Executor 实现类。

> FROM 祖大俊 [《Mybatis3.3.x技术内幕（四）：五鼠闹东京之执行器Executor设计原本》](https://my.oschina.net/zudajun/blog/667214)
>
> 执行update（没有select，JDBC批处理不支持select），将所有sql都添加到批处理中（addBatch()），等待统一执行（executeBatch()），它缓存了多个Statement对象，每个Statement对象都是addBatch()完毕后，等待逐一执行executeBatch()批处理的；BatchExecutor相当于维护了多个桶，每个桶里都装了很多属于自己的SQL，就像苹果蓝里装了很多苹果，番茄蓝里装了很多番茄，最后，再统一倒进仓库。（可以是Statement或PrepareStatement对象）

#### 6.1 构造方法

```
// BatchExecutor.java

/**
 * Statement 数组
 */
private final List<Statement> statementList = new ArrayList<>();
/**
 * BatchResult 数组
 *
 * 每一个 BatchResult 元素，对应一个 {@link #statementList} 的 Statement 元素
 */
private final List<BatchResult> batchResultList = new ArrayList<>();
/**
 * 当前 SQL
 */
private String currentSql;
/**
 * 当前 MappedStatement 对象
 */
private MappedStatement currentStatement;

public BatchExecutor(Configuration configuration, Transaction transaction) {
    super(configuration, transaction);
}
```

- `currentSql` 和 `currentStatement` 属性，当前 SQL 和 MappedStatement 对象。
- `batchResultList` 和 `statementList` 属性，分别是 BatchResult 和 Statement 数组。并且，每一个 `batchResultList` 的 BatchResult 元素，对应一个 `statementList` 的 Statement 元素。
- 具体怎么应用上述属性，我们见 `#doUpdate(...)` 和 `#doFlushStatements(...)` 方法。

#### 6.2 BatchResult

`org.apache.ibatis.executor.BatchResult` ，相同 SQL 聚合的结果。代码如下：

```
// BatchResult.java

/**
 * MappedStatement 对象
 */
private final MappedStatement mappedStatement;
/**
 * SQL
 */
private final String sql;
/**
 * 参数对象集合
 *
 * 每一个元素，对应一次操作的参数
 */
private final List<Object> parameterObjects;
/**
 * 更新数量集合
 *
 * 每一个元素，对应一次操作的更新数量
 */
private int[] updateCounts;

// ... 省略 setting / getting 相关方法
```

#### 6.3 doUpdate

```
// BatchExecutor.java

@Override
public int doUpdate(MappedStatement ms, Object parameterObject) throws SQLException {
    final Configuration configuration = ms.getConfiguration();
    // <1> 创建 StatementHandler 对象
    final StatementHandler handler = configuration.newStatementHandler(this, ms, parameterObject, RowBounds.DEFAULT, null, null);
    final BoundSql boundSql = handler.getBoundSql();
    final String sql = boundSql.getSql();
    final Statement stmt;
    // <2> 如果匹配最后一次 currentSql 和 currentStatement ，则聚合到 BatchResult 中
    if (sql.equals(currentSql) && ms.equals(currentStatement)) {
        // <2.1> 获得最后一次的 Statement 对象
        int last = statementList.size() - 1;
        stmt = statementList.get(last);
        // <2.2> 设置事务超时时间
        applyTransactionTimeout(stmt);
        // <2.3> 设置 SQL 上的参数，例如 PrepareStatement 对象上的占位符
        handler.parameterize(stmt);//fix Issues 322
        // <2.4> 获得最后一次的 BatchResult 对象，并添加参数到其中
        BatchResult batchResult = batchResultList.get(last);
        batchResult.addParameterObject(parameterObject);
    // <3> 如果不匹配最后一次 currentSql 和 currentStatement ，则新建 BatchResult 对象
    } else {
        // <3.1> 获得 Connection
        Connection connection = getConnection(ms.getStatementLog());
        // <3.2> 创建 Statement 或 PrepareStatement 对象
        stmt = handler.prepare(connection, transaction.getTimeout());
        // <3.3> 设置 SQL 上的参数，例如 PrepareStatement 对象上的占位符
        handler.parameterize(stmt);    //fix Issues 322
        // <3.4> 重新设置 currentSql 和 currentStatement
        currentSql = sql;
        currentStatement = ms;
        // <3.5> 添加 Statement 到 statementList 中
        statementList.add(stmt);
        // <3.6> 创建 BatchResult 对象，并添加到 batchResultList 中
        batchResultList.add(new BatchResult(ms, sql, parameterObject));
    }
    // handler.parameterize(stmt);
    // <4> 批处理
    handler.batch(stmt);
    return BATCH_UPDATE_RETURN_VALUE;
}
```

- `<1>` 处，调用 `Configuration#newStatementHandler(...)` 方法，创建 StatementHandler 对象。

- `<2>` 和 `<3>` 处，就是两种不同的情况，差异点在于传入的 `sql` 和 `ms` 是否匹配当前的 `currentSql` 和 `currentStatement` 。如果是，则继续聚合到最后的 BatchResult 中，否则，创建新的 BatchResult 对象，进行“聚合”。

- ```
  <2>
  ```

   

  块：

  - `<2.1>`、`<2.2>`、`<2.3>` 处，逻辑和 ReuseExecutor 相似，使用可重用的 Statement 对象，并进行初始化。
  - `<2.4>` 处，获得最后一次的 BatchResult 对象，并添加参数到其中。

- ```
  <3>
  ```

   

  块：

  - `<3.1>`、`<3.2>`、`<3.3>` 处，逻辑和 SimpleExecutor 相似，创建新的 Statement 对象，并进行初始化。
  - `<3.4>` 处，重新设置 `currentSql` 和 `currentStatement` ，为当前传入的 `sql` 和 `ms` 。
  - `<3.5>` 处，添加 Statement 到 `statementList` 中。
  - `<3.6>` 处，创建 BatchResult 对象，并添加到 `batchResultList` 中。
  - 那么，如果下一次执行这个方法，如果传递相同的 `sql` 和 `ms` 进来，就会聚合到目前新创建的 BatchResult 对象中。

- `<4>` 处，调用 `StatementHandler#batch(Statement statement)` 方法，批处理。代码如下：

  ```
  // 有多个实现类，先看两个
  
  // PreparedStatementHandler.java
  
  @Override
  public void batch(Statement statement) throws SQLException {
      PreparedStatement ps = (PreparedStatement) statement;
      ps.addBatch();
  }
  
  // SimpleStatementHandler.java
  
  @Override
  public void batch(Statement statement) throws SQLException {
      String sql = boundSql.getSql();
      statement.addBatch(sql);
  }
  ```

  - 这段代码，是不是非常熟悉。

#### 6.4 doFlushStatements

```
// BatchExecutor.java

@Override
public List<BatchResult> doFlushStatements(boolean isRollback) throws SQLException {
    try {
        // <1> 如果 isRollback 为 true ，返回空数组
        if (isRollback) {
            return Collections.emptyList();
        }
        // <2> 遍历 statementList 和 batchResultList 数组，逐个提交批处理
        List<BatchResult> results = new ArrayList<>();
        for (int i = 0, n = statementList.size(); i < n; i++) {
            // <2.1> 获得 Statement 和 BatchResult 对象
            Statement stmt = statementList.get(i);
            applyTransactionTimeout(stmt);
            BatchResult batchResult = batchResultList.get(i);
            try {
                // <2.2> 批量执行
                batchResult.setUpdateCounts(stmt.executeBatch());
                // <2.3> 处理主键生成
                MappedStatement ms = batchResult.getMappedStatement();
                List<Object> parameterObjects = batchResult.getParameterObjects();
                KeyGenerator keyGenerator = ms.getKeyGenerator();
                if (Jdbc3KeyGenerator.class.equals(keyGenerator.getClass())) {
                    Jdbc3KeyGenerator jdbc3KeyGenerator = (Jdbc3KeyGenerator) keyGenerator;
                    jdbc3KeyGenerator.processBatch(ms, stmt, parameterObjects);
                } else if (!NoKeyGenerator.class.equals(keyGenerator.getClass())) { //issue #141
                    for (Object parameter : parameterObjects) {
                        keyGenerator.processAfter(this, ms, stmt, parameter);
                    }
                }
                // Close statement to close cursor #1109
                // <2.4> 关闭 Statement 对象
                closeStatement(stmt);
            } catch (BatchUpdateException e) {
                // 如果发生异常，则抛出 BatchExecutorException 异常
                StringBuilder message = new StringBuilder();
                message.append(batchResult.getMappedStatement().getId())
                        .append(" (batch index #")
                        .append(i + 1)
                        .append(")")
                        .append(" failed.");
                if (i > 0) {
                    message.append(" ")
                            .append(i)
                            .append(" prior sub executor(s) completed successfully, but will be rolled back.");
                }
                throw new BatchExecutorException(message.toString(), e, results, batchResult);
            }
            // <2.5> 添加到结果集
            results.add(batchResult);
        }
        return results;
    } finally {
        // <3.1> 关闭 Statement 们
        for (Statement stmt : statementList) {
            closeStatement(stmt);
        }
        // <3.2> 置空 currentSql、statementList、batchResultList 属性
        currentSql = null;
        statementList.clear();
        batchResultList.clear();
    }
}
```

- `<1>` 处，如果 `isRollback` 为 `true` ，返回空数组。

- ```
  <2>
  ```

   

  处，遍历

   

  ```
  statementList
  ```

   

  和

   

  ```
  batchResultList
  ```

   

  数组，逐个 Statement 提交批处理。

  - `<2.1>` 处，获得 Statement 和 BatchResult 对象。
  - 【重要】`<2.2>` 处，调用 `Statement#executeBatch()` 方法，批量执行。执行完成后，将结果赋值到 `BatchResult.updateCounts` 中。
  - `<2.3>` 处，处理主键生成。详细解析，见 [《精尽 MyBatis 源码分析 —— SQL 执行（三）之 KeyGenerator》](http://svip.iocoder.cn/MyBatis/executor-3) 。
  - `<2.4>` 处，调用 `#closeStatement(stmt)` 方法，关闭 Statement 对象。
  - `<2.5>` 处，添加到结果集。

- `<3.1>` 处，关闭 `Statement` 们。

- `<3.2>` 处，置空 `currentSql`、`statementList`、`batchResultList` 属性。

#### 6.5 doQuery

```
// BatchExecutor.java

@Override
public <E> List<E> doQuery(MappedStatement ms, Object parameterObject, RowBounds rowBounds, ResultHandler resultHandler, BoundSql boundSql)
        throws SQLException {
    Statement stmt = null;
    try {
        // <1> 刷入批处理语句
        flushStatements();
        Configuration configuration = ms.getConfiguration();
        // 创建 StatementHandler 对象
        StatementHandler handler = configuration.newStatementHandler(wrapper, ms, parameterObject, rowBounds, resultHandler, boundSql);
        // 获得 Connection 对象
        Connection connection = getConnection(ms.getStatementLog());
        // 创建 Statement 或 PrepareStatement 对象
        stmt = handler.prepare(connection, transaction.getTimeout());
        // 设置 SQL 上的参数，例如 PrepareStatement 对象上的占位符
        handler.parameterize(stmt);
        // 执行 StatementHandler  ，进行读操作
        return handler.query(stmt, resultHandler);
    } finally {
        // 关闭 StatementHandler 对象
        closeStatement(stmt);
    }
}
```

- 和 SimpleExecutor 的该方法，逻辑差不多。差别在于 `<1>` 处，发生查询之前，先调用 `#flushStatements()` 方法，刷入批处理语句。

#### 6.6 doQueryCursor

```
// BatchExecutor.java

@Override
protected <E> Cursor<E> doQueryCursor(MappedStatement ms, Object parameter, RowBounds rowBounds, BoundSql boundSql) throws SQLException {
    // <1> 刷入批处理语句
    flushStatements();
    Configuration configuration = ms.getConfiguration();
    // 创建 StatementHandler 对象
    StatementHandler handler = configuration.newStatementHandler(wrapper, ms, parameter, rowBounds, null, boundSql);
    // 获得 Connection 对象
    Connection connection = getConnection(ms.getStatementLog());
    // 创建 Statement 或 PrepareStatement 对象
    Statement stmt = handler.prepare(connection, transaction.getTimeout());
    // 设置 Statement ，如果执行完成，则进行自动关闭
    stmt.closeOnCompletion();
    // 设置 SQL 上的参数，例如 PrepareStatement 对象上的占位符
    handler.parameterize(stmt);
    // 执行 StatementHandler  ，进行读操作
    return handler.queryCursor(stmt);
}
```

- 和 SimpleExecutor 的该方法，逻辑差不多。差别在于 `<1>` 处，发生查询之前，先调用 `#flushStatements()` 方法，刷入批处理语句。

## 7. 二级缓存

在开始看具体源码之间，我们先来理解**二级缓存**的定义：

> FROM 凯伦 [《聊聊MyBatis缓存机制》](https://tech.meituan.com/mybatis_cache.html)
>
> 在上文中提到的一级缓存中，**其最大的共享范围就是一个 SqlSession 内部**，如果多个 SqlSession 之间需要共享缓存，则需要使用到**二级缓存**。开启二级缓存后，会使用 CachingExecutor 装饰 Executor ，进入一级缓存的查询流程前，先在 CachingExecutor 进行二级缓存的查询，具体的工作流程如下所示。
>
> [![大体流程](http://static.iocoder.cn/images/MyBatis/2020_02_28/04.png)](http://static.iocoder.cn/images/MyBatis/2020_02_28/04.png)大体流程

- 那么，二级缓存，反应到具体代码里，是长什么样的呢？我们来打开 MappedStatement 类，代码如下：

  ```
  // MappedStatement.java
  
  /**
   * Cache 对象
   */
  private Cache cache;
  ```

  - 就是 `cache` 属性。在前面的文章中，我们已经看到，每个 XML Mapper 或 Mapper 接口的每个 SQL 操作声明，对应一个 MappedStatement 对象。通过 `@CacheNamespace` 或 `<cache />` 来声明，创建其所使用的 Cache 对象；也可以通过 `@CacheNamespaceRef` 或 `<cache-ref />` 来声明，使用指定 Namespace 的 Cache 对象。

  - 最终在 Configuration 类中的体现，代码如下：

    ```
    // Configuration.java
    
    /**
     * Cache 对象集合
     *
     * KEY：命名空间 namespace
     */
    protected final Map<String, Cache> caches = new StrictMap<>("Caches collection");
    ```

    - 一个 KEY 为 Namespace 的 Map 对象。

  - 可能上述描述比较绕口，胖友好好理解下。

- 通过在 `mybatis-config.xml` 中，配置如下开启二级缓存功能：

  ```
  <setting name="cacheEnabled" value="true"/>
  ```

#### 7.1 CachingExecutor

`org.apache.ibatis.executor.CachingExecutor` ，实现 Executor 接口，支持**二级缓存**的 Executor 的实现类。

###### 7.1.1 构造方法

```
// CachingExecutor.java

/**
 * 被委托的 Executor 对象
 */
private final Executor delegate;
/**
 * TransactionalCacheManager 对象
 */
private final TransactionalCacheManager tcm = new TransactionalCacheManager();

public CachingExecutor(Executor delegate) {
    // <1>
    this.delegate = delegate;
    // <2> 设置 delegate 被当前执行器所包装
    delegate.setExecutorWrapper(this);
}
```

- `tcm` 属性，TransactionalCacheManager 对象，支持事务的缓存管理器。因为**二级缓存**是支持跨 Session 进行共享，此处需要考虑事务，**那么，必然需要做到事务提交时，才将当前事务中查询时产生的缓存，同步到二级缓存中**。这个功能，就通过 TransactionalCacheManager 来实现。
- `<1>` 处，设置 `delegate` 属性，为被委托的 Executor 对象。
- `<2>` 处，调用 `delegate` 属性的 `#setExecutorWrapper(Executor executor)` 方法，设置 `delegate` 被**当前执行器**所包装。

###### 7.1.2 直接调用委托方法

CachingExecutor 的如下方法，具体的实现代码，是直接调用委托执行器 `delegate` 的对应的方法。代码如下：

```
// CachingExecutor.java

public Transaction getTransaction() { return delegate.getTransaction(); }

@Override
public boolean isClosed() { return delegate.isClosed(); }

@Override
public List<BatchResult> flushStatements() throws SQLException { return delegate.flushStatements(); }

@Override
public CacheKey createCacheKey(MappedStatement ms, Object parameterObject, RowBounds rowBounds, BoundSql boundSql) { return delegate.createCacheKey(ms, parameterObject, rowBounds, boundSql); }

@Override
public boolean isCached(MappedStatement ms, CacheKey key) { return delegate.isCached(ms, key); }

@Override
public void deferLoad(MappedStatement ms, MetaObject resultObject, String property, CacheKey key, Class<?> targetType) { delegate.deferLoad(ms, resultObject, property, key, targetType); }

@Override
public void clearLocalCache() { delegate.clearLocalCache(); }
```

###### 7.1.3 query

```
// CachingExecutor.java

@Override
public <E> List<E> query(MappedStatement ms, Object parameterObject, RowBounds rowBounds, ResultHandler resultHandler) throws SQLException {
    // 获得 BoundSql 对象
    BoundSql boundSql = ms.getBoundSql(parameterObject);
    // 创建 CacheKey 对象
    CacheKey key = createCacheKey(ms, parameterObject, rowBounds, boundSql);
    // 查询
    return query(ms, parameterObject, rowBounds, resultHandler, key, boundSql);
}

@Override
public <E> List<E> query(MappedStatement ms, Object parameterObject, RowBounds rowBounds, ResultHandler resultHandler, CacheKey key, BoundSql boundSql)
        throws SQLException {
    // <1> 
    Cache cache = ms.getCache();
    if (cache != null) { // <2> 
        // <2.1> 如果需要清空缓存，则进行清空
        flushCacheIfRequired(ms);
        if (ms.isUseCache() && resultHandler == null) { // <2.2>
            // 暂时忽略，存储过程相关
            ensureNoOutParams(ms, boundSql);
            @SuppressWarnings("unchecked")
            // <2.3> 从二级缓存中，获取结果
            List<E> list = (List<E>) tcm.getObject(cache, key);
            if (list == null) {
                // <2.4.1> 如果不存在，则从数据库中查询
                list = delegate.query(ms, parameterObject, rowBounds, resultHandler, key, boundSql);
                // <2.4.2> 缓存结果到二级缓存中
                tcm.putObject(cache, key, list); // issue #578 and #116
            }
            // <2.5> 如果存在，则直接返回结果
            return list;
        }
    }
    // <3> 不使用缓存，则从数据库中查询
    return delegate.query(ms, parameterObject, rowBounds, resultHandler, key, boundSql);
}
```

- `<1>` 处，调用 `MappedStatement#getCache()` 方法，获得 Cache 对象，即当前 MappedStatement 对象的**二级缓存**。

- `<3>` 处，如果**没有** Cache 对象，说明该 MappedStatement 对象，未设置**二级缓存**，则调用 `delegate` 属性的 `#query(...)` 方法，直接从数据库中查询。

- `<2>` 处，如果**有** Cache 对象，说明该 MappedStatement 对象，有设置**二级缓存**：

  - `<2.1>` 处，调用 `#flushCacheIfRequired(MappedStatement ms)` 方法，如果需要清空缓存，则进行清空。代码如下：

    ```
    // CachingExecutor.java
    
    private void flushCacheIfRequired(MappedStatement ms) {
        Cache cache = ms.getCache();
        if (cache != null && ms.isFlushCacheRequired()) { // 是否需要清空缓存
            tcm.clear(cache);
        }
    }
    ```

    - 通过 `@Options(flushCache = Options.FlushCachePolicy.TRUE)` 或 `<select flushCache="true">` 方式，开启需要清空缓存。
    - 调用 `TransactionalCache#clear()` 方法，清空缓存。**注意**，此**时**清空的仅仅，当前事务中查询数据产生的缓存。而**真正**的清空，在事务的提交时。这是为什么呢？还是因为**二级缓存**是跨 Session 共享缓存，在事务尚未结束时，不能对二级缓存做任何修改。😈 可能有点绕，胖友好好理解。

  - `<2.2>` 处，当 `MappedStatement#isUseCache()` 方法，返回 `true` 时，才使用二级缓存。默认开启。可通过 `@Options(useCache = false)` 或 `<select useCache="false">` 方法，关闭。

  - `<2.3>` 处，调用 `TransactionalCacheManager#getObject(Cache cache, CacheKey key)` 方法，从二级缓存中，获取结果。

  - 如果不存在缓存

    - `<2.4.1>` 处，调用 `delegate` 属性的 `#query(...)` 方法，再从数据库中查询。
    - `<2.4.2>` 处，调用 `TransactionalCacheManager#put(Cache cache, CacheKey key, Object value)` 方法，缓存结果到二级缓存中。😈 当然，正如上文所言，实际上，此处结果还没添加到二级缓存中。那具体是怎么样的呢？答案见 TransactionalCache 。

  - 如果存在缓存

    - `<2.5>` 处，如果存在，则直接返回结果。

###### 7.1.4 queryCursor

```
// CachingExecutor.java

@Override
public <E> Cursor<E> queryCursor(MappedStatement ms, Object parameter, RowBounds rowBounds) throws SQLException {
    // 如果需要清空缓存，则进行清空
    flushCacheIfRequired(ms);
    // 执行 delegate 对应的方法
    return delegate.queryCursor(ms, parameter, rowBounds);
}
```

- 无法开启二级缓存，所以只好调用 `delegate` 对应的方法。

###### 7.1.5 update

```
// CachingExecutor.java

@Override
public int update(MappedStatement ms, Object parameterObject) throws SQLException {
    // 如果需要清空缓存，则进行清空
    flushCacheIfRequired(ms);
    // 执行 delegate 对应的方法
    return delegate.update(ms, parameterObject);
}
```

###### 7.1.6 commit

```
// CachingExecutor.java

@Override
public void commit(boolean required) throws SQLException {
    // 执行 delegate 对应的方法
    delegate.commit(required);
    // 提交 TransactionalCacheManager
    tcm.commit();
}
```

- `delegate` 和 `tcm` 先后提交。

###### 7.1.7 rollback

```
// CachingExecutor.java

@Override
public void rollback(boolean required) throws SQLException {
    try {
        // 执行 delegate 对应的方法
        delegate.rollback(required);
    } finally {
        if (required) {
            // 回滚 TransactionalCacheManager
            tcm.rollback();
        }
    }
}
```

- `delegate` 和 `tcm` 先后回滚。

###### 7.1.8 close

```
// CachingExecutor.java

@Override
public void close(boolean forceRollback) {
    try {
        //issues #499, #524 and #573
        // 如果强制回滚，则回滚 TransactionalCacheManager
        if (forceRollback) {
            tcm.rollback();
        // 如果强制提交，则提交 TransactionalCacheManager
        } else {
            tcm.commit();
        }
    } finally {
        // 执行 delegate 对应的方法
        delegate.close(forceRollback);
    }
}
```

- 根据 `forceRollback` 属性，进行 `tcm` 和 `delegate` 对应的操作。

#### 7.2 TransactionalCacheManager

`org.apache.ibatis.cache.TransactionalCacheManager` ，TransactionalCache 管理器。

###### 7.2.1 构造方法

```
// TransactionalCacheManager.java

/**
 * Cache 和 TransactionalCache 的映射
 */
private final Map<Cache, TransactionalCache> transactionalCaches = new HashMap<>();
```

- 我们可以看到，`transactionalCaches` 是一个使用 Cache 作为 KEY ，TransactionalCache 作为 VALUE 的 Map 对象。

- 为什么是一个 Map 对象呢？因为在一次的事务过程中，可能有多个不同的 MappedStatement 操作，而它们可能对应多个 Cache 对象。

- TransactionalCache 是怎么创建的呢？答案在 `#getTransactionalCache(Cache cache)` 方法，代码如下：

  ```
  // TransactionalCacheManager.java
  
  private TransactionalCache getTransactionalCache(Cache cache) {
      return transactionalCaches.computeIfAbsent(cache, TransactionalCache::new);
  }
  ```

  - 优先，从 `transactionalCaches` 获得 Cache 对象，对应的 TransactionalCache 对象。
  - 如果不存在，则创建一个 TransactionalCache 对象，并添加到 `transactionalCaches` 中。

###### 7.2.2 putObject

`#putObject(Cache cache, CacheKey key, Object value)` 方法，添加 Cache + KV ，到缓存中。代码如下：

```
// TransactionalCacheManager.java

public void putObject(Cache cache, CacheKey key, Object value) {
    // 首先，获得 Cache 对应的 TransactionalCache 对象
    // 然后，添加 KV 到 TransactionalCache 对象中
    getTransactionalCache(cache).putObject(key, value);
}
```

###### 7.2.3 getObject

`#getObject(Cache cache, CacheKey key)` 方法，获得缓存中，指定 Cache + K 的值。代码如下：

```
// TransactionalCacheManager.java

public Object getObject(Cache cache, CacheKey key) {
    // 首先，获得 Cache 对应的 TransactionalCache 对象
    // 然后从 TransactionalCache 对象中，获得 key 对应的值
    return getTransactionalCache(cache).getObject(key);
}
```

###### 7.2.4 clear

`#clear()` 方法，清空缓存。代码如下：

```
// TransactionalCacheManager.java

public void clear(Cache cache) {
    getTransactionalCache(cache).clear();
}
```

###### 7.2.5 commit

`#commit()` 方法，提交所有 TransactionalCache 。代码如下：

```
// TransactionalCacheManager.java

public void commit() {
    for (TransactionalCache txCache : transactionalCaches.values()) {
        txCache.commit();
    }
}
```

- 通过调用该方法，TransactionalCache 存储的当前事务的缓存，会同步到其对应的 Cache 对象。

###### 7.2.6 rollback

`#rollback()` 方法，回滚所有 TransactionalCache 。代码如下：

```
// TransactionalCacheManager.java

public void rollback() {
    for (TransactionalCache txCache : transactionalCaches.values()) {
        txCache.rollback();
    }
}
```

#### 7.3 TransactionalCache

`org.apache.ibatis.cache.decorators.TransactionalCache` ，实现 Cache 接口，支持事务的 Cache 实现类，主要用于二级缓存中。英语比较好的胖友，可以看看如下注释：

> This class holds all cache entries that are to be added to the 2nd level cache during a Session.
> Entries are sent to the cache when commit is called or discarded if the Session is rolled back.
> Blocking cache support has been added. Therefore any get() that returns a cache miss
> will be followed by a put() so any lock associated with the key can be released.

###### 7.3.1 构造方法

```
// TransactionalCache.java

/**
 * 委托的 Cache 对象。
 *
 * 实际上，就是二级缓存 Cache 对象。
 */
private final Cache delegate;
/**
 * 提交时，清空 {@link #delegate}
 *
 * 初始时，该值为 false
 * 清理后{@link #clear()} 时，该值为 true ，表示持续处于清空状态
 */
private boolean clearOnCommit;
/**
 * 待提交的 KV 映射
 */
private final Map<Object, Object> entriesToAddOnCommit;
/**
 * 查找不到的 KEY 集合
 */
private final Set<Object> entriesMissedInCache;

public TransactionalCache(Cache delegate) {
    this.delegate = delegate;
    this.clearOnCommit = false;
    this.entriesToAddOnCommit = new HashMap<>();
    this.entriesMissedInCache = new HashSet<>();
}
```

- 胖友认真看下每个变量上的注释。
- 在事务未提交时，`entriesToAddOnCommit` 属性，会暂存当前事务新产生的缓存 KV 对。
- 在事务提交时，`entriesToAddOnCommit` 属性，会同步到二级缓存 `delegate` 中。

###### 7.3.2 getObject

```
// TransactionalCache.java

@Override
public Object getObject(Object key) {
    // issue #116
    // <1> 从 delegate 中获取 key 对应的 value
    Object object = delegate.getObject(key);
    // <2> 如果不存在，则添加到 entriesMissedInCache 中
    if (object == null) {
        entriesMissedInCache.add(key);
    }
    // issue #146
    // <3> 如果 clearOnCommit 为 true ，表示处于持续清空状态，则返回 null
    if (clearOnCommit) {
        return null;
    // <4> 返回 value
    } else {
        return object;
    }
}
```

- `<1>` 处，调用 `delegate` 的 `#getObject(Object key)` 方法，从 `delegate` 中获取 `key` 对应的 value 。
- `<2>` 处，如果不存在，则添加到 `entriesMissedInCache` 中。这是个神奇的逻辑？？？答案见 `commit()` 和 `#rollback()` 方法。
- `<3>` 处，如果 `clearOnCommit` 为 `true` ，表示处于持续清空状态，则返回 `null` 。因为在事务未结束前，我们执行的**清空缓存**操作不好同步到 `delegate` 中，所以只好通过 `clearOnCommit` 来标记处于清空状态。那么，如果处于该状态，自然就不能返回 `delegate` 中查找的结果。
- `<4>` 处，返回 value 。

###### 7.3.3 putObject

`#putObject(Object key, Object object)` 方法，暂存 KV 到 `entriesToAddOnCommit` 中。代码如下：

```
// TransactionalCache.java

@Override
public void putObject(Object key, Object object) {
    // 暂存 KV 到 entriesToAddOnCommit 中
    entriesToAddOnCommit.put(key, object);
}
```

###### 7.3.4 removeObject

```
// TransactionalCache.java

public Object removeObject(Object key) {
    return null;
}
```

- 不太明白为什么是这样的实现。不过目前也暂时不存在调用该方法的情况。暂时忽略。

###### 7.3.5 clear

`#clear()` 方法，清空缓存。代码如下：

```
// TransactionalCache.java

@Override
public void clear() {
    // <1> 标记 clearOnCommit 为 true
    clearOnCommit = true;
    // <2> 清空 entriesToAddOnCommit
    entriesToAddOnCommit.clear();
}
```

- `<1>` 处，标记 `clearOnCommit` 为 `true` 。
- `<2>` 处，清空 `entriesToAddOnCommit` 。
- 该方法，不会清空 `delegate` 的缓存。真正的清空，在事务提交时。

###### 7.3.6 commit

`#commit()` 方法，提交事务。**重头戏**，代码如下：

```
// TransactionalCache.java

public void commit() {
    // <1> 如果 clearOnCommit 为 true ，则清空 delegate 缓存
    if (clearOnCommit) {
        delegate.clear();
    }
    // 将 entriesToAddOnCommit、entriesMissedInCache 刷入 delegate 中
    flushPendingEntries();
    // 重置
    reset();
}
```

- `<1>` 处，如果 `clearOnCommit` 为 `true` ，则清空 `delegate` 缓存。

- `<2>` 处，调用 `#flushPendingEntries()` 方法，将 `entriesToAddOnCommit`、`entriesMissedInCache` 同步到 `delegate` 中。代码如下：

  ```
  // TransactionalCache.java
  
  private void flushPendingEntries() {
      // 将 entriesToAddOnCommit 刷入 delegate 中
      for (Map.Entry<Object, Object> entry : entriesToAddOnCommit.entrySet()) {
          delegate.putObject(entry.getKey(), entry.getValue());
      }
      // 将 entriesMissedInCache 刷入 delegate 中
      for (Object entry : entriesMissedInCache) {
          if (!entriesToAddOnCommit.containsKey(entry)) {
              delegate.putObject(entry, null);
          }
      }
  }
  ```

  - 在看这段代码时，笔者一直疑惑 `entriesMissedInCache` 同步到 `delegate` 中，会不会存在问题。因为当前事务未查找到，不代表其他事务恰好实际能查到。这样，岂不是会将缓存错误的置空。后来一想，缓存即使真的被错误的置空，最多也就多从数据库中查询一次罢了。😈

- `<3>` 处，调用 `#reset()` 方法，重置对象。代码如下：

  ```
  // TransactionalCache.java
  
  private void reset() {
      // 重置 clearOnCommit 为 false
      clearOnCommit = false;
      // 清空 entriesToAddOnCommit、entriesMissedInCache
      entriesToAddOnCommit.clear();
      entriesMissedInCache.clear();
  }
  ```

  - 因为，一个 Executor 可以提交多次事务，而 TransactionalCache 需要被重用，那么就需要重置回初始状态。

###### 7.3.7 rollback

`#rollback()` 方法，回滚事务。代码如下：

```
// TransactionalCache.java

public void rollback() {
    // <1> 从 delegate 移除出 entriesMissedInCache
    unlockMissedEntries();
    // <2> 重置
    reset();
}
```

- `<1>` 处，调用 `#unlockMissedEntries()` 方法，将 `entriesMissedInCache` 同步到 `delegate` 中。代码如下：

  ```
  // TransactionalCache.java
  
  private void unlockMissedEntries() {
      for (Object entry : entriesMissedInCache) {
          try {
              delegate.removeObject(entry);
          } catch (Exception e) {
              log.warn("Unexpected exception while notifiying a rollback to the cache adapter."
                      + "Consider upgrading your cache adapter to the latest version.  Cause: " + e);
          }
      }
  }
  ```

  - 即使事务回滚，也不妨碍在事务的执行过程中，发现 `entriesMissedInCache` 不存在对应的缓存。

- `<2>` 处，调用 `#reset()` 方法，重置对象。

## 8. 创建 Executor 对象

在上面的文章中，我们已经看了各种 Executor 的实现代码。那么，Executor 对象究竟在 MyBatis 中，是如何被创建的呢？Configuration 类中，提供 `#newExecutor(Transaction transaction, ExecutorType executorType)` 方法，代码如下：

```
// Configuration.java

/**
 * 创建 Executor 对象
 *
 * @param transaction 事务对象
 * @param executorType 执行器类型
 * @return Executor 对象
 */
public Executor newExecutor(Transaction transaction, ExecutorType executorType) {
    // <1> 获得执行器类型
    executorType = executorType == null ? defaultExecutorType : executorType; // 使用默认
    executorType = executorType == null ? ExecutorType.SIMPLE : executorType; // 使用 ExecutorType.SIMPLE
    // <2> 创建对应实现的 Executor 对象
    Executor executor;
    if (ExecutorType.BATCH == executorType) {
        executor = new BatchExecutor(this, transaction);
    } else if (ExecutorType.REUSE == executorType) {
        executor = new ReuseExecutor(this, transaction);
    } else {
        executor = new SimpleExecutor(this, transaction);
    }
    // <3> 如果开启缓存，创建 CachingExecutor 对象，进行包装
    if (cacheEnabled) {
        executor = new CachingExecutor(executor);
    }
    // <4> 应用插件
    executor = (Executor) interceptorChain.pluginAll(executor);
    return executor;
}
```

- `<1>` 处，获得执行器类型。可以通过在 `mybatis-config.xml` 配置文件，如下：

  ```
  // value 有三种类型：SIMPLE REUSE BATCH
  <setting name="defaultExecutorType" value="" />
  ```

- `org.apache.ibatis.session.ExecutorType` ，执行器类型。代码如下：

  ```
  // ExecutorType.java
  
  public enum ExecutorType {
  
      /**
       * {@link org.apache.ibatis.executor.SimpleExecutor}
       */
      SIMPLE,
      /**
       * {@link org.apache.ibatis.executor.ReuseExecutor}
       */
      REUSE,
      /**
       * {@link org.apache.ibatis.executor.BatchExecutor}
       */
      BATCH
  
  }
  ```

- `<2>` 处，创建对应实现的 Executor 对象。默认为 SimpleExecutor 对象。

- `<3>` 处，如果开启缓存，创建 CachingExecutor 对象，进行包装。

- `<4>` 处，应用插件。关于**插件**，我们在后续的文章中，详细解析。

## 9. ClosedExecutor

> 老艿艿：写到凌晨 1 点多，以为已经写完了，结果发现….

在 ResultLoaderMap 类中，有一个 ClosedExecutor 内部静态类，继承 BaseExecutor 抽象类，已经关闭的 Executor 实现类。代码如下：

```
// ResultLoaderMap.java

private static final class ClosedExecutor extends BaseExecutor {

    public ClosedExecutor() {
        super(null, null);
    }

    @Override
    public boolean isClosed() {
        return true;
    }

    @Override
    protected int doUpdate(MappedStatement ms, Object parameter) throws SQLException {
        throw new UnsupportedOperationException("Not supported.");
    }

    @Override
    protected List<BatchResult> doFlushStatements(boolean isRollback) throws SQLException {
        throw new UnsupportedOperationException("Not supported.");
    }

    @Override
    protected <E> List<E> doQuery(MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler, BoundSql boundSql) throws SQLException {
        throw new UnsupportedOperationException("Not supported.");
    }

    @Override
    protected <E> Cursor<E> doQueryCursor(MappedStatement ms, Object parameter, RowBounds rowBounds, BoundSql boundSql) throws SQLException {
        throw new UnsupportedOperationException("Not supported.");
    }
}
```

- 仅仅在 ResultLoaderMap 中，作为一个“空”的 Executor 对象。没有什么特殊的意义和用途。

## 10. ErrorContext

`org.apache.ibatis.executor.ErrorContext` ，错误上下文，负责记录错误日志。代码比较简单，也蛮有意思，胖友可以自己研究研究。

另外，也可以参考下 [《Mybatis 源码中获取到的错误日志灵感》](https://www.jianshu.com/p/2af47a3e473c) 。

# 2、StatementHandler

## 1. 概述

本文，我们来分享 SQL 执行的第二部分，`statement` 包。整体类图如下：[![类图](http://static.iocoder.cn/images/MyBatis/2020_03_03/01.png)](http://static.iocoder.cn/images/MyBatis/2020_03_03/01.png)类图

- 我们可以看到，整体是以 StatementHandler 为核心。所以，本文主要会看到的就是 StatementHandler 对 JDBC Statement 的各种操作。

而 StatementHandler 在整个 SQL 执行过程中，所处的位置如下：[![整体流程](http://static.iocoder.cn/images/MyBatis/2020_03_03/02.png)](http://static.iocoder.cn/images/MyBatis/2020_03_03/02.png)整体流程

## 2. StatementHandler

`org.apache.ibatis.executor.statement.StatementHandler` ，Statement 处理器，其中 Statement 包含 `java.sql.Statement`、`java.sql.PreparedStatement`、`java.sql.CallableStatement` 三种。代码如下：

```
// StatementHandler.java

public interface StatementHandler {

    /**
     * 准备操作，可以理解成创建 Statement 对象
     *
     * @param connection         Connection 对象
     * @param transactionTimeout 事务超时时间
     * @return Statement 对象
     */
    Statement prepare(Connection connection, Integer transactionTimeout) throws SQLException;

    /**
     * 设置 Statement 对象的参数
     *
     * @param statement Statement 对象
     */
    void parameterize(Statement statement) throws SQLException;
    
    /**
     * 添加 Statement 对象的批量操作
     *
     * @param statement Statement 对象
     */
    void batch(Statement statement) throws SQLException;
    /**
     * 执行写操作
     *
     * @param statement Statement 对象
     * @return 影响的条数
     */
    int update(Statement statement) throws SQLException;
    /**
     * 执行读操作
     *
     * @param statement Statement 对象
     * @param resultHandler ResultHandler 对象，处理结果
     * @param <E> 泛型
     * @return 读取的结果
     */
    <E> List<E> query(Statement statement, ResultHandler resultHandler) throws SQLException;
    /**
     * 执行读操作，返回 Cursor 对象
     *
     * @param statement Statement 对象
     * @param <E> 泛型
     * @return Cursor 对象
     */
    <E> Cursor<E> queryCursor(Statement statement) throws SQLException;

    /**
     * @return BoundSql 对象
     */
    BoundSql getBoundSql();
    /**
     * @return ParameterHandler 对象
     */
    ParameterHandler getParameterHandler();

}
```

- 比较简单，胖友愁一愁。

StatementHandler 有多个子类，如下图所示：[![类图](http://static.iocoder.cn/images/MyBatis/2020_03_03/03.png)](http://static.iocoder.cn/images/MyBatis/2020_03_03/03.png)类图

- 左边的三个实现类，分别对应 `java.sql.Statement`、`java.sql.PreparedStatement`、`java.sql.CallableStatement` 三种不同的实现类。
- 右边的 RoutingStatementHandler 实现类，负责将不同的 Statement 类型，路由到上述三个实现类上。

下面，我们先看右边的实现类，再看左边的实现类。

## 3. RoutingStatementHandler

`org.apache.ibatis.executor.statement.RoutingStatementHandler` ，实现 StatementHandler 接口，路由的 StatementHandler 对象，根据 Statement 类型，转发到对应的 StatementHandler 实现类中。

#### 3.1 构造方法

```
// RoutingStatementHandler.java

/**
 * 被委托的 StatementHandler 对象
 */
private final StatementHandler delegate;

public RoutingStatementHandler(Executor executor, MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler, BoundSql boundSql) {
    // 根据不同的类型，创建对应的 StatementHandler 实现类
    switch (ms.getStatementType()) {
        case STATEMENT:
            delegate = new SimpleStatementHandler(executor, ms, parameter, rowBounds, resultHandler, boundSql);
            break;
        case PREPARED:
            delegate = new PreparedStatementHandler(executor, ms, parameter, rowBounds, resultHandler, boundSql);
            break;
        case CALLABLE:
            delegate = new CallableStatementHandler(executor, ms, parameter, rowBounds, resultHandler, boundSql);
            break;
        default:
            throw new ExecutorException("Unknown statement type: " + ms.getStatementType());
    }
}
```

- 根据不同的类型，创建对应的 StatementHandler 实现类。
- 😈 经典的装饰器模式。实际上，有点多余。。。还不如改成工厂模式。

#### 3.2 实现方法

所有的实现方法，调用 `delegate` 对应的方法即可。代码如下：

```
// RoutingStatementHandler.java

@Override
public Statement prepare(Connection connection, Integer transactionTimeout) throws SQLException {
    return delegate.prepare(connection, transactionTimeout);
}

@Override
public void parameterize(Statement statement) throws SQLException {
    delegate.parameterize(statement);
}

@Override
public void batch(Statement statement) throws SQLException {
    delegate.batch(statement);
}

@Override
public int update(Statement statement) throws SQLException {
    return delegate.update(statement);
}

@Override
public <E> List<E> query(Statement statement, ResultHandler resultHandler) throws SQLException {
    return delegate.query(statement, resultHandler);
}

@Override
public <E> Cursor<E> queryCursor(Statement statement) throws SQLException {
    return delegate.queryCursor(statement);
}

@Override
public BoundSql getBoundSql() {
    return delegate.getBoundSql();
}

@Override
public ParameterHandler getParameterHandler() {
    return delegate.getParameterHandler();
}
```

- 是不是更加觉得，换成工厂模式更合适。

## 4. BaseStatementHandler

`org.apache.ibatis.executor.statement.BaseStatementHandler` ，实现 StatementHandler 接口，StatementHandler 基类，提供骨架方法，从而使子类只要实现指定的几个抽象方法即可。

#### 4.1 构造方法

```
// BaseStatementHandler.java

protected final Configuration configuration;
protected final ObjectFactory objectFactory;
protected final TypeHandlerRegistry typeHandlerRegistry;
protected final ResultSetHandler resultSetHandler;
protected final ParameterHandler parameterHandler;

protected final Executor executor;
protected final MappedStatement mappedStatement;
protected final RowBounds rowBounds;

protected BoundSql boundSql;

protected BaseStatementHandler(Executor executor, MappedStatement mappedStatement, Object parameterObject, RowBounds rowBounds, ResultHandler resultHandler, BoundSql boundSql) {
    // 获得 Configuration 对象
    this.configuration = mappedStatement.getConfiguration();

    this.executor = executor;
    this.mappedStatement = mappedStatement;
    this.rowBounds = rowBounds;

    // 获得 TypeHandlerRegistry 和 ObjectFactory 对象
    this.typeHandlerRegistry = configuration.getTypeHandlerRegistry();
    this.objectFactory = configuration.getObjectFactory();

    // <1> 如果 boundSql 为空，一般是写类操作，例如：insert、update、delete ，则先获得自增主键，然后再创建 BoundSql 对象
    if (boundSql == null) { // issue #435, get the key before calculating the statement
        // <1.1> 获得自增主键
        generateKeys(parameterObject);
        // <1.2> 创建 BoundSql 对象
        boundSql = mappedStatement.getBoundSql(parameterObject);
    }
    this.boundSql = boundSql;

    // <2> 创建 ParameterHandler 对象
    this.parameterHandler = configuration.newParameterHandler(mappedStatement, parameterObject, boundSql);
    // <3> 创建 ResultSetHandler 对象
    this.resultSetHandler = configuration.newResultSetHandler(executor, mappedStatement, rowBounds, parameterHandler, resultHandler, boundSql);
}
```

- 大体是比较好理解的，我们就看几个点。

- `<1>` 处，如果 `boundSql` 为空：

  - 一般是**写**类操作，例如：`insert`、`update`、`delete` 。代码如下：

    ```
    // SimpleExecutor.java
    
    @Override
    public int doUpdate(MappedStatement ms, Object parameter) throws SQLException {
        Statement stmt = null;
        try {
            Configuration configuration = ms.getConfiguration();
            // <x> 创建 StatementHandler 对象
            StatementHandler handler = configuration.newStatementHandler(this, ms, parameter, RowBounds.DEFAULT, null, null);
            // 初始化 StatementHandler 对象
            stmt = prepareStatement(handler, ms.getStatementLog());
            // 执行 StatementHandler ，进行写操作
            return handler.update(stmt);
        } finally {
            // 关闭 StatementHandler 对象
            closeStatement(stmt);
        }
    }
    ```

    - `<x>` 处，调用 `Configuration#newStatementHandler(Executor executor, MappedStatement mappedStatement, Object parameterObject, RowBounds rowBounds, ResultHandler resultHandler, BoundSql boundSql)` 方法，创建 StatementHandler 对象。其中，方法参数 `boundSql` 为 `null` 。

  - `<1.1>` 处，调用 `#generateKeys(Object parameter)` 方法，获得自增主键。代码如下：

    ```
    // BaseStatementHandler.java
    
    protected void generateKeys(Object parameter) {
        // 获得 KeyGenerator 对象
        KeyGenerator keyGenerator = mappedStatement.getKeyGenerator();
        ErrorContext.instance().store();
        // 前置处理，创建自增编号到 parameter 中
        keyGenerator.processBefore(executor, mappedStatement, null, parameter);
        ErrorContext.instance().recall();
    }
    ```

    - 通过 KeyGenerator 对象，创建自增编号到 `parameter` 中。😈 详细的解析，见后续的 KeyGenerator 的内容。

  - `<1.2>` 处，调用 `MappedStatement#getBoundSql(Object parameterObject)` 方法，创建 BoundSql 对象。

  - 这个流程，可以调试下 `BindingTest#shouldInsertAuthorWithSelectKeyAndDynamicParams()` 单元测试方法。

- `<2>` 处，调用 `Configuration#newParameterHandler(MappedStatement mappedStatement, Object parameterObject, BoundSql boundSql)` 方法，创建 ParameterHandler 对象。代码如下：

  ```
  // Configuration.java
  
  public ParameterHandler newParameterHandler(MappedStatement mappedStatement, Object parameterObject, BoundSql boundSql) {
      // 创建 ParameterHandler 对象
      ParameterHandler parameterHandler = mappedStatement.getLang().createParameterHandler(mappedStatement, parameterObject, boundSql);
      // 应用插件
      parameterHandler = (ParameterHandler) interceptorChain.pluginAll(parameterHandler);
      return parameterHandler;
  }
  
  // XMLLanguageDriver.java
  
  @Override
  public ParameterHandler createParameterHandler(MappedStatement mappedStatement, Object parameterObject, BoundSql boundSql) {
      // 创建 DefaultParameterHandler 对象
      return new DefaultParameterHandler(mappedStatement, parameterObject, boundSql);
  }
  ```

  - 从代码中，可以看到，创建的是 DefaultParameterHandler 对象。而这个类，在 [《精尽 MyBatis 源码分析 —— SQL 初始化（下）之 SqlSource》](http://svip.iocoder.cn/MyBatis/scripting-2) 的 [「7.1 DefaultParameterHandler」](http://svip.iocoder.cn/MyBatis/executor-2/#) 已经有详细解析啦。

- `<3>` 处，创建 ResultSetHandler 对象。详细解析，见 [《精尽 MyBatis 源码分析 —— SQL 执行（四）之 ResultSetHandler》](http://svip.iocoder.cn/MyBatis/executor-4) 。

#### 4.2 prepare

```
// BaseStatementHandler.java

@Override
public Statement prepare(Connection connection, Integer transactionTimeout) throws SQLException {
    ErrorContext.instance().sql(boundSql.getSql());
    Statement statement = null;
    try {
        // <1> 创建 Statement 对象
        statement = instantiateStatement(connection);
        // 设置超时时间
        setStatementTimeout(statement, transactionTimeout);
        // 设置 fetchSize
        setFetchSize(statement);
        return statement;
    } catch (SQLException e) {
        // 发生异常，进行关闭
        closeStatement(statement);
        throw e;
    } catch (Exception e) {
        // 发生异常，进行关闭
        closeStatement(statement);
        throw new ExecutorException("Error preparing statement.  Cause: " + e, e);
    }
}
```

- `<1>` 处，创建 `#instantiateStatement(Connection connection)` 方法，创建 Statement 对象。这是一个抽象方法，需要子类去实现。代码如下：

  ```
  // BaseStatementHandler.java
  
  protected abstract Statement instantiateStatement(Connection connection) throws SQLException;
  ```

- `<2>` 处，调用 `#setStatementTimeout(Statement stmt, Integer transactionTimeout)` 方法，设置超时时间。代码如下：

  ```
  // BaseStatementHandler.java
  
  protected void setStatementTimeout(Statement stmt, Integer transactionTimeout) throws SQLException {
      // 获得 queryTimeout
      Integer queryTimeout = null;
      if (mappedStatement.getTimeout() != null) {
          queryTimeout = mappedStatement.getTimeout();
      } else if (configuration.getDefaultStatementTimeout() != null) {
          queryTimeout = configuration.getDefaultStatementTimeout();
      }
      // 设置查询超时时间
      if (queryTimeout != null) {
          stmt.setQueryTimeout(queryTimeout);
      }
      // 设置事务超时时间
      StatementUtil.applyTransactionTimeout(stmt, queryTimeout, transactionTimeout);
  }
  ```

- `<3>` 处，设置 `fetchSize` 。代码如下：

  ```
  // BaseStatementHandler.java
  
  protected void setFetchSize(Statement stmt) throws SQLException {
      // 获得 fetchSize 。非空，则进行设置
      Integer fetchSize = mappedStatement.getFetchSize();
      if (fetchSize != null) {
          stmt.setFetchSize(fetchSize);
          return;
      }
      // 获得 defaultFetchSize 。非空，则进行设置
      Integer defaultFetchSize = configuration.getDefaultFetchSize();
      if (defaultFetchSize != null) {
          stmt.setFetchSize(defaultFetchSize);
      }
  }
  ```

  - 感兴趣的胖友，可以看看 [《聊聊jdbc statement的fetchSize》](https://juejin.im/post/5a6757e351882573541c86bb) 和 [《正确使用MySQL JDBC setFetchSize()方法解决JDBC处理大结果集 java.lang.OutOfMemoryError: Java heap space》](https://blog.csdn.net/seven_3306/article/details/9303879) 。

## 5. SimpleStatementHandler

`org.apache.ibatis.executor.statement.SimpleStatementHandler` ，继承 BaseStatementHandler 抽象类，`java.sql.Statement` 的 StatementHandler 实现类。

#### 5.1 构造方法

```
// SimpleStatementHandler.java

public SimpleStatementHandler(Executor executor, MappedStatement mappedStatement, Object parameter, RowBounds rowBounds, ResultHandler resultHandler, BoundSql boundSql) {
    super(executor, mappedStatement, parameter, rowBounds, resultHandler, boundSql);
}
```

#### 5.2 instantiateStatement

`#instantiateStatement()` 方法，创建 `java.sql.Statement` 对象。代码如下：

```
// SimpleStatementHandler.java

@Override
protected Statement instantiateStatement(Connection connection) throws SQLException {
    if (mappedStatement.getResultSetType() == ResultSetType.DEFAULT) {
        return connection.createStatement();
    } else {
        return connection.createStatement(mappedStatement.getResultSetType().getValue(), ResultSet.CONCUR_READ_ONLY);
    }
}
```

#### 5.3 parameterize

```
// SimpleStatementHandler.java

@Override
public void parameterize(Statement statement) throws SQLException {
    // N/A
}
```

- 空，因为无需做占位符参数的处理。

#### 5.4 query

```
// SimpleStatementHandler.java

@Override
public <E> List<E> query(Statement statement, ResultHandler resultHandler) throws SQLException {
    String sql = boundSql.getSql();
    // <1> 执行查询
    statement.execute(sql);
    // <2> 处理返回结果
    return resultSetHandler.handleResultSets(statement);
}
```

- `<1>` 处，调用 `Statement#execute(String sql)` 方法，执行查询。
- `<2>` 处，调用 `ResultHandler#handleResultSets(Statement stmt)` 方法，处理返回结果。

#### 5.5 queryCursor

```
// SimpleStatementHandler.java

@Override
public <E> Cursor<E> queryCursor(Statement statement) throws SQLException {
    String sql = boundSql.getSql();
    // <1> 执行查询
    statement.execute(sql);
    // <2> 处理返回的 Cursor 结果
    return resultSetHandler.handleCursorResultSets(statement);
}
```

- `<1>` 处，调用 `Statement#execute(String sql)` 方法，执行查询。
- `<2>` 处，调用 `ResultHandler#handleCursorResultSets(Statement stmt)` 方法，处理返回的 Cursor 结果。

#### 5.6 batch

```
// SimpleStatementHandler.java

@Override
public void batch(Statement statement) throws SQLException {
    String sql = boundSql.getSql();
    // 添加到批处理
    statement.addBatch(sql);
}
```

- 调用 `Statement#addBatch(String sql)` 方法，添加到批处理。

#### 5.7 update

```
// SimpleStatementHandler.java

@Override
public int update(Statement statement) throws SQLException {
    String sql = boundSql.getSql();
    Object parameterObject = boundSql.getParameterObject();
    KeyGenerator keyGenerator = mappedStatement.getKeyGenerator();
    int rows;
    // 如果是 Jdbc3KeyGenerator 类型
    if (keyGenerator instanceof Jdbc3KeyGenerator) {
        // <1.1> 执行写操作
        statement.execute(sql, Statement.RETURN_GENERATED_KEYS);
        // <2.2> 获得更新数量
        rows = statement.getUpdateCount();
        // <1.3> 执行 keyGenerator 的后置处理逻辑
        keyGenerator.processAfter(executor, mappedStatement, statement, parameterObject);
    // 如果是 SelectKeyGenerator 类型
    } else if (keyGenerator instanceof SelectKeyGenerator) {
        // <2.1> 执行写操作
        statement.execute(sql);
        // <2.2> 获得更新数量
        rows = statement.getUpdateCount();
        // <2.3> 执行 keyGenerator 的后置处理逻辑
        keyGenerator.processAfter(executor, mappedStatement, statement, parameterObject);
    } else {
        // <3.1> 执行写操作
        statement.execute(sql);
        // <3.2> 获得更新数量
        rows = statement.getUpdateCount();
    }
    return rows;
}
```

- 根据 `keyGenerator` 的类型，执行的逻辑，略有差异。
- `<1.1>`、`<1.2>`、`<1.3>` 处，调用 `Statement#execute(String sql, ...)` 方法，执行写操作。其中，`<1.1>` 比较特殊，使用数据自带的自增功能。
- `<1.2>`、`<2.2>`、`<3.2>` 处，调用 `Statement#getUpdateCount()` 方法，获得更新数量。
- `<1.3>`、`<2.3>` 处，调用 `KeyGenerator#processAfter(Executor executor, MappedStatement ms, Statement stmt, Object parameter)` 方法，执行 `keyGenerator` 的后置处理逻辑。
- 虽然有点长，逻辑还是很清晰的。

## 6. PreparedStatementHandler

`org.apache.ibatis.executor.statement.PreparedStatementHandler` ，继承 BaseStatementHandler 抽象类，`java.sql.PreparedStatement` 的 StatementHandler 实现类。

#### 6.1 构造方法

```
// PreparedStatementHandler.java

public PreparedStatementHandler(Executor executor, MappedStatement mappedStatement, Object parameter, RowBounds rowBounds, ResultHandler resultHandler, BoundSql boundSql) {
    super(executor, mappedStatement, parameter, rowBounds, resultHandler, boundSql);
}
```

#### 6.2 instantiateStatement

`#instantiateStatement()` 方法，创建 `java.sql.Statement` 对象。代码如下：

```
// PreparedStatementHandler.java

@Override
protected Statement instantiateStatement(Connection connection) throws SQLException {
    String sql = boundSql.getSql();
    // <1> 处理 Jdbc3KeyGenerator 的情况
    if (mappedStatement.getKeyGenerator() instanceof Jdbc3KeyGenerator) {
        String[] keyColumnNames = mappedStatement.getKeyColumns();
        if (keyColumnNames == null) {
            return connection.prepareStatement(sql, PreparedStatement.RETURN_GENERATED_KEYS);
        } else {
            return connection.prepareStatement(sql, keyColumnNames);
        }
    // <2>
    } else if (mappedStatement.getResultSetType() == ResultSetType.DEFAULT) {
        return connection.prepareStatement(sql);
    // <3>
    } else {
        return connection.prepareStatement(sql, mappedStatement.getResultSetType().getValue(), ResultSet.CONCUR_READ_ONLY);
    }
}
```

- `<1>` 处，处理 Jdbc3KeyGenerator 的情况。
- `<2>` + `<3>` 处，和 SimpleStatementHandler 的方式是一致的。

#### 6.3 parameterize

```
// PreparedStatementHandler.java

@Override
public void parameterize(Statement statement) throws SQLException {
    parameterHandler.setParameters((PreparedStatement) statement);
}
```

- 调用 `ParameterHandler#setParameters(PreparedStatement ps)` 方法，设置 PreparedStatement 的占位符参数。

#### 6.4 query

```
// PreparedStatementHandler.java

@Override
public <E> List<E> query(Statement statement, ResultHandler resultHandler) throws SQLException {
    PreparedStatement ps = (PreparedStatement) statement;
    // 执行查询
    ps.execute();
    // 处理返回结果
    return resultSetHandler.handleResultSets(ps);
}
```

#### 6.5 queryCursor

```
// PreparedStatementHandler.java

@Override
public <E> Cursor<E> queryCursor(Statement statement) throws SQLException {
    PreparedStatement ps = (PreparedStatement) statement;
    // 执行查询
    ps.execute();
    // 处理返回的 Cursor 结果
    return resultSetHandler.handleCursorResultSets(ps);
}
```

#### 6.6 batch

```
// PreparedStatementHandler.java

@Override
public void batch(Statement statement) throws SQLException {
    PreparedStatement ps = (PreparedStatement) statement;
    // 添加到批处理
    ps.addBatch();
}
```

#### 6.7 update

```
// PreparedStatementHandler.java

@Override
public int update(Statement statement) throws SQLException {
    PreparedStatement ps = (PreparedStatement) statement;
    // 执行写操作
    ps.execute();
    int rows = ps.getUpdateCount();
    // 获得更新数量
    Object parameterObject = boundSql.getParameterObject();
    // 执行 keyGenerator 的后置处理逻辑
    KeyGenerator keyGenerator = mappedStatement.getKeyGenerator();
    keyGenerator.processAfter(executor, mappedStatement, ps, parameterObject);
    return rows;
}
```

## 7. CallableStatementHandler

`org.apache.ibatis.executor.statement.CallableStatementHandler` ，继承 BaseStatementHandler 抽象类，`java.sql.CallableStatement` 的 StatementHandler 实现类。

因为本系列不分享**存储过程**相关的内容，所以省略。感兴趣的胖友，自己研究哈。

## 8. 创建 StatementHandler 对象

在上面的文章中，我们已经看了各种 StatementHandler 的实现代码。那么，StatementHandler 对象究竟在 MyBatis 中，是如何被创建的呢？Configuration 类中，提供 `#newStatementHandler(Executor executor, MappedStatement mappedStatement, Object parameterObject, RowBounds rowBounds, ResultHandler resultHandler, BoundSql boundSql)` 方法，代码如下：

```
// Configuration.java

public StatementHandler newStatementHandler(Executor executor, MappedStatement mappedStatement, Object parameterObject, RowBounds rowBounds, ResultHandler resultHandler, BoundSql boundSql) {
    // <1> 创建 RoutingStatementHandler 对象
    StatementHandler statementHandler = new RoutingStatementHandler(executor, mappedStatement, parameterObject, rowBounds, resultHandler, boundSql);
    // 应用插件
    statementHandler = (StatementHandler) interceptorChain.pluginAll(statementHandler);
    return statementHandler;
}
```

- `<1>` 处，创建 RoutingStatementHandler 对象。通过它，自动**路由**到适合的 StatementHandler 实现类。😈 此处一看，更加适合，使用工厂模式。
- `<2>` 处，应用插件。关于**插件**，我们在后续的文章中，详细解析。

# 3、KeyGenerator

## 1. 概述

本文，我们来分享 SQL 执行的第三部分，`keygen` 包。整体类图如下：[![类图](http://static.iocoder.cn/images/MyBatis/2020_03_06/01.png)](http://static.iocoder.cn/images/MyBatis/2020_03_06/01.png)类图

- 我们可以看到，整体是以 KeyGenerator 为核心。所以，本文主要会看到的就是 KeyGenerator 对**自增主键**的获取。

## 2. KeyGenerator

`org.apache.ibatis.executor.keygen.KeyGenerator` ，主键生成器接口。代码如下：

```
// KeyGenerator.java

public interface KeyGenerator {

    // SQL 执行前
    void processBefore(Executor executor, MappedStatement ms, Statement stmt, Object parameter);

    // SQL 执行后
    void processAfter(Executor executor, MappedStatement ms, Statement stmt, Object parameter);

}
```

- 可在 SQL 执行**之前**或**之后**，进行处理主键的生成。

- 实际上，KeyGenerator 类的命名虽然包含 Generator ，但是目前 MyBatis 默认的 KeyGenerator 实现类，都是基于数据库来实现**主键自增**的功能。

- `parameter` 参数，指的是什么呢？以下面的方法为示例：

  ```
  @Options(useGeneratedKeys = true, keyProperty = "id")
  @Insert({"insert into country (countryname,countrycode) values (#{countryname},#{countrycode})"})
  int insertBean(Country country);
  ```

  - 上面的，`country` 方法参数，就是一个 `parameter` 参数。
  - **KeyGenerator 在获取到主键后，会设置回 `parameter` 参数的对应属性**。

KeyGenerator 有三个子类，如下图所示：[![类图](http://static.iocoder.cn/images/MyBatis/2020_03_06/01.png)](http://static.iocoder.cn/images/MyBatis/2020_03_06/01.png)类图

- 具体的，我们下面逐小节来分享。

## 3. Jdbc3KeyGenerator

`org.apache.ibatis.executor.keygen.Jdbc3KeyGenerator` ，实现 KeyGenerator 接口，基于 `Statement#getGeneratedKeys()` 方法的 KeyGenerator 实现类，适用于 MySQL、H2 主键生成。

#### 3.1 构造方法

```
// Jdbc3KeyGenerator.java

/**
 * A shared instance.
 *
 * 共享的单例
 *
 * @since 3.4.3
 */
public static final Jdbc3KeyGenerator INSTANCE = new Jdbc3KeyGenerator();
```

- 单例。

#### 3.2 processBefore

```
@Override
public void processBefore(Executor executor, MappedStatement ms, Statement stmt, Object parameter) {
    // do nothing
}
```

- 空实现。因为对于 Jdbc3KeyGenerator 类的主键，是在 SQL 执行后，才生成。

#### 3.3 processAfter

```
// Jdbc3KeyGenerator.java

@Override
public void processAfter(Executor executor, MappedStatement ms, Statement stmt, Object parameter) {
    processBatch(ms, stmt, parameter);
}
```

- 调用 `#processBatch(Executor executor, MappedStatement ms, Statement stmt, Object parameter)` 方法，处理返回的自增主键。单个 `parameter` 参数，可以认为是批量的一个**特例**。

#### 3.4 processBatch

```
// Jdbc3KeyGenerator.java

public void processBatch(MappedStatement ms, Statement stmt, Object parameter) {
    // <1> 获得主键属性的配置。如果为空，则直接返回，说明不需要主键
    final String[] keyProperties = ms.getKeyProperties();
    if (keyProperties == null || keyProperties.length == 0) {
        return;
    }
    ResultSet rs = null;
    try {
        // <2> 获得返回的自增主键
        rs = stmt.getGeneratedKeys();
        final Configuration configuration = ms.getConfiguration();
        if (rs.getMetaData().getColumnCount() >= keyProperties.length) {
            // <3> 获得唯一的参数对象
            Object soleParam = getSoleParameter(parameter);
            if (soleParam != null) {
                // <3.1> 设置主键们，到参数 soleParam 中
                assignKeysToParam(configuration, rs, keyProperties, soleParam);
            } else {
                // <3.2> 设置主键们，到参数 parameter 中
                assignKeysToOneOfParams(configuration, rs, keyProperties, (Map<?, ?>) parameter);
            }
        }
    } catch (Exception e) {
        throw new ExecutorException("Error getting generated key or setting result to parameter object. Cause: " + e, e);
    } finally {
        // <4> 关闭 ResultSet 对象
        if (rs != null) {
            try {
                rs.close();
            } catch (Exception e) {
                // ignore
            }
        }
    }
}
```

- `<1>` 处，获得主键属性的配置。如果为空，则直接返回，说明不需要主键。

- 【重要】`<2>` 处，调用 `Statement#getGeneratedKeys()` 方法，获得返回的自增主键。

- ```
  <3>
  ```

   

  处，调用

   

  ```
  #getSoleParameter(Object parameter)
  ```

   

  方法，获得唯一的参数对象。详细解析，先跳到

   

  「3.4.1 getSoleParameter」

   

  。

  - `<3.1>` 处，调用 `#assignKeysToParam(...)` 方法，设置主键们，到参数 `soleParam` 中。详细解析，见 [「3.4.2 assignKeysToParam」](http://svip.iocoder.cn/MyBatis/executor-3/#) 。
  - `<3.2>` 处，调用 `#assignKeysToOneOfParams(...)` 方法，设置主键们，到参数 `parameter` 中。详细解析，见 [「3.4.3 assignKeysToOneOfParams」](http://svip.iocoder.cn/MyBatis/executor-3/#) 。

- `<4>` 处，关闭 ResultSet 对象。

###### 3.4.1 getSoleParameter

```
// Jdbc3KeyGenerator.java

/**
 * 获得唯一的参数对象
 *
 * 如果获得不到唯一的参数对象，则返回 null
 *
 * @param parameter 参数对象
 * @return 唯一的参数对象
 */
private Object getSoleParameter(Object parameter) {
    // <1> 如果非 Map 对象，则直接返回 parameter
    if (!(parameter instanceof ParamMap || parameter instanceof StrictMap)) {
        return parameter;
    }
    // <3> 如果是 Map 对象，则获取第一个元素的值
    // <2> 如果有多个元素，则说明获取不到唯一的参数对象，则返回 null
    Object soleParam = null;
    for (Object paramValue : ((Map<?, ?>) parameter).values()) {
        if (soleParam == null) {
            soleParam = paramValue;
        } else if (soleParam != paramValue) {
            soleParam = null;
            break;
        }
    }
    return soleParam;
}
```

- `<1>` 处，如下可以符合这个条件。代码如下：

  ```
  @Options(useGeneratedKeys = true, keyProperty = "id")
  @Insert({"insert into country (countryname,countrycode) values (#{country.countryname},#{country.countrycode})"})
  int insertNamedBean(@Param("country") Country country);
  ```

- `<2>` 处，如下可以符合这个条件。代码如下：

  ```
  @Options(useGeneratedKeys = true, keyProperty = "country.id")
  @Insert({"insert into country (countryname, countrycode) values (#{country.countryname}, #{country.countrycode})"})
  int insertMultiParams_keyPropertyWithWrongParamName2(@Param("country") Country country,
                                                       @Param("someId") Integer someId);
  ```

  - 虽然有 `country` 和 `someId` 参数，但是最终会被封装成一个 `parameter` 参数，类型为 ParamMap 类型。为什么呢？答案在 `ParamNameResolver#getNamedParams(Object[] args)` 方法中。
  - 如果是这个情况，获得的主键，会设置回 `country` 的 `id` 属性，因为注解上的 `keyProperty = "country.id"` 配置。
  - 😈 此处比较绕，也相对用的少。

- `<3>` 处，如下可以符合这个条件。代码如下：

  ```
  @Options(useGeneratedKeys = true, keyProperty = "id")
  @Insert({"insert into country (countryname, countrycode) values (#{country.countryname}, #{country.countrycode})"})
  int insertMultiParams_keyPropertyWithWrongParamName3(@Param("country") Country country);
  ```

  - 相比 `<2>` 的示例，主要是 `keyProperty = "id"` 的修改，和去掉了 `@Param("someId") Integer someId` 参数。
  - 实际上，这种情况，和 `<1>` 是类似的。

- 三种情况，`<2>` 和 `<3>` 有点复杂，胖友实际上，理解 `<1>` 即可。

###### 3.4.2 assignKeysToParam

```
// Jdbc3KeyGenerator.java

private void assignKeysToParam(final Configuration configuration, ResultSet rs, final String[] keyProperties, Object param)
        throws SQLException {
    final TypeHandlerRegistry typeHandlerRegistry = configuration.getTypeHandlerRegistry();
    final ResultSetMetaData rsmd = rs.getMetaData();
    // Wrap the parameter in Collection to normalize the logic.
    // <1> 包装成 Collection 对象
    Collection<?> paramAsCollection;
    if (param instanceof Object[]) {
        paramAsCollection = Arrays.asList((Object[]) param);
    } else if (!(param instanceof Collection)) {
        paramAsCollection = Collections.singletonList(param);
    } else {
        paramAsCollection = (Collection<?>) param;
    }
    TypeHandler<?>[] typeHandlers = null;
    // <2> 遍历 paramAsCollection 数组
    for (Object obj : paramAsCollection) {
        // <2.1> 顺序遍历 rs
        if (!rs.next()) {
            break;
        }
        // <2.2> 创建 MetaObject 对象
        MetaObject metaParam = configuration.newMetaObject(obj);
        // <2.3> 获得 TypeHandler 数组
        if (typeHandlers == null) {
            typeHandlers = getTypeHandlers(typeHandlerRegistry, metaParam, keyProperties, rsmd);
        }
        // <2.4> 填充主键们
        populateKeys(rs, metaParam, keyProperties, typeHandlers);
    }
}
```

- `<1>` 处，包装成 Collection 对象。通过这样的方式，使单个 `param` 参数的情况下，可以统一。

- `<2>` 处，遍历 `paramAsCollection` 数组：

  - `<2.1>` 处， 顺序遍历 `rs` ，相当于把当前的 ResultSet 对象的主键们，赋值给 `obj` 对象的对应属性。

  - `<2.2>` 处，创建 MetaObject 对象，实现对 `obj` 对象的属性访问。

  - `<2.3>` 处，调用 `#getTypeHandlers(...)` 方法，获得 TypeHandler 数组。代码如下：

    ```
    // Jdbc3KeyGenerator.java
    
    private TypeHandler<?>[] getTypeHandlers(TypeHandlerRegistry typeHandlerRegistry, MetaObject metaParam, String[] keyProperties, ResultSetMetaData rsmd) throws SQLException {
        // 获得主键们，对应的每个属性的，对应的 TypeHandler 对象
        TypeHandler<?>[] typeHandlers = new TypeHandler<?>[keyProperties.length];
        for (int i = 0; i < keyProperties.length; i++) {
            if (metaParam.hasSetter(keyProperties[i])) {
                Class<?> keyPropertyType = metaParam.getSetterType(keyProperties[i]);
                typeHandlers[i] = typeHandlerRegistry.getTypeHandler(keyPropertyType, JdbcType.forCode(rsmd.getColumnType(i + 1)));
            } else {
                throw new ExecutorException("No setter found for the keyProperty '" + keyProperties[i] + "' in '"
                        + metaParam.getOriginalObject().getClass().getName() + "'.");
            }
        }
        return typeHandlers;
    }
    ```

    - x

  - `<2.4>` 处，调用 `#populateKeys(...)` 方法，填充主键们。详细解析，见 [「3.5 populateKeys」](http://svip.iocoder.cn/MyBatis/executor-3/#) 。

###### 3.4.3 assignKeysToOneOfParams

```
// Jdbc3KeyGenerator.java

protected void assignKeysToOneOfParams(final Configuration configuration, ResultSet rs, final String[] keyProperties,
                                       Map<?, ?> paramMap) throws SQLException {
    // Assuming 'keyProperty' includes the parameter name. e.g. 'param.id'.
    // <1> 需要有 `.` 。
    int firstDot = keyProperties[0].indexOf('.');
    if (firstDot == -1) {
        throw new ExecutorException(
                "Could not determine which parameter to assign generated keys to. "
                        + "Note that when there are multiple parameters, 'keyProperty' must include the parameter name (e.g. 'param.id'). "
                        + "Specified key properties are " + ArrayUtil.toString(keyProperties) + " and available parameters are "
                        + paramMap.keySet());
    }
    // 获得真正的参数值
    String paramName = keyProperties[0].substring(0, firstDot);
    Object param;
    if (paramMap.containsKey(paramName)) {
        param = paramMap.get(paramName);
    } else {
        throw new ExecutorException("Could not find parameter '" + paramName + "'. "
                + "Note that when there are multiple parameters, 'keyProperty' must include the parameter name (e.g. 'param.id'). "
                + "Specified key properties are " + ArrayUtil.toString(keyProperties) + " and available parameters are "
                + paramMap.keySet());
    }
    // Remove param name from 'keyProperty' string. e.g. 'param.id' -> 'id'
    // 获得主键的属性的配置
    String[] modifiedKeyProperties = new String[keyProperties.length];
    for (int i = 0; i < keyProperties.length; i++) {
        if (keyProperties[i].charAt(firstDot) == '.' && keyProperties[i].startsWith(paramName)) {
            modifiedKeyProperties[i] = keyProperties[i].substring(firstDot + 1);
        } else {
            throw new ExecutorException("Assigning generated keys to multiple parameters is not supported. "
                    + "Note that when there are multiple parameters, 'keyProperty' must include the parameter name (e.g. 'param.id'). "
                    + "Specified key properties are " + ArrayUtil.toString(keyProperties) + " and available parameters are "
                    + paramMap.keySet());
        }
    }
    // 设置主键们，到参数 param 中
    assignKeysToParam(configuration, rs, modifiedKeyProperties, param);
}
```

- `<1>` 处，需要有 `.` 。例如：`@Options(useGeneratedKeys = true, keyProperty = "country.id")` 。
- `<2>` 处，获得真正的参数值。
- `<3>` 处，获得主键的属性的配置。
- `<4>` 处，调用 `#assignKeysToParam(...)` 方法，设置主键们，到参数 `param` 中。所以，后续流程，又回到了 [「3.4.2」](http://svip.iocoder.cn/MyBatis/executor-3/#) 咧。
- 关于这个方法，胖友自己模拟下这个情况，调试下会比较好理解。😈 当然，也可以不理解，嘿嘿。

#### 3.5 populateKeys

```
// Jdbc3KeyGenerator.java

private void populateKeys(ResultSet rs, MetaObject metaParam, String[] keyProperties, TypeHandler<?>[] typeHandlers) throws SQLException {
    // 遍历 keyProperties
    for (int i = 0; i < keyProperties.length; i++) {
        // 获得属性名
        String property = keyProperties[i];
        // 获得 TypeHandler 对象
        TypeHandler<?> th = typeHandlers[i];
        if (th != null) {
            // 从 rs 中，获得对应的 值
            Object value = th.getResult(rs, i + 1);
            // 设置到 metaParam 的对应 property 属性种
            metaParam.setValue(property, value);
        }
    }
}
```

- 代码比较简单，胖友看下注释即可。

## 4. SelectKeyGenerator

`org.apache.ibatis.executor.keygen.SelectKeyGenerator` ，实现 KeyGenerator 接口，基于从数据库查询主键的 KeyGenerator 实现类，适用于 Oracle、PostgreSQL 。

#### 4.1 构造方法

```
// SelectKeyGenerator.java

    public static final String SELECT_KEY_SUFFIX = "!selectKey";

/**
 * 是否在 before 阶段执行
 *
 * true ：before
 * after ：after
 */
private final boolean executeBefore;
/**
 * MappedStatement 对象
 */
private final MappedStatement keyStatement;

public SelectKeyGenerator(MappedStatement keyStatement, boolean executeBefore) {
    this.executeBefore = executeBefore;
    this.keyStatement = keyStatement;
}
```

#### 4.2 processBefore

```
// SelectKeyGenerator.java

@Override
public void processBefore(Executor executor, MappedStatement ms, Statement stmt, Object parameter) {
    if (executeBefore) {
        processGeneratedKeys(executor, ms, parameter);
    }
}
```

- 调用 `#processGeneratedKeys(...)` 方法。

#### 4.3 processAfter

```
// SelectKeyGenerator.java

@Override
public void processAfter(Executor executor, MappedStatement ms, Statement stmt, Object parameter) {
    if (!executeBefore) {
        processGeneratedKeys(executor, ms, parameter);
    }
}
```

- 也是调用 `#processGeneratedKeys(...)` 方法。

#### 4.4 processGeneratedKeys

```
// SelectKeyGenerator.java

private void processGeneratedKeys(Executor executor, MappedStatement ms, Object parameter) {
    try {
        // <1> 有查询主键的 SQL 语句，即 keyStatement 对象非空
        if (parameter != null && keyStatement != null && keyStatement.getKeyProperties() != null) {
            String[] keyProperties = keyStatement.getKeyProperties();
            final Configuration configuration = ms.getConfiguration();
            final MetaObject metaParam = configuration.newMetaObject(parameter);
            // Do not close keyExecutor.
            // The transaction will be closed by parent executor.
            // <2> 创建执行器，类型为 SimpleExecutor
            Executor keyExecutor = configuration.newExecutor(executor.getTransaction(), ExecutorType.SIMPLE);
            // <3> 执行查询主键的操作
            List<Object> values = keyExecutor.query(keyStatement, parameter, RowBounds.DEFAULT, Executor.NO_RESULT_HANDLER);
            // <4.1> 查不到结果，抛出 ExecutorException 异常
            if (values.size() == 0) {
                throw new ExecutorException("SelectKey returned no data.");
            // <4.2> 查询的结果过多，抛出 ExecutorException 异常
            } else if (values.size() > 1) {
                throw new ExecutorException("SelectKey returned more than one value.");
            } else {
                // <4.3> 创建 MetaObject 对象，访问查询主键的结果
                MetaObject metaResult = configuration.newMetaObject(values.get(0));
                // <4.3.1> 单个主键
                if (keyProperties.length == 1) {
                    // 设置属性到 metaParam 中，相当于设置到 parameter 中
                    if (metaResult.hasGetter(keyProperties[0])) {
                        setValue(metaParam, keyProperties[0], metaResult.getValue(keyProperties[0]));
                    } else {
                        // no getter for the property - maybe just a single value object
                        // so try that
                        setValue(metaParam, keyProperties[0], values.get(0));
                    }
                // <4.3.2> 多个主键
                } else {
                    // 遍历，进行赋值
                    handleMultipleProperties(keyProperties, metaParam, metaResult);
                }
            }
        }
    } catch (ExecutorException e) {
        throw e;
    } catch (Exception e) {
        throw new ExecutorException("Error selecting key or setting result to parameter object. Cause: " + e, e);
    }
}
```

- `<1>` 处，有查询主键的 SQL 语句，即 `keyStatement` 对象非空。

- `<2>` 处，创建执行器，类型为 SimpleExecutor 。

- 【重要】 `<3>` 处，调用 `Executor#query(...)` 方法，执行查询主键的操作。😈 简单脑暴下，按照 SelectKeyGenerator 的思路，岂不是可以可以接入 SnowFlake 算法，从而实现分布式主键。

- `<4.1>` 处，查不到结果，抛出 ExecutorException 异常。

- `<4.2>` 处，查询的结果过多，抛出 ExecutorException 异常。

- `<4.3>` 处，创建 MetaObject 对象，访问查询主键的结果。

  - `<4.3.1>` 处，**单个主键**，调用 `#setValue(MetaObject metaParam, String property, Object value)` 方法，设置属性到 `metaParam` 中，相当于设置到 `parameter` 中。代码如下：

    ```
    // SelectKeyGenerator.java
    
    private void setValue(MetaObject metaParam, String property, Object value) {
        if (metaParam.hasSetter(property)) {
            metaParam.setValue(property, value);
        } else {
            throw new ExecutorException("No setter found for the keyProperty '" + property + "' in " + metaParam.getOriginalObject().getClass().getName() + ".");
        }
    }
    ```

    - 简单，胖友自己瞅瞅。

  - `<4.3.2>` 处，**多个主键**，调用 `#handleMultipleProperties(String[] keyProperties, MetaObject metaParam, MetaObject metaResult)` 方法，遍历，进行赋值。代码如下：

    ```
    // SelectKeyGenerator.java
    
    private void handleMultipleProperties(String[] keyProperties,
                                          MetaObject metaParam, MetaObject metaResult) {
        String[] keyColumns = keyStatement.getKeyColumns();
        // 遍历，进行赋值
        if (keyColumns == null || keyColumns.length == 0) {
            // no key columns specified, just use the property names
            for (String keyProperty : keyProperties) {
                setValue(metaParam, keyProperty, metaResult.getValue(keyProperty));
            }
        } else {
            if (keyColumns.length != keyProperties.length) {
                throw new ExecutorException("If SelectKey has key columns, the number must match the number of key properties.");
            }
            for (int i = 0; i < keyProperties.length; i++) {
                setValue(metaParam, keyProperties[i], metaResult.getValue(keyColumns[i]));
            }
        }
    }
    ```

    - 最终，还是会调用 `#setValue(...)` 方法，进行赋值。

#### 4.5 示例

- [《MyBatis + Oracle 实现主键自增长的几种常用方式》](https://blog.csdn.net/wal1314520/article/details/77132305)
- [《mybatis + postgresql 返回递增主键的正确姿势及勘误》](https://blog.csdn.net/cdnight/article/details/72735108)

## 5. NoKeyGenerator

`org.apache.ibatis.executor.keygen.NoKeyGenerator` ，实现 KeyGenerator 接口，空的 KeyGenerator 实现类，即无需主键生成。代码如下：

```
// NoKeyGenerator.java

public class NoKeyGenerator implements KeyGenerator {

    /**
     * A shared instance.
     * @since 3.4.3
     */
    public static final NoKeyGenerator INSTANCE = new NoKeyGenerator();

    @Override
    public void processBefore(Executor executor, MappedStatement ms, Statement stmt, Object parameter) {
        // Do Nothing
    }

    @Override
    public void processAfter(Executor executor, MappedStatement ms, Statement stmt, Object parameter) {
        // Do Nothing
    }

}
```



# 4、ResultSetHandler

## 1. 概述

本文，我们来分享 SQL 执行的第四部分，SQL 执行后，响应的结果集 ResultSet 的处理，涉及 `executor/resultset`、`executor/result`、`cursor` 包。整体类图如下：[![类图](http://static.iocoder.cn/images/MyBatis/2020_03_09/01.png)](http://static.iocoder.cn/images/MyBatis/2020_03_09/01.png)类图

- 核心类是 ResultSetHandler 接口及其实现类 DefaultResultSetHandler 。在它的代码逻辑中，会调用类图中的其它类，实现将查询结果的 ResultSet ，转换成映射的对应结果。

## 2. ResultSetWrapper

> 老艿艿：在看具体的 DefaultResultSetHandler 的实现代码之前，我们先看看 ResultSetWrapper 的代码。因为 DefaultResultSetHandler 对 ResultSetWrapper 的调用比较多，避免混着解析。

`org.apache.ibatis.executor.resultset.ResultSetWrapper` ，`java.sql.ResultSet` 的 包装器，可以理解成 ResultSet 的工具类，提供给 DefaultResultSetHandler 使用。

#### 2.1 构造方法

```
// ResultSetWrapper.java

/**
 * ResultSet 对象
 */
private final ResultSet resultSet;
private final TypeHandlerRegistry typeHandlerRegistry;
/**
 * 字段的名字的数组
 */
private final List<String> columnNames = new ArrayList<>();
/**
 * 字段的 Java Type 的数组
 */
private final List<String> classNames = new ArrayList<>();
/**
 * 字段的 JdbcType 的数组
 */
private final List<JdbcType> jdbcTypes = new ArrayList<>();
private final Map<String, Map<Class<?>, TypeHandler<?>>> typeHandlerMap = new HashMap<>();
private final Map<String, List<String>> mappedColumnNamesMap = new HashMap<>();
private final Map<String, List<String>> unMappedColumnNamesMap = new HashMap<>();

public ResultSetWrapper(ResultSet rs, Configuration configuration) throws SQLException {
    this.typeHandlerRegistry = configuration.getTypeHandlerRegistry();
    this.resultSet = rs;
    // <1> 遍历 ResultSetMetaData 的字段们，解析出 columnNames、jdbcTypes、classNames 属性
    final ResultSetMetaData metaData = rs.getMetaData();
    final int columnCount = metaData.getColumnCount();
    for (int i = 1; i <= columnCount; i++) {
        columnNames.add(configuration.isUseColumnLabel() ? metaData.getColumnLabel(i) : metaData.getColumnName(i));
        jdbcTypes.add(JdbcType.forCode(metaData.getColumnType(i)));
        classNames.add(metaData.getColumnClassName(i));
    }
}
```

- `resultSet` 属性，被包装的 ResultSet 对象。
- `columnNames`、`classNames`、`jdbcTypes` 属性，在 `<1>` 处，通过遍历 ResultSetMetaData 的字段们，从而解析出来。

#### 2.2 getTypeHandler

```
// ResultSetWrapper.java

/**
 * TypeHandler 的映射
 *
 * KEY1：字段的名字
 * KEY2：Java 属性类型
 */
private final Map<String, Map<Class<?>, TypeHandler<?>>> typeHandlerMap = new HashMap<>();

/**
 * Gets the type handler to use when reading the result set.
 * Tries to get from the TypeHandlerRegistry by searching for the property type.
 * If not found it gets the column JDBC type and tries to get a handler for it.
 *
 * 获得指定字段名的指定 JavaType 类型的 TypeHandler 对象
 *
 * @param propertyType JavaType
 * @param columnName 执行字段
 * @return TypeHandler 对象
 */
public TypeHandler<?> getTypeHandler(Class<?> propertyType, String columnName) {
    TypeHandler<?> handler = null;
    // <1> 先从缓存的 typeHandlerMap 中，获得指定字段名的指定 JavaType 类型的 TypeHandler 对象
    Map<Class<?>, TypeHandler<?>> columnHandlers = typeHandlerMap.get(columnName);
    if (columnHandlers == null) {
        columnHandlers = new HashMap<>();
        typeHandlerMap.put(columnName, columnHandlers);
    } else {
        handler = columnHandlers.get(propertyType);
    }
    // <2> 如果获取不到，则进行查找
    if (handler == null) {
        // <2> 获得 JdbcType 类型
        JdbcType jdbcType = getJdbcType(columnName);
        // <2> 获得 TypeHandler 对象
        handler = typeHandlerRegistry.getTypeHandler(propertyType, jdbcType);
        // Replicate logic of UnknownTypeHandler#resolveTypeHandler
        // See issue #59 comment 10
        // <3> 如果获取不到，则再次进行查找
        if (handler == null || handler instanceof UnknownTypeHandler) {
            // <3> 使用 classNames 中的类型，进行继续查找 TypeHandler 对象
            final int index = columnNames.indexOf(columnName);
            final Class<?> javaType = resolveClass(classNames.get(index));
            if (javaType != null && jdbcType != null) {
                handler = typeHandlerRegistry.getTypeHandler(javaType, jdbcType);
            } else if (javaType != null) {
                handler = typeHandlerRegistry.getTypeHandler(javaType);
            } else if (jdbcType != null) {
                handler = typeHandlerRegistry.getTypeHandler(jdbcType);
            }
        }
        // <4> 如果获取不到，则使用 ObjectTypeHandler 对象
        if (handler == null || handler instanceof UnknownTypeHandler) {
            handler = new ObjectTypeHandler();
        }
        // <5> 缓存到 typeHandlerMap 中
        columnHandlers.put(propertyType, handler);
    }
    return handler;
}
```

- `<1>` 处，先从缓存的 `typeHandlerMap` 中，获得指定字段名的指定 JavaType 类型的 TypeHandler 对象。

- `<2>` 处，如果获取不到，则基于 `propertyType` + `jdbcType` 进行查找。其中，`#getJdbcType(String columnName)` 方法，获得 JdbcType 类型。代码如下：

  ```
  // ResultSetWrapper.java
  
  public JdbcType getJdbcType(String columnName) {
      for (int i = 0; i < columnNames.size(); i++) {
          if (columnNames.get(i).equalsIgnoreCase(columnName)) {
              return jdbcTypes.get(i);
          }
      }
      return null;
  }
  ```

  - 通过 `columnNames` 索引到位置 `i` ，从而到 `jdbcTypes` 中获得 JdbcType 类型。

- `<3>` 处，如果获取不到，则基于 `javaType` + `jdbcType` 进行查找。其中，`javaType` 使用 `classNames` 中的类型。而 `#resolveClass(String className)` 方法，获得对应的类。代码如下：

  ```
  // ResultSetWrapper.java
  
  private Class<?> resolveClass(String className) {
      try {
          // #699 className could be null
          if (className != null) {
              return Resources.classForName(className);
          }
      } catch (ClassNotFoundException e) {
          // ignore
      }
      return null;
  }
  ```

- `<4>` 处，如果获取不到，则使用 ObjectTypeHandler 对象。

- `<5>` 处，缓存 TypeHandler 对象，到 `typeHandlerMap` 中。

#### 2.3 loadMappedAndUnmappedColumnNames

`#loadMappedAndUnmappedColumnNames(ResultMap resultMap, String columnPrefix)` 方法，初始化**有 mapped** 和**无 mapped**的字段的名字数组。代码如下：

```
// ResultSetWrapper.java

/**
 * 有 mapped 的字段的名字的映射
 *
 * KEY：{@link #getMapKey(ResultMap, String)}
 * VALUE：字段的名字的数组
 */
private final Map<String, List<String>> mappedColumnNamesMap = new HashMap<>();
/**
 * 无 mapped 的字段的名字的映射
 *
 * 和 {@link #mappedColumnNamesMap} 相反
 */
private final Map<String, List<String>> unMappedColumnNamesMap = new HashMap<>();

private void loadMappedAndUnmappedColumnNames(ResultMap resultMap, String columnPrefix) throws SQLException {
    List<String> mappedColumnNames = new ArrayList<>();
    List<String> unmappedColumnNames = new ArrayList<>();
    // <1> 将 columnPrefix 转换成大写，并拼接到 resultMap.mappedColumns 属性上
    final String upperColumnPrefix = columnPrefix == null ? null : columnPrefix.toUpperCase(Locale.ENGLISH);
    final Set<String> mappedColumns = prependPrefixes(resultMap.getMappedColumns(), upperColumnPrefix);
    // <2> 遍历 columnNames 数组，根据是否在 mappedColumns 中，分别添加到 mappedColumnNames 和 unmappedColumnNames 中
    for (String columnName : columnNames) {
        final String upperColumnName = columnName.toUpperCase(Locale.ENGLISH);
        if (mappedColumns.contains(upperColumnName)) {
            mappedColumnNames.add(upperColumnName);
        } else {
            unmappedColumnNames.add(columnName);
        }
    }
    // <3> 将 mappedColumnNames 和 unmappedColumnNames 结果，添加到 mappedColumnNamesMap 和 unMappedColumnNamesMap 中
    mappedColumnNamesMap.put(getMapKey(resultMap, columnPrefix), mappedColumnNames);
    unMappedColumnNamesMap.put(getMapKey(resultMap, columnPrefix), unmappedColumnNames);
}
```

- `<1>` 处，将 `columnPrefix` 转换成大写，后调用 `#prependPrefixes(Set<String> columnNames, String prefix)` 方法，拼接到 `resultMap.mappedColumns` 属性上。代码如下：

  ```
  // ResultSetWrapper.java
  
  private Set<String> prependPrefixes(Set<String> columnNames, String prefix) {
      // 直接返回 columnNames ，如果符合如下任一情况
      if (columnNames == null || columnNames.isEmpty() || prefix == null || prefix.length() == 0) {
          return columnNames;
      }
      // 拼接前缀 prefix ，然后返回
      final Set<String> prefixed = new HashSet<>();
      for (String columnName : columnNames) {
          prefixed.add(prefix + columnName);
      }
      return prefixed;
  }
  ```

  - 当然，可能有胖友，跟我会懵逼，可能已经忘记什么是 `resultMap.mappedColumns` 。我们来举个示例：

    ```
    <resultMap id="B" type="Object">
        <result property="year" column="year"/>
    </resultMap>
    
    <select id="testResultMap" parameterType="Integer" resultMap="A">
        SELECT * FROM subject
    </select>
    ```

    - 此处的 `column="year"` ，就会被添加到 `resultMap.mappedColumns` 属性上。

- `<2>` 处，遍历 `columnNames` 数组，根据是否在 `mappedColumns` 中，分别添加到 `mappedColumnNames` 和 `unmappedColumnNames` 中。

- `<3>` 处，将 `mappedColumnNames` 和 `unmappedColumnNames` 结果，添加到 `mappedColumnNamesMap` 和 `unMappedColumnNamesMap` 中。其中，`#getMapKey(ResultMap resultMap, String columnPrefix)` 方法，获得缓存的 KEY 。代码如下：

  ```
  // ResultSetWrapper.java
  
  private String getMapKey(ResultMap resultMap, String columnPrefix) {
      return resultMap.getId() + ":" + columnPrefix;
  }
  ```

下面，我们看个类似的，会调用该方法的方法：

- `#getMappedColumnNames(ResultMap resultMap, String columnPrefix)` 方法，获得**有** mapped 的字段的名字的数组。代码如下：

  ```
  // ResultSetWrapper.java
  
  public List<String> getMappedColumnNames(ResultMap resultMap, String columnPrefix) throws SQLException {
      // 获得对应的 mapped 数组
      List<String> mappedColumnNames = mappedColumnNamesMap.get(getMapKey(resultMap, columnPrefix));
      if (mappedColumnNames == null) {
          // 初始化
          loadMappedAndUnmappedColumnNames(resultMap, columnPrefix);
          // 重新获得对应的 mapped 数组
          mappedColumnNames = mappedColumnNamesMap.get(getMapKey(resultMap, columnPrefix));
      }
      return mappedColumnNames;
  }
  ```

- `#getUnmappedColumnNames(ResultMap resultMap, String columnPrefix)` 方法，获得**无** mapped 的字段的名字的数组。代码如下：

  ```
  // ResultSetWrapper.java
  
  public List<String> getUnmappedColumnNames(ResultMap resultMap, String columnPrefix) throws SQLException {
      // 获得对应的 unMapped 数组
      List<String> unMappedColumnNames = unMappedColumnNamesMap.get(getMapKey(resultMap, columnPrefix));
      if (unMappedColumnNames == null) {
          // 初始化
          loadMappedAndUnmappedColumnNames(resultMap, columnPrefix);
          // 重新获得对应的 unMapped 数组
          unMappedColumnNames = unMappedColumnNamesMap.get(getMapKey(resultMap, columnPrefix));
      }
      return unMappedColumnNames;
  }
  ```

😈 具体这两个方法什么用途呢？待到我们在 DefaultResultSetHandler 类里来看。

## 3. ResultSetHandler

`org.apache.ibatis.executor.resultset.ResultSetHandler` ，`java.sql.ResultSet` 处理器接口。代码如下：

```
// ResultSetHandler.java

public interface ResultSetHandler {

    /**
     * 处理 {@link java.sql.ResultSet} 成映射的对应的结果
     *
     * @param stmt Statement 对象
     * @param <E> 泛型
     * @return 结果数组
     */
    <E> List<E> handleResultSets(Statement stmt) throws SQLException;

    /**
     * 处理 {@link java.sql.ResultSet} 成 Cursor 对象
     *
     * @param stmt Statement 对象
     * @param <E> 泛型
     * @return Cursor 对象
     */
    <E> Cursor<E> handleCursorResultSets(Statement stmt) throws SQLException;

    // 暂时忽略，和存储过程相关
    void handleOutputParameters(CallableStatement cs) throws SQLException;

}
```

#### 3.1 DefaultResultSetHandler

> 老艿艿：保持冷静，DefaultResultSetHandler 有小 1000 行的代码。

`org.apache.ibatis.executor.resultset.DefaultResultSetHandler` ，实现 ResultSetHandler 接口，默认的 ResultSetHandler 实现类。

###### 3.1.1 构造方法

```
// DefaultResultSetHandler.java

private static final Object DEFERED = new Object();

private final Executor executor;
private final Configuration configuration;
private final MappedStatement mappedStatement;
private final RowBounds rowBounds;
private final ParameterHandler parameterHandler;
/**
 * 用户指定的用于处理结果的处理器。
 *
 * 一般情况下，不设置
 */
private final ResultHandler<?> resultHandler;
private final BoundSql boundSql;
private final TypeHandlerRegistry typeHandlerRegistry;
private final ObjectFactory objectFactory;
private final ReflectorFactory reflectorFactory;

// nested resultmaps
private final Map<CacheKey, Object> nestedResultObjects = new HashMap<>();
private final Map<String, Object> ancestorObjects = new HashMap<>();
private Object previousRowValue;

// multiple resultsets
// 存储过程相关的多 ResultSet 涉及的属性，可以暂时忽略
private final Map<String, ResultMapping> nextResultMaps = new HashMap<>();
private final Map<CacheKey, List<PendingRelation>> pendingRelations = new HashMap<>();

// Cached Automappings
/**
 * 自动映射的缓存
 *
 * KEY：{@link ResultMap#getId()} + ":" +  columnPrefix
 *
 * @see #createRowKeyForUnmappedProperties(ResultMap, ResultSetWrapper, CacheKey, String) 
 */
private final Map<String, List<UnMappedColumnAutoMapping>> autoMappingsCache = new HashMap<>();

// temporary marking flag that indicate using constructor mapping (use field to reduce memory usage)
/**
 * 是否使用构造方法创建该结果对象
 */
private boolean useConstructorMappings;

public DefaultResultSetHandler(Executor executor, MappedStatement mappedStatement, ParameterHandler parameterHandler, ResultHandler<?> resultHandler, BoundSql boundSql, RowBounds rowBounds) {
    this.executor = executor;
    this.configuration = mappedStatement.getConfiguration();
    this.mappedStatement = mappedStatement;
    this.rowBounds = rowBounds;
    this.parameterHandler = parameterHandler;
    this.boundSql = boundSql;
    this.typeHandlerRegistry = configuration.getTypeHandlerRegistry();
    this.objectFactory = configuration.getObjectFactory();
    this.reflectorFactory = configuration.getReflectorFactory();
    this.resultHandler = resultHandler;
}
```

- 属性比较多，我们看重点的几个。
- `resultHandler` 属性，ResultHandler 对象。用户指定的用于处理结果的处理器，一般情况下，不设置。详细解析，见 [「5. ResultHandler」](http://svip.iocoder.cn/MyBatis/executor-4/#) 和 [「3.1.2.3.3 storeObject」](http://svip.iocoder.cn/MyBatis/executor-4/#) 。
- `autoMappingsCache` 属性，自动映射的缓存。其中，KEY 为 `{@link ResultMap#getId()} + ":" + columnPrefix` 。详细解析，见 [「3.1.2.3.2.4 applyAutomaticMappings」](http://svip.iocoder.cn/MyBatis/executor-4/#) 。

###### 3.1.2 handleResultSets

`#handleResultSets(Statement stmt)` 方法，处理 `java.sql.ResultSet` 结果集，转换成映射的对应结果。代码如下：

```
// DefaultResultSetHandler.java

@Override
public List<Object> handleResultSets(Statement stmt) throws SQLException {
    ErrorContext.instance().activity("handling results").object(mappedStatement.getId());

    // <1> 多 ResultSet 的结果集合，每个 ResultSet 对应一个 Object 对象。而实际上，每个 Object 是 List<Object> 对象。
    // 在不考虑存储过程的多 ResultSet 的情况，普通的查询，实际就一个 ResultSet ，也就是说，multipleResults 最多就一个元素。
    final List<Object> multipleResults = new ArrayList<>();

    int resultSetCount = 0;
    // <2> 获得首个 ResultSet 对象，并封装成 ResultSetWrapper 对象
    ResultSetWrapper rsw = getFirstResultSet(stmt);

    // <3> 获得 ResultMap 数组
    // 在不考虑存储过程的多 ResultSet 的情况，普通的查询，实际就一个 ResultSet ，也就是说，resultMaps 就一个元素。
    List<ResultMap> resultMaps = mappedStatement.getResultMaps();
    int resultMapCount = resultMaps.size();
    validateResultMapsCount(rsw, resultMapCount); // <3.1> 校验
    while (rsw != null && resultMapCount > resultSetCount) {
        // <4.1> 获得 ResultMap 对象
        ResultMap resultMap = resultMaps.get(resultSetCount);
        // <4.2> 处理 ResultSet ，将结果添加到 multipleResults 中
        handleResultSet(rsw, resultMap, multipleResults, null);
        // <4.3> 获得下一个 ResultSet 对象，并封装成 ResultSetWrapper 对象
        rsw = getNextResultSet(stmt);
        // <4.4> 清理
        cleanUpAfterHandlingResultSet();
        // resultSetCount ++
        resultSetCount++;
    }

    // <5> 因为 `mappedStatement.resultSets` 只在存储过程中使用，本系列暂时不考虑，忽略即可
    String[] resultSets = mappedStatement.getResultSets();
    if (resultSets != null) {
        while (rsw != null && resultSetCount < resultSets.length) {
            ResultMapping parentMapping = nextResultMaps.get(resultSets[resultSetCount]);
            if (parentMapping != null) {
                String nestedResultMapId = parentMapping.getNestedResultMapId();
                ResultMap resultMap = configuration.getResultMap(nestedResultMapId);
                handleResultSet(rsw, resultMap, null, parentMapping);
            }
            rsw = getNextResultSet(stmt);
            cleanUpAfterHandlingResultSet();
            resultSetCount++;
        }
    }

    // <6> 如果是 multipleResults 单元素，则取首元素返回
    return collapseSingleResultList(multipleResults);
}
```

- 这个方法，不仅仅支持处理 Statement 和 PreparedStatement 返回的结果集，也支持存储过程的 CallableStatement 返回的结果集。而 CallableStatement 是支持返回多结果集的，这个大家要注意。😈 当然，还是老样子，本文不分析仅涉及存储过程的相关代码。哈哈哈。

- `<1>` 处，多 ResultSet 的结果集合，每个 ResultSet 对应一个 Object 对象。而实际上，每个 Object 是 List 对象。在不考虑存储过程的多 ResultSet 的情况，普通的查询，实际就一个 ResultSet ，也就是说，`multipleResults` **最多就一个元素**。

- `<2>` 处，调用 `#getFirstResultSet(Statement stmt)` 方法，获得首个 ResultSet 对象，并封装成 ResultSetWrapper 对象。代码如下：

  ```
  // DefaultResultSetHandler.java
  
  private ResultSetWrapper getFirstResultSet(Statement stmt) throws SQLException {
      ResultSet rs = stmt.getResultSet();
      // 可以忽略
      while (rs == null) {
          // move forward to get the first resultset in case the driver
          // doesn't return the resultset as the first result (HSQLDB 2.1)
          if (stmt.getMoreResults()) {
              rs = stmt.getResultSet();
          } else {
              if (stmt.getUpdateCount() == -1) {
                  // no more results. Must be no resultset
                  break;
              }
          }
      }
      // 将 ResultSet 对象，封装成 ResultSetWrapper 对象
      return rs != null ? new ResultSetWrapper(rs, configuration) : null;
  }
  ```

- `<3>` 处，调用 `MappedStatement#getResultMaps()` 方法，获得 ResultMap 数组。在不考虑存储过程的多 ResultSet 的情况，普通的查询，实际就一个 ResultSet ，也就是说，`resultMaps` **就一个元素**。

  - `<3.1>` 处，调用 `#validateResultMapsCount(ResultSetWrapper rsw, int resultMapCount)` 方法，校验至少有一个 ResultMap 对象。代码如下：

    ```
    // DefaultResultSetHandler.java
    
    private void validateResultMapsCount(ResultSetWrapper rsw, int resultMapCount) {
        if (rsw != null && resultMapCount < 1) {
            throw new ExecutorException("A query was run and no Result Maps were found for the Mapped Statement '" + mappedStatement.getId()
                    + "'.  It's likely that neither a Result Type nor a Result Map was specified.");
        }
    }
    ```

    - 不符合，则抛出 ExecutorException 异常。

- `<4.1>` 处，获得 ResultMap 对象。

- `<4.2>` 处，调用 `#handleResultSet(ResultSetWrapper rsw, ResultMap resultMap, List<Object> multipleResults, ResultMapping parentMapping)` 方法，处理 ResultSet ，将结果添加到 `multipleResults` 中。详细解析，见 [「3.1.2.1 handleResultSet」](http://svip.iocoder.cn/MyBatis/executor-4/#) 。

- `<4.3>` 处，调用 `#getNextResultSet(Statement stmt)` 方法，获得下一个 ResultSet 对象，并封装成 ResultSetWrapper 对象。😈 只有存储过程才有多 ResultSet 对象，所以可以忽略。也就是说，实际上，这个 `while` 循环对我们来说，就不需要啦。

- `<4.4>` 处，调用 `#cleanUpAfterHandlingResultSet()` 方法，执行清理。代码如下：

  ```
  // DefaultResultSetHandler.java
  
  private void cleanUpAfterHandlingResultSet() {
      nestedResultObjects.clear();
  }
  ```

- `<5>` 处，因为 `mappedStatement.resultSets` 只在存储过程中使用，本系列暂时不考虑，忽略即可。

- `<6>` 处，调用 `#collapseSingleResultList(List<Object> multipleResults)` 方法，如果是 `multipleResults` 单元素，则取首元素返回。代码如下：

  ```
  // DefaultResultSetHandler.java
  
  private List<Object> collapseSingleResultList(List<Object> multipleResults) {
      return multipleResults.size() == 1 ? (List<Object>) multipleResults.get(0) : multipleResults;
  }
  ```

  - 对于非存储过程的结果处理，都能符合 `multipleResults.size()` 。

####### 3.1.2.1 handleResultSet

`#handleResultSet(ResultSetWrapper rsw, ResultMap resultMap, List<Object> multipleResults, ResultMapping parentMapping)` 方法，处理 ResultSet ，将结果添加到 `multipleResults` 中。代码如下：

```
// DefaultResultSetHandler.java

private void handleResultSet(ResultSetWrapper rsw, ResultMap resultMap, List<Object> multipleResults, ResultMapping parentMapping) throws SQLException {
    try {
        // <1> 暂时忽略，因为只有存储过程的情况，调用该方法，parentMapping 为非空
        if (parentMapping != null) {
            handleRowValues(rsw, resultMap, null, RowBounds.DEFAULT, parentMapping);
        } else {
            // <2> 如果没有自定义的 resultHandler ，则创建默认的 DefaultResultHandler 对象
            if (resultHandler == null) {
                // <2> 创建 DefaultResultHandler 对象
                DefaultResultHandler defaultResultHandler = new DefaultResultHandler(objectFactory);
                // <3> 处理 ResultSet 返回的每一行 Row
                handleRowValues(rsw, resultMap, defaultResultHandler, rowBounds, null);
                // <4> 添加 defaultResultHandler 的处理的结果，到 multipleResults 中
                multipleResults.add(defaultResultHandler.getResultList());
            } else {
                // <3> 处理 ResultSet 返回的每一行 Row
                handleRowValues(rsw, resultMap, resultHandler, rowBounds, null);
            }
        }
    } finally {
        // issue #228 (close resultsets)
        // 关闭 ResultSet 对象
        closeResultSet(rsw.getResultSet());
    }
}
```

- `<1>` 处，暂时忽略，因为只有存储过程的情况，调用该方法，`parentMapping` 为非空。

- `<2>` 处，如果没有自定义的 `resultHandler` ，则创建默认的 DefaultResultHandler 对象。

- `<3>` 处，调用 `#handleRowValues(ResultSetWrapper rsw, ResultMap resultMap, ResultHandler<?> resultHandler, RowBounds rowBounds, ResultMapping parentMapping)` 方法，处理 ResultSet 返回的每一行 Row 。详细解析，见 [「3.1.2.2 handleRowValues」](http://svip.iocoder.cn/MyBatis/executor-4/#) 。

- 【特殊】`<4>` 处，使用**默认**的 DefaultResultHandler 对象，最终会将 `defaultResultHandler` 的处理的结果，到 `multipleResults` 中。而使用**自定义**的 `resultHandler` ，不会添加到 `multipleResults` 中。当然，因为自定义的 `resultHandler` 对象，是作为一个对象传入，所以在其内部，还是可以存储结果的。例如：

  ```
  @Select("select * from users")
  @ResultType(User.class)
  void getAllUsers(UserResultHandler resultHandler);
  ```

  - 感兴趣的胖友，可以看看 [《mybatis ResultHandler 示例》](http://outofmemory.cn/code-snippet/13271/mybatis-complex-bean-property-handle-with-custom-ResultHandler) 。

- `<5>` 处，调用 `#closeResultSet(ResultSet rs)` 方法关闭 ResultSet 对象。代码如下：

  ```
  // DefaultResultSetHandler.java
  
  private void closeResultSet(ResultSet rs) {
      try {
          if (rs != null) {
              rs.close();
          }
      } catch (SQLException e) {
          // ignore
      }
  }
  ```

####### 3.1.2.2 handleRowValues

`#handleRowValues(ResultSetWrapper rsw, ResultMap resultMap, ResultHandler<?> resultHandler, RowBounds rowBounds, ResultMapping parentMapping)` 方法，处理 ResultSet 返回的每一行 Row 。代码如下：

```
// DefaultResultSetHandler.java

public void handleRowValues(ResultSetWrapper rsw, ResultMap resultMap, ResultHandler<?> resultHandler, RowBounds rowBounds, ResultMapping parentMapping) throws SQLException {
    // <1> 处理嵌套映射的情况
    if (resultMap.hasNestedResultMaps()) {
        // 校验不要使用 RowBounds
        ensureNoRowBounds();
        // 校验不要使用自定义的 resultHandler
        checkResultHandler();
        // 处理嵌套映射的结果
        handleRowValuesForNestedResultMap(rsw, resultMap, resultHandler, rowBounds, parentMapping);
    // <2> 处理简单映射的情况
    } else {
        // <2.1> 处理简单映射的结果
        handleRowValuesForSimpleResultMap(rsw, resultMap, resultHandler, rowBounds, parentMapping);
    }
}
```

- 分成**嵌套映射**和**简单映射**的两种情况。

- ```
  <2>
  ```

   

  处理嵌套映射的情况：

  - `<2.1>` 处，调用 `#handleRowValuesForSimpleResultMap(...)` 方法，处理简单映射的结果。详细解析，见 [「3.1.2.3 handleRowValuesForSimpleResultMap 简单映射」](http://svip.iocoder.cn/MyBatis/executor-4/#) 。

- `<1>` 处理嵌套映射的情况：

  - `<1.1>` 处，调用 `#ensureNoRowBounds()` 方法，校验不要使用 RowBounds 。代码如下：

    ```
    // DefaultResultSetHandler.java
    
    private void ensureNoRowBounds() {
        // configuration.isSafeRowBoundsEnabled() 默认为 false
        if (configuration.isSafeRowBoundsEnabled() && rowBounds != null && (rowBounds.getLimit() < RowBounds.NO_ROW_LIMIT || rowBounds.getOffset() > RowBounds.NO_ROW_OFFSET)) {
            throw new ExecutorException("Mapped Statements with nested result mappings cannot be safely constrained by RowBounds. "
                    + "Use safeRowBoundsEnabled=false setting to bypass this check.");
        }
    }
    ```

    - 简单看看即可。

  - `<1.2>` 处，调用 `#checkResultHandler()` 方法，校验不要使用自定义的 `resultHandler` 。代码如下：

    ```
    // DefaultResultSetHandler.java
    
    protected void checkResultHandler() {
        // configuration.isSafeResultHandlerEnabled() 默认为 false
        if (resultHandler != null && configuration.isSafeResultHandlerEnabled() && !mappedStatement.isResultOrdered()) {
            throw new ExecutorException("Mapped Statements with nested result mappings cannot be safely used with a custom ResultHandler. "
                    + "Use safeResultHandlerEnabled=false setting to bypass this check "
                    + "or ensure your statement returns ordered data and set resultOrdered=true on it.");
        }
    }
    ```

    - 简单看看即可。

  - `<1.3>` 处，调用 `#handleRowValuesForSimpleResultMap(...)` 方法，处理嵌套映射的结果。详细解析，见 [「3.1.2.3 handleRowValuesForNestedResultMap 嵌套映射」](http://svip.iocoder.cn/MyBatis/executor-4/#) 。

####### 3.1.2.3 handleRowValuesForSimpleResultMap 简单映射

`#handleRowValuesForSimpleResultMap(ResultSetWrapper rsw, ResultMap resultMap, ResultHandler<?> resultHandler, RowBounds rowBounds, ResultMapping parentMapping)` 方法，处理简单映射的结果。代码如下：

```
// DefaultResultSetHandler.java

private void handleRowValuesForSimpleResultMap(ResultSetWrapper rsw, ResultMap resultMap, ResultHandler<?> resultHandler, RowBounds rowBounds, ResultMapping parentMapping) throws SQLException {
    // <1> 创建 DefaultResultContext 对象
    DefaultResultContext<Object> resultContext = new DefaultResultContext<>();
    // <2> 获得 ResultSet 对象，并跳到 rowBounds 指定的开始位置
    ResultSet resultSet = rsw.getResultSet();
    skipRows(resultSet, rowBounds);
    // <3> 循环
    while (shouldProcessMoreRows(resultContext, rowBounds) // 是否继续处理 ResultSet
            && !resultSet.isClosed() // ResultSet 是否已经关闭
            && resultSet.next()) { // ResultSet 是否还有下一条
        // <4> 根据该行记录以及 ResultMap.discriminator ，决定映射使用的 ResultMap 对象
        ResultMap discriminatedResultMap = resolveDiscriminatedResultMap(resultSet, resultMap, null);
        // <5> 根据最终确定的 ResultMap 对 ResultSet 中的该行记录进行映射，得到映射后的结果对象
        Object rowValue = getRowValue(rsw, discriminatedResultMap, null);
        // <6> 将映射创建的结果对象添加到 ResultHandler.resultList 中保存
        storeObject(resultHandler, resultContext, rowValue, parentMapping, resultSet);
    }
}
```

- `<1>` 处，创建 DefaultResultContext 对象。详细解析，胖友**先**跳到 [「4. ResultContext」](http://svip.iocoder.cn/MyBatis/executor-4/#) 中，看完就回来。

- `<2>` 处，获得 ResultSet 对象，并调用 `#skipRows(ResultSet rs, RowBounds rowBounds)` 方法，跳到 `rowBounds` 指定的开始位置。代码如下：

  ```
  // DefaultResultSetHandler.java
  
  private void skipRows(ResultSet rs, RowBounds rowBounds) throws SQLException {
      if (rs.getType() != ResultSet.TYPE_FORWARD_ONLY) {
          // 直接跳转到指定开始的位置
          if (rowBounds.getOffset() != RowBounds.NO_ROW_OFFSET) {
              rs.absolute(rowBounds.getOffset());
          }
      } else {
          // 循环，不断跳到开始的位置
          for (int i = 0; i < rowBounds.getOffset(); i++) {
              if (!rs.next()) {
                  break;
              }
          }
      }
  }
  ```

  - 关于 `org.apache.ibatis.session.RowBounds` 类，胖友可以看看 [《Mybatis3.3.x技术内幕（十三）：Mybatis之RowBounds分页原理》](https://my.oschina.net/zudajun/blog/671446) ，解释的非常不错。

- `<3>` 处，循环，满足如下三个条件。其中 `#shouldProcessMoreRows(ResultContext<?> context, RowBounds rowBounds)` 方法，是否继续处理 ResultSet 。代码如下：

  ```
  // DefaultResultSetHandler.java
  
  private boolean shouldProcessMoreRows(ResultContext<?> context, RowBounds rowBounds) {
      return !context.isStopped() && context.getResultCount() < rowBounds.getLimit();
  }
  ```

- `<4>` 处，调用 `#resolveDiscriminatedResultMap(...)` 方法，根据该行记录以及 `ResultMap.discriminator` ，决定映射使用的 ResultMap 对象。详细解析，等下看 [「3.1.2.3.1 resolveDiscriminatedResultMap」](http://svip.iocoder.cn/MyBatis/executor-4/#) 。

- `<5>` 处，调用 `#getRowValue(...)` 方法，根据最终确定的 ResultMap 对 ResultSet 中的该行记录进行映射，得到映射后的结果对象。详细解析，等下看 [「3.1.2.3.2 getRowValue」](http://svip.iocoder.cn/MyBatis/executor-4/#) 。

- `<6>` 处，调用 `#storeObject(...)` 方法，将映射创建的结果对象添加到 `ResultHandler.resultList` 中保存。详细解析，等下看 [「3.1.2.3.3 storeObject」](http://svip.iocoder.cn/MyBatis/executor-4/#) 。

######## 3.1.2.3.1 resolveDiscriminatedResultMap

`#resolveDiscriminatedResultMap(ResultSet rs, ResultMap resultMap, String columnPrefix)` 方法，根据该行记录以及 `ResultMap.discriminator` ，决定映射使用的 ResultMap 对象。代码如下：

```
// DefaultResultSetHandler.java

public ResultMap resolveDiscriminatedResultMap(ResultSet rs, ResultMap resultMap, String columnPrefix) throws SQLException {
    // 记录已经处理过的 Discriminator 对应的 ResultMap 的编号
    Set<String> pastDiscriminators = new HashSet<>();
    // 如果存在 Discriminator 对象，则基于其获得 ResultMap 对象
    Discriminator discriminator = resultMap.getDiscriminator();
    while (discriminator != null) { // 因为 Discriminator 可以嵌套 Discriminator ，所以是一个递归的过程
        // 获得 Discriminator 的指定字段，在 ResultSet 中该字段的值
        final Object value = getDiscriminatorValue(rs, discriminator, columnPrefix);
        // 从 Discriminator 获取该值对应的 ResultMap 的编号
        final String discriminatedMapId = discriminator.getMapIdFor(String.valueOf(value));
        // 如果存在，则使用该 ResultMap 对象
        if (configuration.hasResultMap(discriminatedMapId)) {
            // 获得该 ResultMap 对象
            resultMap = configuration.getResultMap(discriminatedMapId);
            // 判断，如果出现“重复”的情况，结束循环
            Discriminator lastDiscriminator = discriminator;
            discriminator = resultMap.getDiscriminator();
            if (discriminator == lastDiscriminator || !pastDiscriminators.add(discriminatedMapId)) {
                break;
            }
        // 如果不存在，直接结束循环
        } else {
            break;
        }
    }
    return resultMap;
}
```

- 对于大多数情况下，大家不太会使用 Discriminator 的功能，此处就直接返回 `resultMap` ，不会执行这个很复杂的逻辑。😈 所以，如果看不太懂的胖友，可以略过这个方法，问题也不大。

- 代码比较繁杂，胖友跟着注释看看，甚至可以调试下。其中，`<1>` 处，调用 `#getDiscriminatorValue(ResultSet rs, Discriminator discriminator, String columnPrefix)` 方法，获得 Discriminator 的指定字段，在 ResultSet 中该字段的值。代码如下：

  ```
  // DefaultResultSetHandler.java
  
  /**
   * 获得 ResultSet 的指定字段的值
   *
   * @param rs ResultSet 对象
   * @param discriminator Discriminator 对象
   * @param columnPrefix 字段名的前缀
   * @return 指定字段的值
   */
  private Object getDiscriminatorValue(ResultSet rs, Discriminator discriminator, String columnPrefix) throws SQLException {
      final ResultMapping resultMapping = discriminator.getResultMapping();
      final TypeHandler<?> typeHandler = resultMapping.getTypeHandler();
      // 获得 ResultSet 的指定字段的值
      return typeHandler.getResult(rs, prependPrefix(resultMapping.getColumn(), columnPrefix));
  }
  
  /**
   * 拼接指定字段的前缀
   *
   * @param columnName 字段的名字
   * @param prefix 前缀
   * @return prefix + columnName
   */
  private String prependPrefix(String columnName, String prefix) {
      if (columnName == null || columnName.length() == 0 || prefix == null || prefix.length() == 0) {
          return columnName;
      }
      return prefix + columnName;
  }
  ```

######## 3.1.2.3.2 getRowValue

`#getRowValue(ResultSetWrapper rsw, ResultMap resultMap, String columnPrefix)` 方法，根据最终确定的 ResultMap 对 ResultSet 中的该行记录进行映射，得到映射后的结果对象。代码如下：

```
// DefaultResultSetHandler.java

private Object getRowValue(ResultSetWrapper rsw, ResultMap resultMap, String columnPrefix) throws SQLException {
    // <1> 创建 ResultLoaderMap 对象
    final ResultLoaderMap lazyLoader = new ResultLoaderMap();
    // <2> 创建映射后的结果对象
    Object rowValue = createResultObject(rsw, resultMap, lazyLoader, columnPrefix);
    // <3> 如果 hasTypeHandlerForResultObject(rsw, resultMap.getType()) 返回 true ，意味着 rowValue 是基本类型，无需执行下列逻辑。
    if (rowValue != null && !hasTypeHandlerForResultObject(rsw, resultMap.getType())) {
        // <4> 创建 MetaObject 对象，用于访问 rowValue 对象
        final MetaObject metaObject = configuration.newMetaObject(rowValue);
        // <5> foundValues 代表，是否成功映射任一属性。若成功，则为 true ，若失败，则为 false
        boolean foundValues = this.useConstructorMappings;
        /// <6.1> 判断是否开启自动映射功能
        if (shouldApplyAutomaticMappings(resultMap, false)) {
            // <6.2> 自动映射未明确的列
            foundValues = applyAutomaticMappings(rsw, resultMap, metaObject, columnPrefix) || foundValues;
        }
        // <7> 映射 ResultMap 中明确映射的列
        foundValues = applyPropertyMappings(rsw, resultMap, metaObject, lazyLoader, columnPrefix) || foundValues;
        // <8> ↑↑↑ 至此，当前 ResultSet 的该行记录的数据，已经完全映射到结果对象 rowValue 的对应属性种
        foundValues = lazyLoader.size() > 0 || foundValues;
        // <9> 如果没有成功映射任意属性，则置空 rowValue 对象。
        // 当然，如果开启 `configuration.returnInstanceForEmptyRow` 属性，则不置空。默认情况下，该值为 false
        rowValue = foundValues || configuration.isReturnInstanceForEmptyRow() ? rowValue : null;
    }
    return rowValue;
}
```

- `<1>` 处，创建 ResultLoaderMap 对象。延迟加载相关。

- `<2>` 处，调用 `#createResultObject(...)` 方法，创建映射后的结果对象。详细解析，见 [「3.1.2.3.2.1 createResultObject」](http://svip.iocoder.cn/MyBatis/executor-4/#) 。😈 mmp ，这个逻辑的嵌套，真的太深太深了。

- `<3>` 处，调用 `#hasTypeHandlerForResultObject(rsw, resultMap.getType())` 方法，返回 `true` ，意味着 `rowValue` 是基本类型，无需执行下列逻辑。代码如下：

  ```
  // DefaultResultSetHandler.java
  
  // 判断是否结果对象是否有 TypeHandler 对象
  private boolean hasTypeHandlerForResultObject(ResultSetWrapper rsw, Class<?> resultType) {
      // 如果返回的字段只有一个，则直接判断该字段是否有 TypeHandler 对象
      if (rsw.getColumnNames().size() == 1) {
          return typeHandlerRegistry.hasTypeHandler(resultType, rsw.getJdbcType(rsw.getColumnNames().get(0)));
      }
      // 判断 resultType 是否有对应的 TypeHandler 对象
      return typeHandlerRegistry.hasTypeHandler(resultType);
  }
  ```

  - 有点绕，胖友可以调试下 `BindingTest#shouldInsertAuthorWithSelectKeyAndDynamicParams()` 方法。
  - 再例如，`<select resultType="Integer" />` 的情况。

- `<4>` 处，创建 MetaObject 对象，用于访问 `rowValue` 对象。

- `<5>` 处，`foundValues` 代表，是否成功映射任一属性。若成功，则为 `true` ，若失败，则为 `false` 。另外，此处使用 `useConstructorMappings` 作为 `foundValues` 的初始值，原因是，使用了构造方法创建该结果对象，意味着一定找到了任一属性。

- `<6.1>` 处，调用 `#shouldApplyAutomaticMappings(ResultMap resultMap, boolean isNested)` 方法，判断是否使用自动映射的功能。代码如下：

  ```
  // DefaultResultSetHandler.java
  
  private boolean shouldApplyAutomaticMappings(ResultMap resultMap, boolean isNested) {
      // 判断是否开启自动映射功能
      if (resultMap.getAutoMapping() != null) {
          return resultMap.getAutoMapping();
      } else {
          // 内嵌查询或嵌套映射时
          if (isNested) {
              return AutoMappingBehavior.FULL == configuration.getAutoMappingBehavior(); // 需要 FULL
          // 普通映射
          } else {
              return AutoMappingBehavior.NONE != configuration.getAutoMappingBehavior(); // 需要 PARTIAL 或 FULL
          }
      }
  }
  ```

  - `org.apache.ibatis.session.AutoMappingBehavior` ，自动映射行为的枚举。代码如下：

    ```
    // AutoMappingBehavior.java
    
    /**
     * Specifies if and how MyBatis should automatically map columns to fields/properties.
     *
     * 自动映射行为的枚举
     *
     * @author Eduardo Macarron
     */
    public enum AutoMappingBehavior {
    
        /**
         * Disables auto-mapping.
         *
         * 禁用自动映射的功能
         */
        NONE,
    
        /**
         * Will only auto-map results with no nested result mappings defined inside.
         *
         * 开启部分映射的功能
         */
        PARTIAL,
    
        /**
         * Will auto-map result mappings of any complexity (containing nested or otherwise).
         *
         * 开启全部映射的功能
         */
        FULL
    
    }
    ```

    - x

  - `Configuration.autoMappingBehavior` 属性，默认为 `AutoMappingBehavior.PARTIAL` 。

- `<6.2>` 处，调用 `#applyAutomaticMappings(...)` 方法，自动映射未明确的列。代码有点长，所以，详细解析，见 [「3.1.2.3.2.3 applyAutomaticMappings」](http://svip.iocoder.cn/MyBatis/executor-4/#) 。

- `<7>` 处，调用 `#applyPropertyMappings(...)` 方法，映射 ResultMap 中明确映射的列。代码有点长，所以，详细解析，见 [「3.1.2.3.2.4 applyPropertyMappings」](http://svip.iocoder.cn/MyBatis/executor-4/#) 。

- `<8>` 处，↑↑↑ 至此，当前 ResultSet 的该行记录的数据，已经完全映射到结果对象 `rowValue` 的对应属性中。😈 整个过程，非常非常非常长，胖友耐心理解和调试下。

- `<9>` 处，如果没有成功映射任意属性，则置空 rowValue 对象。当然，如果开启 `configuration.returnInstanceForEmptyRow` 属性，则不置空。默认情况下，该值为 `false` 。

######### 3.1.2.3.2.1 createResultObject

`#createResultObject(ResultSetWrapper rsw, ResultMap resultMap, ResultLoaderMap lazyLoader, String columnPrefix)` 方法，创建映射后的结果对象。代码如下：

```
// DefaultResultSetHandler.java

private Object createResultObject(ResultSetWrapper rsw, ResultMap resultMap, ResultLoaderMap lazyLoader, String columnPrefix) throws SQLException {
    // <1> useConstructorMappings ，表示是否使用构造方法创建该结果对象。此处将其重置
    this.useConstructorMappings = false; // reset previous mapping result
    final List<Class<?>> constructorArgTypes = new ArrayList<>(); // 记录使用的构造方法的参数类型的数组
    final List<Object> constructorArgs = new ArrayList<>(); // 记录使用的构造方法的参数值的数组
    // <2> 创建映射后的结果对象
    Object resultObject = createResultObject(rsw, resultMap, constructorArgTypes, constructorArgs, columnPrefix);
    if (resultObject != null && !hasTypeHandlerForResultObject(rsw, resultMap.getType())) {
        // <3> 如果有内嵌的查询，并且开启延迟加载，则创建结果对象的代理对象
        final List<ResultMapping> propertyMappings = resultMap.getPropertyResultMappings();
        for (ResultMapping propertyMapping : propertyMappings) {
            // issue gcode #109 && issue #149
            if (propertyMapping.getNestedQueryId() != null && propertyMapping.isLazy()) {
                resultObject = configuration.getProxyFactory().createProxy(resultObject, lazyLoader, configuration, objectFactory, constructorArgTypes, constructorArgs);
                break;
            }
        }
    }
    // <4> 判断是否使用构造方法创建该结果对象
    this.useConstructorMappings = resultObject != null && !constructorArgTypes.isEmpty(); // set current mapping result
    return resultObject;
}
```

- `<1>` 处，`useConstructorMappings` ，表示是否使用构造方法创建该结果对象。而此处，将其重置为 `false` 。
- `<2>` 处，调用 `#createResultObject(...)` 方法，创建映射后的结果对象。详细解析，往下看。😈 再次 mmp ，调用链太长了。
- `<3>` 处，如果有内嵌的查询，并且开启延迟加载，则调用 `ProxyFactory#createProxy(...)` 方法，创建结果对象的代理对象。详细解析，见 [《精尽 MyBatis 源码分析 —— SQL 执行（五）之延迟加载》](http://svip.iocoder.cn/MyBatis/executor-5) 。
- `<4>` 处，判断是否使用构造方法创建该结果对象，并设置到 `useConstructorMappings` 中。

------

`#createResultObject(ResultSetWrapper rsw, ResultMap resultMap, List<Class<?>> constructorArgTypes, List<Object> constructorArgs, String columnPrefix)` 方法，创建映射后的结果对象。代码如下：

```
// DefaultResultSetHandler.java

private Object createResultObject(ResultSetWrapper rsw, ResultMap resultMap, List<Class<?>> constructorArgTypes, List<Object> constructorArgs, String columnPrefix)
        throws SQLException {
    final Class<?> resultType = resultMap.getType();
    final MetaClass metaType = MetaClass.forClass(resultType, reflectorFactory);
    final List<ResultMapping> constructorMappings = resultMap.getConstructorResultMappings();
    // 下面，分成四种创建结果对象的情况
    // <1> 情况一，如果有对应的 TypeHandler 对象，则意味着是基本类型，直接创建对结果应对象
    if (hasTypeHandlerForResultObject(rsw, resultType)) {
        return createPrimitiveResultObject(rsw, resultMap, columnPrefix);
    // 情况二，如果 ResultMap 中，如果定义了 `<constructor />` 节点，则通过反射调用该构造方法，创建对应结果对象
    } else if (!constructorMappings.isEmpty()) {
        return createParameterizedResultObject(rsw, resultType, constructorMappings, constructorArgTypes, constructorArgs, columnPrefix);
    // 情况三，如果有默认的无参的构造方法，则使用该构造方法，创建对应结果对象
    } else if (resultType.isInterface() || metaType.hasDefaultConstructor()) {
        return objectFactory.create(resultType);
    // 情况四，通过自动映射的方式查找合适的构造方法，后使用该构造方法，创建对应结果对象
    } else if (shouldApplyAutomaticMappings(resultMap, false)) {
        return createByConstructorSignature(rsw, resultType, constructorArgTypes, constructorArgs, columnPrefix);
    }
    // 不支持，抛出 ExecutorException 异常
    throw new ExecutorException("Do not know how to create an instance of " + resultType);
}
```

- 分成四种创建结果对象的情况。

- `<1>` 处，情况一，如果有对应的 TypeHandler 对象，则意味着是基本类型，直接创建对结果应对象。调用 `#createPrimitiveResultObject(ResultSetWrapper rsw, ResultMap resultMap, String columnPrefix)` 方法，代码如下：

  ```
  // DefaultResultSetHandler.java
  
  private Object createPrimitiveResultObject(ResultSetWrapper rsw, ResultMap resultMap, String columnPrefix) throws SQLException {
      final Class<?> resultType = resultMap.getType();
      // 获得字段名
      final String columnName;
      if (!resultMap.getResultMappings().isEmpty()) {
          final List<ResultMapping> resultMappingList = resultMap.getResultMappings();
          final ResultMapping mapping = resultMappingList.get(0);
          columnName = prependPrefix(mapping.getColumn(), columnPrefix);
      } else {
          columnName = rsw.getColumnNames().get(0);
      }
      // 获得 TypeHandler 对象
      final TypeHandler<?> typeHandler = rsw.getTypeHandler(resultType, columnName);
      // 获得 ResultSet 的指定字段的值
      return typeHandler.getResult(rsw.getResultSet(), columnName);
  }
  ```

- `<2>` 处，情况二，如果 ResultMap 中，如果定义了 `<constructor />` 节点，则通过反射调用该构造方法，创建对应结果对象。调用 `#createParameterizedResultObject(ResultSetWrapper rsw, Class<?> resultType, List<ResultMapping> constructorMappings, List<Class<?>> constructorArgTypes, List<Object> constructorArgs, String columnPrefix)` 方法，代码如下：

  ```
  // DefaultResultSetHandler.java
  
  Object createParameterizedResultObject(ResultSetWrapper rsw, Class<?> resultType, List<ResultMapping> constructorMappings,
                                         List<Class<?>> constructorArgTypes, List<Object> constructorArgs, String columnPrefix) {
      // 获得到任一的属性值。即，只要一个结果对象，有一个属性非空，就会设置为 true
      boolean foundValues = false;
      for (ResultMapping constructorMapping : constructorMappings) {
          // 获得参数类型
          final Class<?> parameterType = constructorMapping.getJavaType();
          // 获得数据库的字段名
          final String column = constructorMapping.getColumn();
          // 获得属性值
          final Object value;
          try {
              // 如果是内嵌的查询，则获得内嵌的值
              if (constructorMapping.getNestedQueryId() != null) {
                  value = getNestedQueryConstructorValue(rsw.getResultSet(), constructorMapping, columnPrefix);
              // 如果是内嵌的 resultMap ，则递归 getRowValue 方法，获得对应的属性值
              } else if (constructorMapping.getNestedResultMapId() != null) {
                  final ResultMap resultMap = configuration.getResultMap(constructorMapping.getNestedResultMapId());
                  value = getRowValue(rsw, resultMap, constructorMapping.getColumnPrefix());
              // 最常用的情况，直接使用 TypeHandler 获取当前 ResultSet 的当前行的指定字段的值
              } else {
                  final TypeHandler<?> typeHandler = constructorMapping.getTypeHandler();
                  value = typeHandler.getResult(rsw.getResultSet(), prependPrefix(column, columnPrefix));
              }
          } catch (ResultMapException | SQLException e) {
              throw new ExecutorException("Could not process result for mapping: " + constructorMapping, e);
          }
          // 添加到 constructorArgTypes 和 constructorArgs 中
          constructorArgTypes.add(parameterType);
          constructorArgs.add(value);
          // 判断是否获得到属性值
          foundValues = value != null || foundValues;
      }
      // 查找 constructorArgTypes 对应的构造方法
      // 查找到后，传入 constructorArgs 作为参数，创建结果对象
      return foundValues ? objectFactory.create(resultType, constructorArgTypes, constructorArgs) : null;
  }
  ```

  - 代码比较简单，胖友看下注释即可。
  - 当然，里面的 `#getNestedQueryConstructorValue(ResultSet rs, ResultMapping constructorMapping, String columnPrefix)` 方法的逻辑，还是略微比较复杂的。所以，我们在讲完情况三、情况四，我们再来看看它的实现。😈 写到这里，艿艿的心里无比苦闷。详细解析，见 [「3.1.2.3.2.2 getNestedQueryConstructorValue」](http://svip.iocoder.cn/MyBatis/executor-4/#) 。

- `<3>` 处，情况三，如果有默认的无参的构造方法，则使用该构造方法，创建对应结果对象。

- `<4>` 处，情况四，通过自动映射的方式查找合适的构造方法，后使用该构造方法，创建对应结果对象。调用 `#createByConstructorSignature(ResultSetWrapper rsw, Class<?> resultType, List<Class<?>> constructorArgTypes, List<Object> constructorArgs, String columnPrefix)` 方法，代码如下：

  ```
  // DefaultResultSetHandler.java
  
  private Object createByConstructorSignature(ResultSetWrapper rsw, Class<?> resultType, List<Class<?>> constructorArgTypes, List<Object> constructorArgs,
                                              String columnPrefix) throws SQLException {
      // <1> 获得所有构造方法
      final Constructor<?>[] constructors = resultType.getDeclaredConstructors();
      // <2> 获得默认构造方法
      final Constructor<?> defaultConstructor = findDefaultConstructor(constructors);
      // <3> 如果有默认构造方法，使用该构造方法，创建结果对象
      if (defaultConstructor != null) {
          return createUsingConstructor(rsw, resultType, constructorArgTypes, constructorArgs, columnPrefix, defaultConstructor);
      } else {
          // <4> 遍历所有构造方法，查找符合的构造方法，创建结果对象
          for (Constructor<?> constructor : constructors) {
              if (allowedConstructorUsingTypeHandlers(constructor, rsw.getJdbcTypes())) {
                  return createUsingConstructor(rsw, resultType, constructorArgTypes, constructorArgs, columnPrefix, constructor);
              }
          }
      }
      throw new ExecutorException("No constructor found in " + resultType.getName() + " matching " + rsw.getClassNames());
  }
  ```

  - `<1>` 处，获得所有构造方法。

  - `<2>` 处，调用 `#findDefaultConstructor(final Constructor<?>[] constructors)` 方法，获得默认构造方法。代码如下：

    ```
    // DefaultResultSetHandler.java
    
    private Constructor<?> findDefaultConstructor(final Constructor<?>[] constructors) {
        // 构造方法只有一个，直接返回
        if (constructors.length == 1) return constructors[0];
        // 获得使用 @AutomapConstructor 注解的构造方法
        for (final Constructor<?> constructor : constructors) {
            if (constructor.isAnnotationPresent(AutomapConstructor.class)) {
                return constructor;
            }
        }
        return null;
    }
    ```

    - 两种情况，比较简单。

  - `<3>` 处，如果有默认构造方法，调用 `#createUsingConstructor(ResultSetWrapper rsw, Class<?> resultType, List<Class<?>> constructorArgTypes, List<Object> constructorArgs, String columnPrefix, Constructor<?> constructor)` 方法，使用该构造方法，创建结果对象。代码如下：

    ```
    // DefaultResultSetHandler.java
    
    private Object createUsingConstructor(ResultSetWrapper rsw, Class<?> resultType, List<Class<?>> constructorArgTypes, List<Object> constructorArgs, String columnPrefix, Constructor<?> constructor) throws SQLException {
        boolean foundValues = false;
        for (int i = 0; i < constructor.getParameterTypes().length; i++) {
            // 获得参数类型
            Class<?> parameterType = constructor.getParameterTypes()[i];
            // 获得数据库的字段名
            String columnName = rsw.getColumnNames().get(i);
            // 获得 TypeHandler 对象
            TypeHandler<?> typeHandler = rsw.getTypeHandler(parameterType, columnName);
            // 获取当前 ResultSet 的当前行的指定字段的值
            Object value = typeHandler.getResult(rsw.getResultSet(), prependPrefix(columnName, columnPrefix));
            // 添加到 constructorArgTypes 和 constructorArgs 中
            constructorArgTypes.add(parameterType);
            constructorArgs.add(value);
            // 判断是否获得到属性值
            foundValues = value != null || foundValues;
        }
        // 查找 constructorArgTypes 对应的构造方法
        // 查找到后，传入 constructorArgs 作为参数，创建结果对象
        return foundValues ? objectFactory.create(resultType, constructorArgTypes, constructorArgs) : null;
    }
    ```

    - 从代码实现上，和 `#createParameterizedResultObject(...)` 方法，类似。

  - `<4>` 处，遍历所有构造方法，调用 `#allowedConstructorUsingTypeHandlers(final Constructor<?> constructor, final List<JdbcType> jdbcTypes)` 方法，查找符合的构造方法，后创建结果对象。代码如下：

    ```
    // DefaultResultSetHandler.java
    
    private boolean allowedConstructorUsingTypeHandlers(final Constructor<?> constructor, final List<JdbcType> jdbcTypes) {
        final Class<?>[] parameterTypes = constructor.getParameterTypes();
        // 结果集的返回字段的数量，要和构造方法的参数数量，一致
        if (parameterTypes.length != jdbcTypes.size()) return false;
        // 每个构造方法的参数，和对应的返回字段，都要有对应的 TypeHandler 对象
        for (int i = 0; i < parameterTypes.length; i++) {
            if (!typeHandlerRegistry.hasTypeHandler(parameterTypes[i], jdbcTypes.get(i))) {
                return false;
            }
        }
        // 返回匹配
        return true;
    }
    ```

    - 基于结果集的返回字段和构造方法的参数做比较。

######### 3.1.2.3.2.2 getNestedQueryConstructorValue 嵌套查询

> 老艿艿：冲鸭！！！太冗长了！！！各种各种各种！！！！情况！！！！！

`#getNestedQueryConstructorValue(ResultSet rs, ResultMapping constructorMapping, String columnPrefix)` 方法，获得嵌套查询的值。代码如下：

```
// DefaultResultSetHandler.java

private Object getNestedQueryConstructorValue(ResultSet rs, ResultMapping constructorMapping, String columnPrefix) throws SQLException {
    // <1> 获得内嵌查询的编号
    final String nestedQueryId = constructorMapping.getNestedQueryId();
    // <1> 获得内嵌查询的 MappedStatement 对象
    final MappedStatement nestedQuery = configuration.getMappedStatement(nestedQueryId);
    // <1> 获得内嵌查询的参数类型
    final Class<?> nestedQueryParameterType = nestedQuery.getParameterMap().getType();
    // <2> 获得内嵌查询的参数对象
    final Object nestedQueryParameterObject = prepareParameterForNestedQuery(rs, constructorMapping, nestedQueryParameterType, columnPrefix);
    Object value = null;
    // <3> 执行查询
    if (nestedQueryParameterObject != null) {
        // <3.1> 获得 BoundSql 对象
        final BoundSql nestedBoundSql = nestedQuery.getBoundSql(nestedQueryParameterObject);
        // <3.2> 获得 CacheKey 对象
        final CacheKey key = executor.createCacheKey(nestedQuery, nestedQueryParameterObject, RowBounds.DEFAULT, nestedBoundSql);
        final Class<?> targetType = constructorMapping.getJavaType();
        // <3.3> 创建 ResultLoader 对象
        final ResultLoader resultLoader = new ResultLoader(configuration, executor, nestedQuery, nestedQueryParameterObject, targetType, key, nestedBoundSql);
        // <3.3> 加载结果
        value = resultLoader.loadResult();
    }
    return value;
}
```

- 关于这个方法，胖友可以调试 `BaseExecutorTest#shouldFetchOneOrphanedPostWithNoBlog()` 这个单元测试方法。

- `<1>` 处，获得内嵌查询的**编号**、**MappedStatement 对象**、**参数类型**。

- `<2>` 处，调用 `#prepareParameterForNestedQuery(ResultSet rs, ResultMapping resultMapping, Class<?> parameterType, String columnPrefix)` 方法，获得内嵌查询的**参数对象**。代码如下：

  ```
  // DefaultResultSetHandler.java
  
  // 获得内嵌查询的参数类型
  private Object prepareParameterForNestedQuery(ResultSet rs, ResultMapping resultMapping, Class<?> parameterType, String columnPrefix) throws SQLException {
      if (resultMapping.isCompositeResult()) { // ② 组合
          return prepareCompositeKeyParameter(rs, resultMapping, parameterType, columnPrefix);
      } else { // ① 普通
          return prepareSimpleKeyParameter(rs, resultMapping, parameterType, columnPrefix);
      }
  }
  
  // ① 获得普通类型的内嵌查询的参数对象
  private Object prepareSimpleKeyParameter(ResultSet rs, ResultMapping resultMapping, Class<?> parameterType, String columnPrefix) throws SQLException {
      // 获得 TypeHandler 对象
      final TypeHandler<?> typeHandler;
      if (typeHandlerRegistry.hasTypeHandler(parameterType)) {
          typeHandler = typeHandlerRegistry.getTypeHandler(parameterType);
      } else {
          typeHandler = typeHandlerRegistry.getUnknownTypeHandler();
      }
      // 获得指定字段的值
      return typeHandler.getResult(rs, prependPrefix(resultMapping.getColumn(), columnPrefix));
  }
  
  // ② 获得组合类型的内嵌查询的参数对象
  private Object prepareCompositeKeyParameter(ResultSet rs, ResultMapping resultMapping, Class<?> parameterType, String columnPrefix) throws SQLException {
      // 创建参数对象
      final Object parameterObject = instantiateParameterObject(parameterType);
      // 创建参数对象的 MetaObject 对象，可对其进行访问
      final MetaObject metaObject = configuration.newMetaObject(parameterObject);
      boolean foundValues = false;
      // 遍历组合的所有字段
      for (ResultMapping innerResultMapping : resultMapping.getComposites()) {
          // 获得属性类型
          final Class<?> propType = metaObject.getSetterType(innerResultMapping.getProperty());
          // 获得对应的 TypeHandler 对象
          final TypeHandler<?> typeHandler = typeHandlerRegistry.getTypeHandler(propType);
          // 获得指定字段的值
          final Object propValue = typeHandler.getResult(rs, prependPrefix(innerResultMapping.getColumn(), columnPrefix));
          // issue #353 & #560 do not execute nested query if key is null
          // 设置到 parameterObject 中，通过 metaObject
          if (propValue != null) {
              metaObject.setValue(innerResultMapping.getProperty(), propValue);
              foundValues = true; // 标记 parameterObject 非空对象
          }
      }
      // 返回参数对象
      return foundValues ? parameterObject : null;
  }
  
  // ② 创建参数对象
  private Object instantiateParameterObject(Class<?> parameterType) {
      if (parameterType == null) {
          return new HashMap<>();
      } else if (ParamMap.class.equals(parameterType)) {
          return new HashMap<>(); // issue #649
      } else {
          return objectFactory.create(parameterType);
      }
  }
  ```

  - 虽然代码比较长，但是非常简单。注意下，艿艿添加了 `①` 和 `②` 两个序号，分别对应两种情况。

- ```
  <3>
  ```

   

  处，整体是，执行查询，获得值。

  - `<3.1>` 处，调用 `MappedStatement#getBoundSql(Object parameterObject)` 方法，获得 BoundSql 对象。
  - `<3.2>` 处，创建 CacheKey 对象。
  - `<3.3>` 处，创建 ResultLoader 对象，并调用 `ResultLoader#loadResult()` 方法，加载结果。详细解析，见 [《精尽 MyBatis 源码分析 —— SQL 执行（五）之延迟加载》](http://svip.iocoder.cn/MyBatis/executor-5) 。

######### 3.1.2.3.2.3 applyAutomaticMappings

`#createResultObject(ResultSetWrapper rsw, ResultMap resultMap, List<Class<?>> constructorArgTypes, List<Object> constructorArgs, String columnPrefix)` 方法，创建映射后的结果对象。代码如下：

```
// DefaultResultSetHandler.java

private boolean applyAutomaticMappings(ResultSetWrapper rsw, ResultMap resultMap, MetaObject metaObject, String columnPrefix) throws SQLException {
    // <1> 获得 UnMappedColumnAutoMapping 数组
    List<UnMappedColumnAutoMapping> autoMapping = createAutomaticMappings(rsw, resultMap, metaObject, columnPrefix);
    boolean foundValues = false;
    if (!autoMapping.isEmpty()) {
        // <2> 遍历 UnMappedColumnAutoMapping 数组
        for (UnMappedColumnAutoMapping mapping : autoMapping) {
            // 获得指定字段的值
            final Object value = mapping.typeHandler.getResult(rsw.getResultSet(), mapping.column);
            // 若非空，标记 foundValues 有值
            if (value != null) {
                foundValues = true;
            }
            // 设置到 parameterObject 中，通过 metaObject
            if (value != null || (configuration.isCallSettersOnNulls() && !mapping.primitive)) {
                // gcode issue #377, call setter on nulls (value is not 'found')
                metaObject.setValue(mapping.property, value);
            }
        }
    }
    return foundValues;
}
```

- `<1>` 处，调用 `#createAutomaticMappings(ResultSetWrapper rsw, ResultMap resultMap, MetaObject metaObject, String columnPrefix)` 方法，获得 UnMappedColumnAutoMapping 数组。代码如下：

  ```
  // DefaultResultSetHandler.java
  
  private List<UnMappedColumnAutoMapping> createAutomaticMappings(ResultSetWrapper rsw, ResultMap resultMap, MetaObject metaObject, String columnPrefix) throws SQLException {
      // 生成 autoMappingsCache 的 KEY
      final String mapKey = resultMap.getId() + ":" + columnPrefix;
      // 从缓存 autoMappingsCache 中，获得 UnMappedColumnAutoMapping 数组
      List<UnMappedColumnAutoMapping> autoMapping = autoMappingsCache.get(mapKey);
      // 如果获取不到，则进行初始化
      if (autoMapping == null) {
          autoMapping = new ArrayList<>();
          // 获得未 mapped 的字段的名字的数组
          final List<String> unmappedColumnNames = rsw.getUnmappedColumnNames(resultMap, columnPrefix);
          // 遍历 unmappedColumnNames 数组
          for (String columnName : unmappedColumnNames) {
              // 获得属性名
              String propertyName = columnName;
              if (columnPrefix != null && !columnPrefix.isEmpty()) {
                  // When columnPrefix is specified,
                  // ignore columns without the prefix.
                  if (columnName.toUpperCase(Locale.ENGLISH).startsWith(columnPrefix)) {
                      propertyName = columnName.substring(columnPrefix.length());
                  } else {
                      continue;
                  }
              }
              // 从结果对象的 metaObject 中，获得对应的属性名
              final String property = metaObject.findProperty(propertyName, configuration.isMapUnderscoreToCamelCase());
              // 获得到属性名，并且可以进行设置
              if (property != null && metaObject.hasSetter(property)) {
                  // 排除已映射的属性
                  if (resultMap.getMappedProperties().contains(property)) {
                      continue;
                  }
                  // 获得属性的类型
                  final Class<?> propertyType = metaObject.getSetterType(property);
                  // 判断是否有对应的 TypeHandler 对象。如果有，则创建 UnMappedColumnAutoMapping 对象，并添加到 autoMapping 中
                  if (typeHandlerRegistry.hasTypeHandler(propertyType, rsw.getJdbcType(columnName))) {
                      final TypeHandler<?> typeHandler = rsw.getTypeHandler(propertyType, columnName);
                      autoMapping.add(new UnMappedColumnAutoMapping(columnName, property, typeHandler, propertyType.isPrimitive()));
                  // 如果没有，则执行 AutoMappingUnknownColumnBehavior 对应的逻辑
                  } else {
                      configuration.getAutoMappingUnknownColumnBehavior()
                              .doAction(mappedStatement, columnName, property, propertyType);
                  }
              // 如果没有属性，或者无法设置，则则执行 AutoMappingUnknownColumnBehavior 对应的逻辑
              } else {
                  configuration.getAutoMappingUnknownColumnBehavior()
                          .doAction(mappedStatement, columnName, (property != null) ? property : propertyName, null);
              }
          }
          // 添加到缓存中
          autoMappingsCache.put(mapKey, autoMapping);
      }
      return autoMapping;
  }
  ```

  - 虽然代码比较长，但是逻辑很简单。遍历未 mapped 的字段的名字的数组，映射每一个字段在**结果对象**的相同名字的属性，最终生成 UnMappedColumnAutoMapping 对象。

  - UnMappedColumnAutoMapping ，是 DefaultResultSetHandler 的内部静态类，未 mapped 字段自动映射后的对象。代码如下：

    ```
    // DefaultResultSetHandler.java
    
    private static class UnMappedColumnAutoMapping {
    
        /**
         * 字段名
         */
        private final String column;
        /**
         * 属性名
         */
        private final String property;
        /**
         * TypeHandler 处理器
         */
        private final TypeHandler<?> typeHandler;
        /**
         * 是否为基本属性
         */
        private final boolean primitive;
    
        public UnMappedColumnAutoMapping(String column, String property, TypeHandler<?> typeHandler, boolean primitive) {
            this.column = column;
            this.property = property;
            this.typeHandler = typeHandler;
            this.primitive = primitive;
        }
    
    }
    ```

    - x

  - 当找不到映射的属性时，会调用 `AutoMappingUnknownColumnBehavior#doAction(MappedStatement mappedStatement, String columnName, String propertyName, Class<?> propertyType)` 方法，执行相应的逻辑。比较简单，胖友直接看 [`org.apache.ibatis.session.AutoMappingUnknownColumnBehavior`](https://github.com/YunaiV/mybatis-3/blob/master/src/main/java/org/apache/ibatis/session/AutoMappingUnknownColumnBehavior.java) 即可。

  - `Configuration.autoMappingUnknownColumnBehavior` 为 `AutoMappingUnknownColumnBehavior.NONE` ，即**不处理**。

- `<2>` 处，遍历 UnMappedColumnAutoMapping 数组，获得指定字段的值，设置到 `parameterObject` 中，通过 `metaObject` 。

######### 3.1.2.3.2.4 applyPropertyMappings

`#applyPropertyMappings(ResultSetWrapper rsw, ResultMap resultMap, MetaObject metaObject, ResultLoaderMap lazyLoader, String columnPrefix)` 方法，映射 ResultMap 中明确映射的列。代码如下：

```
// DefaultResultSetHandler.java

private boolean applyPropertyMappings(ResultSetWrapper rsw, ResultMap resultMap, MetaObject metaObject, ResultLoaderMap lazyLoader, String columnPrefix)
        throws SQLException {
    // 获得 mapped 的字段的名字的数组
    final List<String> mappedColumnNames = rsw.getMappedColumnNames(resultMap, columnPrefix);
    boolean foundValues = false;
    // 遍历 ResultMapping 数组
    final List<ResultMapping> propertyMappings = resultMap.getPropertyResultMappings();
    for (ResultMapping propertyMapping : propertyMappings) {
        // 获得字段名
        String column = prependPrefix(propertyMapping.getColumn(), columnPrefix);
        if (propertyMapping.getNestedResultMapId() != null) {
            // the user added a column attribute to a nested result map, ignore it
            column = null;
        }
        if (propertyMapping.isCompositeResult() // 组合
                || (column != null && mappedColumnNames.contains(column.toUpperCase(Locale.ENGLISH))) // 属于 mappedColumnNames
                || propertyMapping.getResultSet() != null) { // 存储过程
            // <1> 获得指定字段的值
            Object value = getPropertyMappingValue(rsw.getResultSet(), metaObject, propertyMapping, lazyLoader, columnPrefix);
            // issue #541 make property optional
            final String property = propertyMapping.getProperty();
            if (property == null) {
                continue;
            // 存储过程相关，忽略
            } else if (value == DEFERED) {
                foundValues = true;
                continue;
            }
            // 标记获取到任一属性
            if (value != null) {
                foundValues = true;
            }
            // 设置到 parameterObject 中，通过 metaObject
            if (value != null || (configuration.isCallSettersOnNulls() && !metaObject.getSetterType(property).isPrimitive())) {
                // gcode issue #377, call setter on nulls (value is not 'found')
                metaObject.setValue(property, value);
            }
        }
    }
    return foundValues;
}
```

- 虽然代码比较长，但是逻辑很简单。胖友自己瞅瞅。

- 在 `<1>` 处，调用 `#getPropertyMappingValue(ResultSet rs, MetaObject metaResultObject, ResultMapping propertyMapping, ResultLoaderMap lazyLoader, String columnPrefix)` 方法，获得指定字段的值。代码如下：

  ```
  // DefaultResultSetHandler.java
  
  private Object getPropertyMappingValue(ResultSet rs, MetaObject metaResultObject, ResultMapping propertyMapping, ResultLoaderMap lazyLoader, String columnPrefix)
          throws SQLException {
      // <2> 内嵌查询，获得嵌套查询的值
      if (propertyMapping.getNestedQueryId() != null) {
          return getNestedQueryMappingValue(rs, metaResultObject, propertyMapping, lazyLoader, columnPrefix);
      // 存储过程相关，忽略
      } else if (propertyMapping.getResultSet() != null) {
          addPendingChildRelation(rs, metaResultObject, propertyMapping);   // TODO is that OK?
          return DEFERED;
      // 普通，直接获得指定字段的值
      } else {
          final TypeHandler<?> typeHandler = propertyMapping.getTypeHandler();
          final String column = prependPrefix(propertyMapping.getColumn(), columnPrefix);
          return typeHandler.getResult(rs, column);
      }
  }
  ```

  - 在 `<2>` 处，我们又碰到了一个内嵌查询，调用 `#getNestedQueryMappingValue(ResultSet rs, MetaObject metaResultObject, ResultMapping propertyMapping, ResultLoaderMap lazyLoader, String columnPrefix)` 方法，获得嵌套查询的值。详细解析，见 [「」](http://svip.iocoder.cn/MyBatis/executor-4/#) 。

######### 3.1.2.3.2.5 getNestedQueryMappingValue 嵌套查询

`#getNestedQueryMappingValue(ResultSet rs, MetaObject metaResultObject, ResultMapping propertyMapping, ResultLoaderMap lazyLoader, String columnPrefix)` 方法，获得嵌套查询的值。代码如下：

```
// DefaultResultSetHandler.java

private Object getNestedQueryMappingValue(ResultSet rs, MetaObject metaResultObject, ResultMapping propertyMapping, ResultLoaderMap lazyLoader, String columnPrefix)
        throws SQLException {
    // 获得内嵌查询的编号
    final String nestedQueryId = propertyMapping.getNestedQueryId();
    // 获得属性名
    final String property = propertyMapping.getProperty();
    // 获得内嵌查询的 MappedStatement 对象
    final MappedStatement nestedQuery = configuration.getMappedStatement(nestedQueryId);
    // 获得内嵌查询的参数类型
    final Class<?> nestedQueryParameterType = nestedQuery.getParameterMap().getType();
    // 获得内嵌查询的参数对象
    final Object nestedQueryParameterObject = prepareParameterForNestedQuery(rs, propertyMapping, nestedQueryParameterType, columnPrefix);
    Object value = null;
    if (nestedQueryParameterObject != null) {
        // 获得 BoundSql 对象
        final BoundSql nestedBoundSql = nestedQuery.getBoundSql(nestedQueryParameterObject);
        // 获得 CacheKey 对象
        final CacheKey key = executor.createCacheKey(nestedQuery, nestedQueryParameterObject, RowBounds.DEFAULT, nestedBoundSql);
        final Class<?> targetType = propertyMapping.getJavaType();
        // <1> 检查缓存中已存在
        if (executor.isCached(nestedQuery, key)) { //  有缓存
            // <2.1> 创建 DeferredLoad 对象，并通过该 DeferredLoad 对象从缓存中加载结采对象
            executor.deferLoad(nestedQuery, metaResultObject, property, key, targetType);
            // <2.2> 返回已定义
            value = DEFERED;
        // 检查缓存中不存在
        } else { // 无缓存
            // <3.1> 创建 ResultLoader 对象
            final ResultLoader resultLoader = new ResultLoader(configuration, executor, nestedQuery, nestedQueryParameterObject, targetType, key, nestedBoundSql);
            // <3.2> 如果要求延迟加载，则延迟加载
            if (propertyMapping.isLazy()) {
                // 如果该属性配置了延迟加载，则将其添加到 `ResultLoader.loaderMap` 中，等待真正使用时再执行嵌套查询并得到结果对象。
                lazyLoader.addLoader(property, metaResultObject, resultLoader);
                // 返回已定义
                value = DEFERED;
            // <3.3> 如果不要求延迟加载，则直接执行加载对应的值
            } else {
                value = resultLoader.loadResult();
            }
        }
    }
    return value;
}
```

- 和

   

  「3.1.2.3.2.2 getNestedQueryConstructorValue」

   

  一样，也是

  嵌套查询

  。所以，从整体代码的实现上，也是非常

  类似

  的。差别在于：

  - `#getNestedQueryConstructorValue(...)` 方法，用于构造方法需要用到的嵌套查询的值，它是**不用**考虑延迟加载的。
  - `#getNestedQueryMappingValue(...)` 方法，用于 setting 方法需要用到的嵌套查询的值，它是**需要**考虑延迟加载的。

- `<1>` 处，调用 `Executor#isCached(MappedStatement ms, CacheKey key)` 方法，检查缓存中已存在。下面，我们分成两种情况来解析。

- ========== 有缓存 ==========

- `<2.1>` 处，调用 `Executor#deferLoad(...)` 方法，创建 DeferredLoad 对象，并通过该 DeferredLoad 对象从缓存中加载结采对象。详细解析，见 [《精尽 MyBatis 源码分析 —— SQL 执行（五）之延迟加载》](http://svip.iocoder.cn/MyBatis/executor-5) 。

- `<2.2>` 处，返回已定义 `DEFERED` 。

- ========== 有缓存 ==========

- `<3.1>` 处，创建 ResultLoader 对象。

- ```
  <3.2>
  ```

   

  处，如果要求延迟加载，则延迟加载。

  - `<3.2.1>` 处，调用 `ResultLoader#addLoader(...)` 方法，如果该属性配置了延迟加载，则将其添加到 `ResultLoader.loaderMap` 中，等待真正使用时再执行嵌套查询并得到结果对象。详细解析，见 [《精尽 MyBatis 源码分析 —— SQL 执行（五）之延迟加载》](http://svip.iocoder.cn/MyBatis/executor-5) 。

- `<3.3>` 处，如果不要求延迟加载，则调用 `ResultLoader#loadResult()` 方法，直接执行加载对应的值。

######## 3.1.2.3.3 storeObject

`#storeObject(ResultHandler<?> resultHandler, DefaultResultContext<Object> resultContext, Object rowValue, ResultMapping parentMapping, ResultSet rs)` 方法，将映射创建的结果对象添加到 `ResultHandler.resultList` 中保存。代码如下：

```
// DefaultResultSetHandler.java

private void storeObject(ResultHandler<?> resultHandler, DefaultResultContext<Object> resultContext, Object rowValue, ResultMapping parentMapping, ResultSet rs) throws SQLException {
    // 暂时忽略，这个情况，只有存储过程会出现
    if (parentMapping != null) {
        linkToParents(rs, parentMapping, rowValue);
    } else {
        callResultHandler(resultHandler, resultContext, rowValue);
    }
}

@SuppressWarnings("unchecked" /* because ResultHandler<?> is always ResultHandler<Object>*/)
// 调用 ResultHandler ，进行结果的处理
private void callResultHandler(ResultHandler<?> resultHandler, DefaultResultContext<Object> resultContext, Object rowValue) {
    // 设置结果对象到 resultContext 中
    resultContext.nextResultObject(rowValue);
    // <x> 使用 ResultHandler 处理结果。
    // 如果使用 DefaultResultHandler 实现类的情况，会将映射创建的结果对象添加到 ResultHandler.resultList 中保存
    ((ResultHandler<Object>) resultHandler).handleResult(resultContext);
}
```

- 逻辑比较简单，认真看下注释，特别是 `<x>` 处。

####### 3.1.2.4 handleRowValuesForNestedResultMap 嵌套映射

可能胖友对**嵌套映射**的概念不是很熟悉，胖友可以调试 `AncestorRefTest#testAncestorRef()` 这个单元测试方法。

------

> 老艿艿：本小节，还是建议看 [《MyBatis 技术内幕》](https://item.jd.com/12125531.html) 的 [「3.3.4 嵌套映射」](http://svip.iocoder.cn/MyBatis/executor-4/#) 小节。因为，它提供了比较好的这块逻辑的原理讲解，并且配置了大量的图。
>
> 😈 精力有限，后续补充哈。
> 😜 实际是，因为艿艿比较少用嵌套映射，所以对这块逻辑，不是很感兴趣。

`#handleRowValuesForNestedResultMap(ResultSetWrapper rsw, ResultMap resultMap, ResultHandler<?> resultHandler, RowBounds rowBounds, ResultMapping parentMapping)` 方法，处理嵌套**映射**的结果。代码如下：

```
// DefaultResultSetHandler.java
```

- TODO 9999 状态不太好，有点写不太明白。不感兴趣的胖友，可以跳过。感兴趣的胖友，可以看看 [《MyBatis 技术内幕》](https://item.jd.com/12125531.html) 的 [「3.3.4 嵌套映射」](http://svip.iocoder.cn/MyBatis/executor-4/#) 小节。

###### 3.1.3 handleCursorResultSets

`#handleCursorResultSets(Statement stmt)` 方法，处理 `java.sql.ResultSet` 成 Cursor 对象。代码如下：

```
// DefaultResultSetHandler.java

@Override
public <E> Cursor<E> handleCursorResultSets(Statement stmt) throws SQLException {
    ErrorContext.instance().activity("handling cursor results").object(mappedStatement.getId());

    // 获得首个 ResultSet 对象，并封装成 ResultSetWrapper 对象
    ResultSetWrapper rsw = getFirstResultSet(stmt);

    // 游标方式的查询，只允许一个 ResultSet 对象。因此，resultMaps 数组的数量，元素只能有一个
    List<ResultMap> resultMaps = mappedStatement.getResultMaps();
    int resultMapCount = resultMaps.size();
    validateResultMapsCount(rsw, resultMapCount);
    if (resultMapCount != 1) {
        throw new ExecutorException("Cursor results cannot be mapped to multiple resultMaps");
    }

    // 获得 ResultMap 对象，后创建 DefaultCursor 对象
    ResultMap resultMap = resultMaps.get(0);
    return new DefaultCursor<>(this, resultMap, rsw, rowBounds);
}
```

- 最终，创建成 DefaultCursor 对象。详细解析，见 [「6. DefaultCursor」](http://svip.iocoder.cn/MyBatis/executor-4/#) 。
- 可能很多人没用 MyBatis Cursor 功能，所以可以看看 [《Mybatis 3.4.0 Cursor的使用》](https://www.jianshu.com/p/97d96201295b) 。

## 4. ResultContext

> 老艿艿：这个类，大体看看每个方法的用途，结合上文一起理解即可。

`org.apache.ibatis.session.ResultContext` ，结果上下文接口。代码如下：

```
// ResultContext.java

public interface ResultContext<T> {

    /**
     * @return 当前结果对象
     */
    T getResultObject();

    /**
     * @return 总的结果对象的数量
     */
    int getResultCount();

    /**
     * @return 是否暂停
     */
    boolean isStopped();

    /**
     * 暂停
     */
    void stop();

}
```

#### 4.1 DefaultResultContext

`org.apache.ibatis.executor.result.DefaultResultContext` ，实现 ResultContext 接口，默认的 ResultContext 的实现类。代码如下：

```
// DefaultResultContext.java

public class DefaultResultContext<T> implements ResultContext<T> {

    /**
     * 当前结果对象
     */
    private T resultObject;
    /**
     * 总的结果对象的数量
     */
    private int resultCount;
    /**
     * 是否暂停
     */
    private boolean stopped;

    public DefaultResultContext() {
        resultObject = null;
        resultCount = 0;
        stopped = false; // 默认非暂停
    }

    @Override
    public T getResultObject() {
        return resultObject;
    }

    @Override
    public int getResultCount() {
        return resultCount;
    }

    @Override
    public boolean isStopped() {
        return stopped;
    }

    /**
     * 当前结果对象
     *
     * @param resultObject 当前结果对象
     */
    public void nextResultObject(T resultObject) {
        // 数量 + 1
        resultCount++;
        // 当前结果对象
        this.resultObject = resultObject;
    }

    @Override
    public void stop() {
        this.stopped = true;
    }

}
```

## 5. ResultHandler

`org.apache.ibatis.session.ResultHandler` ，结果处理器接口。代码如下：

```
// ResultHandler.java

public interface ResultHandler<T> {

    /**
     * 处理当前结果
     *
     * @param resultContext ResultContext 对象。在其中，可以获得当前结果
     */
    void handleResult(ResultContext<? extends T> resultContext);

}
```

#### 5.1 DefaultResultHandler

`org.apache.ibatis.executor.result.DefaultResultHandler` ，实现 ResultHandler 接口，默认的 ResultHandler 的实现类。代码如下：

```
// DefaultResultHandler.java

public class DefaultResultHandler implements ResultHandler<Object> {

    /**
     * 结果数组
     */
    private final List<Object> list;

    public DefaultResultHandler() {
        list = new ArrayList<>();
    }

    @SuppressWarnings("unchecked")
    public DefaultResultHandler(ObjectFactory objectFactory) {
        list = objectFactory.create(List.class);
    }

    @Override
    public void handleResult(ResultContext<? extends Object> context) {
        // <1> 将当前结果，添加到结果数组中 
        list.add(context.getResultObject());
    }

    public List<Object> getResultList() {
        return list;
    }

}
```

- 核心代码就是 `<1>` 处，将当前结果，添加到结果数组中。

#### 5.2 DefaultMapResultHandler

该类在 `session` 包中实现，我们放在**会话模块**的文章中，详细解析。

## 6. Cursor

`org.apache.ibatis.cursor.Cursor` ，继承 Closeable、Iterable 接口，游标接口。代码如下：

```
// Cursor.java

public interface Cursor<T> extends Closeable, Iterable<T> {

    /**
     * 是否处于打开状态
     *
     * @return true if the cursor has started to fetch items from database.
     */
    boolean isOpen();

    /**
     * 是否全部消费完成
     *
     * @return true if the cursor is fully consumed and has returned all elements matching the query.
     */
    boolean isConsumed();

    /**
     * 获得当前索引
     *
     * Get the current item index. The first item has the index 0.
     * @return -1 if the first cursor item has not been retrieved. The index of the current item retrieved.
     */
    int getCurrentIndex();

}
```

#### 6.1 DefaultCursor

`org.apache.ibatis.cursor.defaults.DefaultCursor` ，实现 Cursor 接口，默认 Cursor 实现类。

###### 6.1.1 构造方法

```
// DefaultCursor.java

// ResultSetHandler stuff
private final DefaultResultSetHandler resultSetHandler;
private final ResultMap resultMap;
private final ResultSetWrapper rsw;
private final RowBounds rowBounds;
/**
 * ObjectWrapperResultHandler 对象
 */
private final ObjectWrapperResultHandler<T> objectWrapperResultHandler = new ObjectWrapperResultHandler<>();

/**
 * CursorIterator 对象，游标迭代器。
 */
private final CursorIterator cursorIterator = new CursorIterator();
/**
 * 是否开始迭代
 *
 * {@link #iterator()}
 */
private boolean iteratorRetrieved;
/**
 * 游标状态
 */
private CursorStatus status = CursorStatus.CREATED;
/**
 * 已完成映射的行数
 */
private int indexWithRowBound = -1;

public DefaultCursor(DefaultResultSetHandler resultSetHandler, ResultMap resultMap, ResultSetWrapper rsw, RowBounds rowBounds) {
    this.resultSetHandler = resultSetHandler;
    this.resultMap = resultMap;
    this.rsw = rsw;
    this.rowBounds = rowBounds;
}
```

- 大体瞄下每个属性的意思。下面，每个方法，胖友会更好的理解每个属性。

###### 6.1.2 CursorStatus

CursorStatus ，是 DefaultCursor 的内部枚举类。代码如下：

```
// DefaultCursor.java

private enum CursorStatus {

    /**
     * A freshly created cursor, database ResultSet consuming has not started
     */
    CREATED,
    /**
     * A cursor currently in use, database ResultSet consuming has started
     */
    OPEN,
    /**
     * A closed cursor, not fully consumed
     *
     * 已关闭，并未完全消费
     */
    CLOSED,
    /**
     * A fully consumed cursor, a consumed cursor is always closed
     *
     * 已关闭，并且完全消费
     */
    CONSUMED
}
```

###### 6.1.3 isOpen

```
// DefaultCursor.java

@Override
public boolean isOpen() {
    return status == CursorStatus.OPEN;
}
```

###### 6.1.4 isConsumed

```
// DefaultCursor.java

@Override
public boolean isConsumed() {
    return status == CursorStatus.CONSUMED;
}
```

###### 6.1.5 iterator

`#iterator()` 方法，获取迭代器。代码如下：

```
// DefaultCursor.java

@Override
public Iterator<T> iterator() {
    // 如果已经获取，则抛出 IllegalStateException 异常
    if (iteratorRetrieved) {
        throw new IllegalStateException("Cannot open more than one iterator on a Cursor");
    }
    if (isClosed()) {
        throw new IllegalStateException("A Cursor is already closed.");
    }
    // 标记已经获取
    iteratorRetrieved = true;
    return cursorIterator;
}
```

- 通过 `iteratorRetrieved` 属性，保证有且仅返回一次 `cursorIterator` 对象。

###### 6.1.6 ObjectWrapperResultHandler

ObjectWrapperResultHandler ，DefaultCursor 的内部静态类，实现 ResultHandler 接口，代码如下：

```
// DefaultCursor.java

private static class ObjectWrapperResultHandler<T> implements ResultHandler<T> {

    /**
     * 结果对象
     */
    private T result;

    @Override
    public void handleResult(ResultContext<? extends T> context) {
        // <1> 设置结果对象
        this.result = context.getResultObject();
        // <2> 暂停
        context.stop();
    }

}
```

- `<1>` 处，暂存 [「3.1 DefaultResultSetHandler」](http://svip.iocoder.cn/MyBatis/executor-4/#) 处理的 ResultSet 的**当前行**的结果。

- `<2>` 处，通过调用 `ResultContext#stop()` 方法，暂停 DefaultResultSetHandler 在向下遍历下一条记录，从而实现每次在调用 `CursorIterator#hasNext()` 方法，只遍历一行 ResultSet 的记录。如果胖友有点懵逼，可以在看看 `DefaultResultSetHandler#shouldProcessMoreRows(ResultContext<?> context, RowBounds rowBounds)` 方法，代码如下：

  ```
  // DefaultCursor.java
  
  private boolean shouldProcessMoreRows(ResultContext<?> context, RowBounds rowBounds) {
      return !context.isStopped() && context.getResultCount() < rowBounds.getLimit();
  }
  ```

###### 6.1.7 CursorIterator

CursorIterator ，DefaultCursor 的内部类，实现 Iterator 接口，游标的迭代器实现类。代码如下：

```
// DefaultCursor.java

private class CursorIterator implements Iterator<T> {

    /**
     * Holder for the next object to be returned
     *
     * 结果对象，提供给 {@link #next()} 返回
     */
    T object;

    /**
     * Index of objects returned using next(), and as such, visible to users.
     * 索引位置
     */
    int iteratorIndex = -1;

    @Override
    public boolean hasNext() {
        // <1> 如果 object 为空，则遍历下一条记录
        if (object == null) {
            object = fetchNextUsingRowBound();
        }
        // <2> 判断 object 是否非空
        return object != null;
    }

    @Override
    public T next() {
        // <3> Fill next with object fetched from hasNext()
        T next = object;

        // <4> 如果 next 为空，则遍历下一条记录
        if (next == null) {
            next = fetchNextUsingRowBound();
        }

        // <5> 如果 next 非空，说明有记录，则进行返回
        if (next != null) {
            // <5.1> 置空 object 对象
            object = null;
            // <5.2> 增加 iteratorIndex
            iteratorIndex++;
            // <5.3> 返回 next
            return next;
        }

        // <6> 如果 next 为空，说明没有记录，抛出 NoSuchElementException 异常
        throw new NoSuchElementException();
    }

    @Override
    public void remove() {
        throw new UnsupportedOperationException("Cannot remove element from Cursor");
    }

}
```

- ```
  #hasNext()
  ```

   

  方法，判断是否有

  下一个

  结果对象：

  - `<1>` 处，如果 `object` 为空，则调用 `#fetchNextUsingRowBound()` 方法，遍历下一条记录。也就是说，该方法是先获取下一条记录，然后在判断是否存在下一条记录。实际上，和 `java.util.ResultSet` 的方式，是一致的。如果再调用一次该方法，则不会去遍历下一条记录。关于 `#fetchNextUsingRowBound()` 方法，详细解析，见 [「6.1.8 fetchNextUsingRowBound」](http://svip.iocoder.cn/MyBatis/executor-4/#) 。
  - `<2>` 处，判断 `object` 非空。

- ```
  #next()
  ```

   

  方法，获得

  下一个

  结果对象：

  - `<3>` 处，先记录 `object` 到 `next` 中。为什么要这么做呢？继续往下看。

  - `<4>` 处，如果 `next` 为空，有两种可能性：1）使用方未调用 `#hasNext()` 方法；2）调用 `#hasNext()` 方法，发现没下一条，还是调用了 `#next()` 方法。如果 `next()` 方法为空，通过“再次”调用 `#fetchNextUsingRowBound()` 方法，去遍历下一条记录。

  - ```
    <5>
    ```

     

    处，如果

     

    ```
    next
    ```

     

    非空，说明有记录，则进行返回。

    - `<5.1>` 处，置空 `object` 对象。
    - `<5.2>` 处，增加 `iteratorIndex` 。
    - `<5.3>` 处，返回 `next` 。如果 `<3>` 处，不进行 `next` 的赋值，如果 `<5.1>` 处的置空，此处就无法返回 `next` 了。

  - `<6>` 处，如果 `next` 为空，说明没有记录，抛出 NoSuchElementException 异常。

###### 6.1.8 fetchNextUsingRowBound

`#fetchNextUsingRowBound()` 方法，遍历下一条记录。代码如下：

```
// DefaultCursor.java

protected T fetchNextUsingRowBound() {
    // <1> 遍历下一条记录
    T result = fetchNextObjectFromDatabase();
    // 循环跳过 rowBounds 的索引
    while (result != null && indexWithRowBound < rowBounds.getOffset()) {
        result = fetchNextObjectFromDatabase();
    }
    // 返回记录
    return result;
}
```

- `<1>` 处，调用 `#fetchNextObjectFromDatabase()` 方法，遍历下一条记录。代码如下：

```
// DefaultCursor.java

protected T fetchNextObjectFromDatabase() {
    // <1> 如果已经关闭，返回 null
    if (isClosed()) {
        return null;
    }

    try {
        // <2> 设置状态为 CursorStatus.OPEN
        status = CursorStatus.OPEN;
        // <3> 遍历下一条记录
        if (!rsw.getResultSet().isClosed()) {
            resultSetHandler.handleRowValues(rsw, resultMap, objectWrapperResultHandler, RowBounds.DEFAULT, null);
        }
    } catch (SQLException e) {
        throw new RuntimeException(e);
    }

    // <4> 复制给 next
    T next = objectWrapperResultHandler.result;
    // <5> 增加 indexWithRowBound
    if (next != null) {
        indexWithRowBound++;
    }
    // No more object or limit reached
    // <6> 没有更多记录，或者到达 rowBounds 的限制索引位置，则关闭游标，并设置状态为 CursorStatus.CONSUMED
    if (next == null || getReadItemsCount() == rowBounds.getOffset() + rowBounds.getLimit()) {
        close();
        status = CursorStatus.CONSUMED;
    }
    // <7> 置空 objectWrapperResultHandler.result 属性
    objectWrapperResultHandler.result = null;
    // <8> 返回下一条结果
    return next;
}
```

- `<1>` 处，调用 `#isClosed()` 方法，判断是否已经关闭。若是，则返回 `null` 。代码如下：

  ```
  // DefaultCursor.java
  
  private boolean isClosed() {
      return status == CursorStatus.CLOSED || status == CursorStatus.CONSUMED;
  }
  ```

- `<2>` 处，设置状态为 `CursorStatus.OPEN` 。

- `<3>` 处，调用 `DefaultResultSetHandler#handleRowValues(...)` 方法，遍历下一条记录。也就是说，回到了 [[「3.1.2.2 handleRowValues」](http://svip.iocoder.cn/MyBatis/executor-4/#) 的流程。遍历的下一条件记录，会暂存到 `objectWrapperResultHandler.result` 中。

- `<4>` 处，复制给 `next` 。

- `<5>` 处，增加 `indexWithRowBound` 。

- `<6>` 处，没有更多记录，或者到达 `rowBounds` 的限制索引位置，则关闭游标，并设置状态为 `CursorStatus.CONSUMED` 。其中，涉及到的方法，代码如下：

  ```
  // DefaultCursor.java
  
  private int getReadItemsCount() {
      return indexWithRowBound + 1;
  }
  
  @Override
  public void close() {
      if (isClosed()) {
          return;
      }
  
      // 关闭 ResultSet
      ResultSet rs = rsw.getResultSet();
      try {
          if (rs != null) {
              rs.close();
          }
      } catch (SQLException e) {
          // ignore
      } finally {
          // 设置状态为 CursorStatus.CLOSED
          status = CursorStatus.CLOSED;
      }
  }
  ```

- `<7>` 处，置空 `objectWrapperResultHandler.result` 属性。
- `<8>` 处，返回 `next` 。

# 5、延迟加载

## 1. 概述

本文，我们来分享 SQL 执行的第五部分，延迟加载的功能的实现，涉及 `executor/loader` 包。整体类图如下：[![类图](http://static.iocoder.cn/images/MyBatis/2020_03_12/01.png)](http://static.iocoder.cn/images/MyBatis/2020_03_12/01.png)类图

- 从类图，我们发现，延迟加载的功能，是通过**动态代理**实现的。也就是说，通过拦截指定方法，执行数据加载，从而实现延迟加载。
- 并且，MyBatis 提供了 Cglib 和 Javassist 两种动态代理的创建方式。

在 [《精尽 MyBatis 源码分析 —— SQL 执行（四）之 ResultSetHandler》](http://svip.iocoder.cn/MyBatis/executor-4) 方法中，我们已经看到延迟加载相关的代码，下面让我们一处一处来看看。

另外，如果胖友并未使用过 MyBatis 的延迟加载的功能，可以先看看 [《【MyBatis框架】高级映射-延迟加载》](https://blog.csdn.net/acmman/article/details/46696167) 文章。

## 2. ResultLoader

在 DefaultResultSetHandler 的 `#getNestedQueryConstructorValue(ResultSet rs, ResultMapping constructorMapping, String columnPrefix)` 方法中，我们可以看到 ResultLoader 的身影。代码如下：

```
// DefaultResultSetHandler.java

// 获得嵌套查询的值
private Object getNestedQueryConstructorValue(ResultSet rs, ResultMapping constructorMapping, String columnPrefix) throws SQLException {
    // 获得内嵌查询的编号
    final String nestedQueryId = constructorMapping.getNestedQueryId();
    // 获得内嵌查询的 MappedStatement 对象
    final MappedStatement nestedQuery = configuration.getMappedStatement(nestedQueryId);
    // 获得内嵌查询的参数类型
    final Class<?> nestedQueryParameterType = nestedQuery.getParameterMap().getType();
    // 获得内嵌查询的参数对象
    final Object nestedQueryParameterObject = prepareParameterForNestedQuery(rs, constructorMapping, nestedQueryParameterType, columnPrefix);
    Object value = null;
    if (nestedQueryParameterObject != null) {
        // 获得 BoundSql 对象
        final BoundSql nestedBoundSql = nestedQuery.getBoundSql(nestedQueryParameterObject);
        // 获得 CacheKey 对象
        final CacheKey key = executor.createCacheKey(nestedQuery, nestedQueryParameterObject, RowBounds.DEFAULT, nestedBoundSql);
        final Class<?> targetType = constructorMapping.getJavaType();
        // <x> 创建 ResultLoader 对象
        final ResultLoader resultLoader = new ResultLoader(configuration, executor, nestedQuery, nestedQueryParameterObject, targetType, key, nestedBoundSql);
        // 加载结果
        value = resultLoader.loadResult();
    }
    return value;
}
```

- `<x>` 处，因为是结果对象的构造方法中使用的值，无法使用延迟加载的功能，所以使用 ResultLoader 直接加载。

------

`org.apache.ibatis.executor.loader.ResultLoader` ，结果加载器。

#### 2.1 构造方法

```
// ResultLoader.java

protected final Configuration configuration;
protected final Executor executor;
protected final MappedStatement mappedStatement;
/**
 * 查询的参数对象
 */
protected final Object parameterObject;
/**
 * 结果的类型
 */
protected final Class<?> targetType;
protected final ObjectFactory objectFactory;
protected final CacheKey cacheKey;
protected final BoundSql boundSql;
/**
 * ResultExtractor 对象
 */
protected final ResultExtractor resultExtractor;
/**
 * 创建 ResultLoader 对象时，所在的线程
 */
protected final long creatorThreadId;

/**
 * 是否已经加载
 */
protected boolean loaded;
/**
 * 查询的结果对象
 */
protected Object resultObject;

public ResultLoader(Configuration config, Executor executor, MappedStatement mappedStatement, Object parameterObject, Class<?> targetType, CacheKey cacheKey, BoundSql boundSql) {
    this.configuration = config;
    this.executor = executor;
    this.mappedStatement = mappedStatement;
    this.parameterObject = parameterObject;
    this.targetType = targetType;
    this.objectFactory = configuration.getObjectFactory();
    this.cacheKey = cacheKey;
    this.boundSql = boundSql;
    // 初始化 resultExtractor
    this.resultExtractor = new ResultExtractor(configuration, objectFactory);
    // 初始化 creatorThreadId
    this.creatorThreadId = Thread.currentThread().getId();
}
```

- 重点属性，看添加了中文注释的。

#### 2.2 loadResult

`#loadResult()` 方法，加载结果。代码如下：

```
// ResultLoader.java

public Object loadResult() throws SQLException {
    // <1> 查询结果
    List<Object> list = selectList();
    // <2> 提取结果
    resultObject = resultExtractor.extractObjectFromList(list, targetType);
    // <3> 返回结果
    return resultObject;
}
```

- `<1>` 处，调用 `#selectList()` 方法，查询结果。详细解析，见 [「2.3 selectList」](http://svip.iocoder.cn/MyBatis/executor-5/#) 。
- `<2>` 处，调用 `ResultExtractor#extractObjectFromList(List<Object> list, Class<?> targetType)` 方法，提取结果。详细解析，见 [「3. ResultExtractor」](http://svip.iocoder.cn/MyBatis/executor-5/#) 。
- `<3>` 处，返回结果。

#### 2.3 selectList

`#selectList()` 方法，查询结果。代码如下：

```
// ResultLoader.java

private <E> List<E> selectList() throws SQLException {
    // <1> 获得 Executor 对象
    Executor localExecutor = executor;
    if (Thread.currentThread().getId() != this.creatorThreadId || localExecutor.isClosed()) {
        localExecutor = newExecutor();
    }
    // <2> 执行查询
    try {
        return localExecutor.query(mappedStatement, parameterObject, RowBounds.DEFAULT, Executor.NO_RESULT_HANDLER, cacheKey, boundSql);
    } finally {
        // <3> 关闭 Executor 对象
        if (localExecutor != executor) {
            localExecutor.close(false);
        }
    }
}
```

- `<1>` 处，如果当前线程不是创建线程，则调用 `#newExecutor()` 方法，创建 Executor 对象，因为 Executor 是非线程安全的。代码如下：

  ```
  // ResultLoader.java
  
  private Executor newExecutor() {
      // 校验 environment
      final Environment environment = configuration.getEnvironment();
      if (environment == null) {
          throw new ExecutorException("ResultLoader could not load lazily.  Environment was not configured.");
      }
      // 校验 ds
      final DataSource ds = environment.getDataSource();
      if (ds == null) {
          throw new ExecutorException("ResultLoader could not load lazily.  DataSource was not configured.");
      }
      // 创建 Transaction 对象
      final TransactionFactory transactionFactory = environment.getTransactionFactory();
      final Transaction tx = transactionFactory.newTransaction(ds, null, false);
      // 创建 Executor 对象
      return configuration.newExecutor(tx, ExecutorType.SIMPLE);
  }
  ```

- `<2>` 处，调用 `Executor#query(...)` 方法，执行查询。

- `<3>` 处，如果是新创建的 Executor 对象，则调用 `Executor#close()` 方法，关闭 Executor 对象。

#### 2.4 wasNull

`#wasNull()` 方法，是否结果为空。代码如下：

```
// ResultLoader.java

public boolean wasNull() {
    return resultObject == null;
}
```

## 3. ResultExtractor

`org.apache.ibatis.executor.ResultExtractor` ，结果提取器。代码如下：

```
// ResultExtractor.java

public class ResultExtractor {

    private final Configuration configuration;
    private final ObjectFactory objectFactory;

    public ResultExtractor(Configuration configuration, ObjectFactory objectFactory) {
        this.configuration = configuration;
        this.objectFactory = objectFactory;
    }

    /**
     * 从 list 中，提取结果
     *
     * @param list list
     * @param targetType 结果类型
     * @return 结果
     */
    public Object extractObjectFromList(List<Object> list, Class<?> targetType) {
        Object value = null;
        // 情况一，targetType 就是 list ，直接返回
        if (targetType != null && targetType.isAssignableFrom(list.getClass())) {
            value = list;
        // 情况二，targetType 是集合，添加到其中
        } else if (targetType != null && objectFactory.isCollection(targetType)) {
            // 创建 Collection 对象
            value = objectFactory.create(targetType);
            // 将结果添加到其中
            MetaObject metaObject = configuration.newMetaObject(value);
            metaObject.addAll(list);
        // 情况三，targetType 是数组
        } else if (targetType != null && targetType.isArray()) {
            // 创建 array 数组
            Class<?> arrayComponentType = targetType.getComponentType();
            Object array = Array.newInstance(arrayComponentType, list.size());
            // 赋值到 array 中
            if (arrayComponentType.isPrimitive()) {
                for (int i = 0; i < list.size(); i++) {
                    Array.set(array, i, list.get(i));
                }
                value = array;
            } else {
                value = list.toArray((Object[]) array);
            }
        // 情况四，普通对象，取首个对象
        } else {
            if (list != null && list.size() > 1) {
                throw new ExecutorException("Statement returned more than one row, where no more than one was expected.");
            } else if (list != null && list.size() == 1) {
                value = list.get(0);
            }
        }
        return value;
    }
    
}
```

- 分成四种情况，胖友看下代码注释。

## 4. ResultLoaderMap

在 DefaultResultSetHandler 的 `#getNestedQueryMappingValue(ResultSet rs, MetaObject metaResultObject, ResultMapping propertyMapping, ResultLoaderMap lazyLoader, String columnPrefix)` 方法中，我们可以看到 ResultLoaderMap 的身影。代码如下：

```
// DefaultResultSetHandler.java

// 获得嵌套查询的值
private Object getNestedQueryMappingValue(ResultSet rs, MetaObject metaResultObject, ResultMapping propertyMapping, ResultLoaderMap lazyLoader, String columnPrefix)
        throws SQLException {
    // 获得内嵌查询的编号
    final String nestedQueryId = propertyMapping.getNestedQueryId();
    // 获得属性名
    final String property = propertyMapping.getProperty();
    // 获得内嵌查询的 MappedStatement 对象
    final MappedStatement nestedQuery = configuration.getMappedStatement(nestedQueryId);
    // 获得内嵌查询的参数类型
    final Class<?> nestedQueryParameterType = nestedQuery.getParameterMap().getType();
    // 获得内嵌查询的参数对象
    final Object nestedQueryParameterObject = prepareParameterForNestedQuery(rs, propertyMapping, nestedQueryParameterType, columnPrefix);
    Object value = null;
    if (nestedQueryParameterObject != null) {
        // 获得 BoundSql 对象
        final BoundSql nestedBoundSql = nestedQuery.getBoundSql(nestedQueryParameterObject);
        // 获得 CacheKey 对象
        final CacheKey key = executor.createCacheKey(nestedQuery, nestedQueryParameterObject, RowBounds.DEFAULT, nestedBoundSql);
        final Class<?> targetType = propertyMapping.getJavaType();
        // <y> 检查缓存中已存在
        if (executor.isCached(nestedQuery, key)) {
            // 创建 DeferredLoad 对象，并通过该 DeferredLoad 对象从缓存中加载结采对象
            executor.deferLoad(nestedQuery, metaResultObject, property, key, targetType);
            // 返回已定义
            value = DEFERED;
        // 检查缓存中不存在
        } else {
            // 创建 ResultLoader 对象
            final ResultLoader resultLoader = new ResultLoader(configuration, executor, nestedQuery, nestedQueryParameterObject, targetType, key, nestedBoundSql);
            // <x> 如果要求延迟加载，则延迟加载
            if (propertyMapping.isLazy()) {
                // 如果该属性配置了延迟加载，则将其添加到 `ResultLoader.loaderMap` 中，等待真正使用时再执行嵌套查询并得到结果对象。
                lazyLoader.addLoader(property, metaResultObject, resultLoader);
                // 返回已定义
                value = DEFERED;
            // 如果不要求延迟加载，则直接执行加载对应的值
            } else {
                value = resultLoader.loadResult();
            }
        }
    }
    return value;
}
```

- `<x>` 处，因为是结果对象的 setting 方法中使用的值，可以使用延迟加载的功能，所以使用 ResultLoaderMap 记录。最终会创建结果对象的**代理对象**，而 ResultLoaderMap 对象会传入其中，作为一个参数。从而能够，在加载该属性时，能够调用 `ResultLoader#loadResult()` 方法，加载结果。

- 另外，在 `<y>` 处，检查缓存中已存在，则会调用 `Executor#deferLoad(...)` 方法，**尝试**加载结果。代码如下：

  > 老艿艿：此处是插入，😈 找不到适合放这块内容的地方了，哈哈哈。

  ```
  // 该方法在 BaseExecutor 抽象类中实现
  // BaseExecutor.java 
      
  /**
   * DeferredLoad( 延迟加载 ) 队列
   */
  protected ConcurrentLinkedQueue<DeferredLoad> deferredLoads;
  
  @Override
  public void deferLoad(MappedStatement ms, MetaObject resultObject, String property, CacheKey key, Class<?> targetType) {
      // 如果执行器已关闭，抛出 ExecutorException 异常
      if (closed) {
          throw new ExecutorException("Executor was closed.");
      }
      // 创建 DeferredLoad 对象
      DeferredLoad deferredLoad = new DeferredLoad(resultObject, property, key, localCache, configuration, targetType);
      // 如果可加载，则执行加载
      if (deferredLoad.canLoad()) {
          deferredLoad.load();
      // 如果不可加载，则添加到 deferredLoads 中
      } else {
          deferredLoads.add(new DeferredLoad(resultObject, property, key, localCache, configuration, targetType));
      }
  }
  
  private static class DeferredLoad {
  
      private final MetaObject resultObject;
      private final String property;
      private final Class<?> targetType;
      private final CacheKey key;
      private final PerpetualCache localCache;
      private final ObjectFactory objectFactory;
      private final ResultExtractor resultExtractor;
  
      // issue #781
      public DeferredLoad(MetaObject resultObject,
                          String property,
                          CacheKey key,
                          PerpetualCache localCache,
                          Configuration configuration,
                          Class<?> targetType) {
          this.resultObject = resultObject;
          this.property = property;
          this.key = key;
          this.localCache = localCache;
          this.objectFactory = configuration.getObjectFactory();
          this.resultExtractor = new ResultExtractor(configuration, objectFactory);
          this.targetType = targetType;
      }
  
      public boolean canLoad() {
          return localCache.getObject(key) != null && localCache.getObject(key) != EXECUTION_PLACEHOLDER;
      }
  
      public void load() {
          @SuppressWarnings("unchecked")
          // we suppose we get back a List
          // 从缓存 localCache 中获取
          List<Object> list = (List<Object>) localCache.getObject(key);
          // 解析结果
          Object value = resultExtractor.extractObjectFromList(list, targetType);
          // 设置到 resultObject 中
          resultObject.setValue(property, value);
      }
  
  }
  ```

  - 代码比较简单，胖友自己瞅瞅。

------

`org.apache.ibatis.executor.loader.ResultLoaderMap` ， ResultLoader 的映射。该映射，最终创建代理对象时，会作为参数传入代理。

#### 4.1 构造方法

```
// ResultLoaderMap.java

/**
 * LoadPair 的映射
 */
private final Map<String, LoadPair> loaderMap = new HashMap<>();
```

#### 4.2 addLoader

`#addLoader(String property, MetaObject metaResultObject, ResultLoader resultLoader)` 方法，添加到 `loaderMap` 中。代码如下：

```
// ResultLoaderMap.java

public void addLoader(String property, MetaObject metaResultObject, ResultLoader resultLoader) {
    String upperFirst = getUppercaseFirstProperty(property);
    // 已存在，则抛出 ExecutorException 异常
    if (!upperFirst.equalsIgnoreCase(property) && loaderMap.containsKey(upperFirst)) {
        throw new ExecutorException("Nested lazy loaded result property '" + property +
                "' for query id '" + resultLoader.mappedStatement.getId() +
                " already exists in the result map. The leftmost property of all lazy loaded properties must be unique within a result map.");
    }
    // 创建 LoadPair 对象，添加到 loaderMap 中
    loaderMap.put(upperFirst, new LoadPair(property, metaResultObject, resultLoader));
}

/**
 * 使用 . 分隔属性，并获得首个字符串，并大写
 *
 * @param property 属性
 * @return 字符串 + 大写
 */
private static String getUppercaseFirstProperty(String property) {
    String[] parts = property.split("\\.");
    return parts[0].toUpperCase(Locale.ENGLISH);
}
```

- 其中，LoadPair 是 ResultLoaderMap 的内部静态类。代码如下：

  ```
  // ResultLoaderMap.java
  
  public static class LoadPair implements Serializable {
  
      private static final long serialVersionUID = 20130412;
  
      /**
       * Name of factory method which returns database connection.
       */
      private static final String FACTORY_METHOD = "getConfiguration";
      /**
       * Object to check whether we went through serialization..
       */
      private final transient Object serializationCheck = new Object();
      /**
       * Meta object which sets loaded properties.
       */
      private transient MetaObject metaResultObject;
      /**
       * Result loader which loads unread properties.
       */
      private transient ResultLoader resultLoader;
      /**
       * Wow, logger.
       */
      private transient Log log;
      /**
       * Factory class through which we get database connection.
       */
      private Class<?> configurationFactory;
      /**
       * Name of the unread property.
       */
      private String property;
      /**
       * ID of SQL statement which loads the property.
       */
      private String mappedStatement;
      /**
       * Parameter of the sql statement.
       */
      private Serializable mappedParameter;
  
      private LoadPair(final String property, MetaObject metaResultObject, ResultLoader resultLoader) {
          this.property = property;
          this.metaResultObject = metaResultObject;
          this.resultLoader = resultLoader;
  
          /* Save required information only if original object can be serialized. */
          // 当 `metaResultObject.originalObject` 可序列化时，则记录 mappedStatement、mappedParameter、configurationFactory 属性
          if (metaResultObject != null && metaResultObject.getOriginalObject() instanceof Serializable) {
              final Object mappedStatementParameter = resultLoader.parameterObject;
  
              /* @todo May the parameter be null? */
              if (mappedStatementParameter instanceof Serializable) {
                  this.mappedStatement = resultLoader.mappedStatement.getId();
                  this.mappedParameter = (Serializable) mappedStatementParameter;
  
                  this.configurationFactory = resultLoader.configuration.getConfigurationFactory();
              } else {
                  Log log = this.getLogger();
                  if (log.isDebugEnabled()) {
                      log.debug("Property [" + this.property + "] of ["
                              + metaResultObject.getOriginalObject().getClass() + "] cannot be loaded "
                              + "after deserialization. Make sure it's loaded before serializing "
                              + "forenamed object.");
                  }
              }
          }
      }
      
      // ... 暂时省略其它方法
  }
  ```

#### 4.3 load

`#load(String property)` 方法，执行指定属性的加载。代码如下：

```
// ResultLoaderMap.java

public boolean load(String property) throws SQLException {
    // 获得 LoadPair 对象，并移除
    LoadPair pair = loaderMap.remove(property.toUpperCase(Locale.ENGLISH));
    // 执行加载
    if (pair != null) {
        pair.load();
        return true; // 加载成功
    }
    return false; // 加载失败
}
```

- 调用 `LoadPair#load()` 方法，执行加载。代码如下：

```
// ResultLoaderMap.java

public void load() throws SQLException {
    /* These field should not be null unless the loadpair was serialized.
     * Yet in that case this method should not be called. */
    // 若 metaResultObject 或 resultLoader 为空，抛出 IllegalArgumentException 异常
    if (this.metaResultObject == null) {
        throw new IllegalArgumentException("metaResultObject is null");
    }
    if (this.resultLoader == null) {
        throw new IllegalArgumentException("resultLoader is null");
    }

    // 执行加载
    this.load(null);
}

public void load(final Object userObject) throws SQLException {
    if (this.metaResultObject == null || this.resultLoader == null) { // <1>
        if (this.mappedParameter == null) {
            throw new ExecutorException("Property [" + this.property + "] cannot be loaded because "
                    + "required parameter of mapped statement ["
                    + this.mappedStatement + "] is not serializable.");
        }

        // 获得 Configuration 对象
        final Configuration config = this.getConfiguration();
        // 获得 MappedStatement 对象
        final MappedStatement ms = config.getMappedStatement(this.mappedStatement);
        if (ms == null) {
            throw new ExecutorException("Cannot lazy load property [" + this.property
                    + "] of deserialized object [" + userObject.getClass()
                    + "] because configuration does not contain statement ["
                    + this.mappedStatement + "]");
        }

        // 获得对应的 MetaObject 对象
        this.metaResultObject = config.newMetaObject(userObject);
        // 创建 ResultLoader 对象
        this.resultLoader = new ResultLoader(config, new ClosedExecutor(), ms, this.mappedParameter,
                metaResultObject.getSetterType(this.property), null, null);
    }

    /* We are using a new executor because we may be (and likely are) on a new thread
     * and executors aren't thread safe. (Is this sufficient?)
     *
     * A better approach would be making executors thread safe. */
    if (this.serializationCheck == null) { // <2>
        final ResultLoader old = this.resultLoader;
        this.resultLoader = new ResultLoader(old.configuration, new ClosedExecutor(), old.mappedStatement,
                old.parameterObject, old.targetType, old.cacheKey, old.boundSql);
    }

    // <3>
    this.metaResultObject.setValue(property, this.resultLoader.loadResult());
}
```

- `<1>` 和 `<2>` 处，胖友可以暂时无视，主要用于延迟加载在**序列化和反序列化**的时候，一般很少碰到。当然，感兴趣的胖友，可以调试下 `org.apache.ibatis.submitted.lazy_deserialize.LazyDeserializeTest` 单元测试类。
- 【重点】`<3>` 处，调用 `ResultLoader#loadResult()` 方法，执行查询结果。
- `<3>` 处，调用 `MetaObject#setValue(String name, Object value)` 方法，设置属性。

#### 4.4 loadAll

`#loadAll()` 方法，执行所有属性的加载。代码如下：

```
// ResultLoaderMap.java

public void loadAll() throws SQLException {
    // 遍历 loaderMap 属性
    final Set<String> methodNameSet = loaderMap.keySet();
    String[] methodNames = methodNameSet.toArray(new String[methodNameSet.size()]);
    for (String methodName : methodNames) {
        // 执行加载
        load(methodName);
    }
}
```

#### 4.5 其它方法

ResultLoaderMap 还有其它方法，比较简单，胖友可以自己看看。

## 5. ProxyFactory

在 DefaultResultSetHandler 的 `#createResultObject(ResultSetWrapper rsw, ResultMap resultMap, ResultLoaderMap lazyLoader, String columnPrefix)` 方法中，我们可以看到 ProxyFactory 的身影。代码如下：

```
// DefaultResultSetHandler.java

private Object createResultObject(ResultSetWrapper rsw, ResultMap resultMap, ResultLoaderMap lazyLoader, String columnPrefix) throws SQLException {
    // useConstructorMappings ，表示是否使用构造方法创建该结果对象。此处将其重置
    this.useConstructorMappings = false; // reset previous mapping result
    final List<Class<?>> constructorArgTypes = new ArrayList<>(); // 记录使用的构造方法的参数类型的数组
    final List<Object> constructorArgs = new ArrayList<>(); // 记录使用的构造方法的参数值的数组
    // 创建映射后的结果对象
    Object resultObject = createResultObject(rsw, resultMap, constructorArgTypes, constructorArgs, columnPrefix);
    if (resultObject != null && !hasTypeHandlerForResultObject(rsw, resultMap.getType())) {
        // 如果有内嵌的查询，并且开启延迟加载，则创建结果对象的代理对象
        final List<ResultMapping> propertyMappings = resultMap.getPropertyResultMappings();
        for (ResultMapping propertyMapping : propertyMappings) {
            // issue gcode #109 && issue #149
            if (propertyMapping.getNestedQueryId() != null && propertyMapping.isLazy()) {
                resultObject = configuration.getProxyFactory().createProxy(resultObject, lazyLoader, configuration, objectFactory, constructorArgTypes, constructorArgs); // <X>
                break;
            }
        }
    }
    // 判断是否使用构造方法创建该结果对象
    this.useConstructorMappings = resultObject != null && !constructorArgTypes.isEmpty(); // set current mapping result
    return resultObject;
}
```

- `<x>` 处，调用 `ProxyFactory#createProxy(...)` 方法，创建结果对象的代理对象。

------

`org.apache.ibatis.executor.loader.ProxyFactory` ，代理工厂接口，用于创建需要延迟加载属性的结果对象。代码如下：

```
// ProxyFactory.java

public interface ProxyFactory {

    // 设置属性，目前是空实现。可以暂时无视该方法
    void setProperties(Properties properties);

    // 创建代理对象
    Object createProxy(Object target, ResultLoaderMap lazyLoader, Configuration configuration, ObjectFactory objectFactory, List<Class<?>> constructorArgTypes, List<Object> constructorArgs);

}
```

- ProxyFactory 有 JavassistProxyFactory 和 CglibProxyFactory 两个实现类，默认使用**前者**。原因见如下代码：

  ```
  // Configuration.java
  
  protected ProxyFactory proxyFactory = new JavassistProxyFactory(); // #224 Using internal Javassist instead of OGNL
  ```

#### 5.1 JavassistProxyFactory

`org.apache.ibatis.executor.loader.javassist.JavassistProxyFactory` ，实现 ProxyFactory 接口，基于 Javassist 的 ProxyFactory 实现类。

###### 5.1.1 构造方法

```
// JavassistProxyFactory.java

private static final Log log = LogFactory.getLog(JavassistProxyFactory.class);

private static final String FINALIZE_METHOD = "finalize";
private static final String WRITE_REPLACE_METHOD = "writeReplace";

public JavassistProxyFactory() {
    try {
        // 加载 javassist.util.proxy.ProxyFactory 类
        Resources.classForName("javassist.util.proxy.ProxyFactory");
    } catch (Throwable e) {
        throw new IllegalStateException("Cannot enable lazy loading because Javassist is not available. Add Javassist to your classpath.", e);
    }
}
```

###### 5.1.2 createDeserializationProxy

`#createDeserializationProxy(Object target, Map<String, ResultLoaderMap.LoadPair> unloadedProperties, ObjectFactory objectFactory, List<Class<?>> constructorArgTypes, List<Object> constructorArgs)` 方法，创建支持**反序列化**的代理对象。代码如下：

```
// JavassistProxyFactory.java

public Object createDeserializationProxy(Object target, Map<String, ResultLoaderMap.LoadPair> unloadedProperties, ObjectFactory objectFactory, List<Class<?>> constructorArgTypes, List<Object> constructorArgs) {
    return EnhancedDeserializationProxyImpl.createProxy(target, unloadedProperties, objectFactory, constructorArgTypes, constructorArgs);
}
```

- 因为实际场景下，不太使用该功能，所以本文暂时无视。

###### 5.1.3 createProxy 普通方法

`#createProxy(Object target, ResultLoaderMap lazyLoader, Configuration configuration, ObjectFactory objectFactory, List<Class<?>> constructorArgTypes, List<Object> constructorArgs)` 方法，创建代理对象。代码如下：

```
// JavassistProxyFactory.java

@Override
public Object createProxy(Object target, ResultLoaderMap lazyLoader, Configuration configuration, ObjectFactory objectFactory, List<Class<?>> constructorArgTypes, List<Object> constructorArgs) {
    return EnhancedResultObjectProxyImpl.createProxy(target, lazyLoader, configuration, objectFactory, constructorArgTypes, constructorArgs);
}
```

###### 5.1.4 crateProxy 静态方法

`#crateProxy(Class<?> type, MethodHandler callback, List<Class<?>> constructorArgTypes, List<Object> constructorArgs)` **静态**方法，创建代理对象。代码如下：

```
// JavassistProxyFactory.java

static Object crateProxy(Class<?> type, MethodHandler callback, List<Class<?>> constructorArgTypes, List<Object> constructorArgs) {
    // 创建 javassist ProxyFactory 对象
    ProxyFactory enhancer = new ProxyFactory();
    // 设置父类
    enhancer.setSuperclass(type);

    // 根据情况，设置接口为 WriteReplaceInterface 。和序列化相关，可以无视
    try {
        type.getDeclaredMethod(WRITE_REPLACE_METHOD); // 如果已经存在 writeReplace 方法，则不用设置接口为 WriteReplaceInterface
        // ObjectOutputStream will call writeReplace of objects returned by writeReplace
        if (log.isDebugEnabled()) {
            log.debug(WRITE_REPLACE_METHOD + " method was found on bean " + type + ", make sure it returns this");
        }
    } catch (NoSuchMethodException e) {
        enhancer.setInterfaces(new Class[]{WriteReplaceInterface.class}); // 如果不存在 writeReplace 方法，则设置接口为 WriteReplaceInterface
    } catch (SecurityException e) {
        // nothing to do here
    }

    // 创建代理对象
    Object enhanced;
    Class<?>[] typesArray = constructorArgTypes.toArray(new Class[constructorArgTypes.size()]);
    Object[] valuesArray = constructorArgs.toArray(new Object[constructorArgs.size()]);
    try {
        enhanced = enhancer.create(typesArray, valuesArray);
    } catch (Exception e) {
        throw new ExecutorException("Error creating lazy proxy.  Cause: " + e, e);
    }

    // <x> 设置代理对象的执行器
    ((Proxy) enhanced).setHandler(callback);
    return enhanced;
}
```

- 常见的基于 Javassist 的 API ，创建代理对象。
- `<x>` 处，设置代理对象的执行器。该执行器，就是 EnhancedResultObjectProxyImpl 对象。详细解析，见 [「5.1.5 EnhancedResultObjectProxyImpl」](http://svip.iocoder.cn/MyBatis/executor-5/#) 。

###### 5.1.5 EnhancedResultObjectProxyImpl

EnhancedResultObjectProxyImpl ，是 JavassistProxyFactory 的内部静态类，实现 `javassist.util.proxy.MethodHandler` 接口，方法处理器实现类。

####### 5.1.5.1 构造方法

```
// JavassistProxyFactory.java

private static class EnhancedResultObjectProxyImpl implements MethodHandler {

    private final Class<?> type;
    private final ResultLoaderMap lazyLoader;
    private final boolean aggressive;
    private final Set<String> lazyLoadTriggerMethods;
    private final ObjectFactory objectFactory;
    private final List<Class<?>> constructorArgTypes;
    private final List<Object> constructorArgs;

    private EnhancedResultObjectProxyImpl(Class<?> type, ResultLoaderMap lazyLoader, Configuration configuration, ObjectFactory objectFactory, List<Class<?>> constructorArgTypes, List<Object> constructorArgs) {
        this.type = type;
        this.lazyLoader = lazyLoader;
        this.aggressive = configuration.isAggressiveLazyLoading();
        this.lazyLoadTriggerMethods = configuration.getLazyLoadTriggerMethods();
        this.objectFactory = objectFactory;
        this.constructorArgTypes = constructorArgTypes;
        this.constructorArgs = constructorArgs;
    }
    
    // ... 暂时省略无关方法
}
```

- 涉及的 `aggressive` 和 `lazyLoadTriggerMethods` 属性，在 Configuration 定义如下：

  ```
  // Configuration.java
  
  /**
   * 当开启时，任何方法的调用都会加载该对象的所有属性。否则，每个属性会按需加载（参考lazyLoadTriggerMethods)
   */
  protected boolean aggressiveLazyLoading;
  
  /**
   * 指定哪个对象的方法触发一次延迟加载。
   */
  protected Set<String> lazyLoadTriggerMethods = new HashSet<>(Arrays.asList("equals", "clone", "hashCode", "toString"));
  ```

####### 5.1.5.2 createProxy

`#createProxy(Object target, ResultLoaderMap lazyLoader, Configuration configuration, ObjectFactory objectFactory, List<Class<?>> constructorArgTypes, List<Object> constructorArgs)` 方法，创建代理对象，并设置方法处理器为 EnhancedResultObjectProxyImpl 对象。代码如下：

> 因为方法名 createProxy 一直在重复，所以这里艿艿说下调用链：
>
> 「5.1.3 createProxy 普通方法」 => 「5.1.4.2 createProxy」 => 「5.1.4 createProxy 静态方法」

```
// JavassistProxyFactory.java

public static Object createProxy(Object target, ResultLoaderMap lazyLoader, Configuration configuration, ObjectFactory objectFactory, List<Class<?>> constructorArgTypes, List<Object> constructorArgs) {
    final Class<?> type = target.getClass();
    // 创建 EnhancedResultObjectProxyImpl 对象
    EnhancedResultObjectProxyImpl callback = new EnhancedResultObjectProxyImpl(type, lazyLoader, configuration, objectFactory, constructorArgTypes, constructorArgs);
    // 创建代理对象
    Object enhanced = crateProxy(type, callback, constructorArgTypes, constructorArgs);
    // 将 target 的属性，复制到 enhanced 中
    PropertyCopier.copyBeanProperties(type, target, enhanced);
    return enhanced;
}
```

- 代码比较简单，胖友仔细瞅瞅，不要绕晕噢。

####### 5.1.5.3 invoke

`#invoke(Object enhanced, Method method, Method methodProxy, Object[] args)` 方法，执行方法。代码如下：

```
// JavassistProxyFactory.java

    @Override
    public Object invoke(Object enhanced, Method method, Method methodProxy, Object[] args) throws Throwable {
        final String methodName = method.getName();
        try {
            synchronized (lazyLoader) {
                // 忽略 WRITE_REPLACE_METHOD ，和序列化相关
                if (WRITE_REPLACE_METHOD.equals(methodName)) {
                    Object original;
                    if (constructorArgTypes.isEmpty()) {
                        original = objectFactory.create(type);
                    } else {
                        original = objectFactory.create(type, constructorArgTypes, constructorArgs);
                    }
                    PropertyCopier.copyBeanProperties(type, enhanced, original);
                    if (lazyLoader.size() > 0) {
                        return new JavassistSerialStateHolder(original, lazyLoader.getProperties(), objectFactory, constructorArgTypes, constructorArgs);
                    } else {
                        return original;
                    }
                } else {
                    if (lazyLoader.size() > 0 && !FINALIZE_METHOD.equals(methodName)) {
                        // <1.1> 加载所有延迟加载的属性
                        if (aggressive || lazyLoadTriggerMethods.contains(methodName)) {
                            lazyLoader.loadAll();
                        // <1.2> 如果调用了 setting 方法，则不在使用延迟加载
                        } else if (PropertyNamer.isSetter(methodName)) {
                            final String property = PropertyNamer.methodToProperty(methodName);
                            lazyLoader.remove(property); // 移除
                        // <1.3> 如果调用了 getting 方法，则执行延迟加载
                        } else if (PropertyNamer.isGetter(methodName)) {
                            final String property = PropertyNamer.methodToProperty(methodName);
                            if (lazyLoader.hasLoader(property)) {
                                lazyLoader.load(property);
                            }
                        }
                    }
                }
            }
            // <2> 继续执行原方法
            return methodProxy.invoke(enhanced, args);
        } catch (Throwable t) {
            throw ExceptionUtil.unwrapThrowable(t);
        }
    }
}
```

- `<1.1>` 处，如果满足条件，则调用 `ResultLoaderMap#loadAll()` 方法，加载所有延迟加载的属性。
- `<1.2>` 处，如果调用了 setting 方法，则调用 `ResultLoaderMap#remove(String property)` 方法，不在使用延迟加载。因为，具体的值都设置了，无需在延迟加载了。
- `<1.3>` 处，如果调用了 getting 方法，则调用 `ResultLoaderMap#load(String property)` 方法，执行指定属性的延迟加载。**此处，就会去数据库中查询，并设置到对应的属性**。
- `<2>` 处，继续执行原方法。

#### 5.2 CglibProxyFactory

`org.apache.ibatis.executor.loader.cglib.CglibProxyFactory` ，实现 ProxyFactory 接口，基于 Cglib 的 ProxyFactory 实现类。

CglibProxyFactory 和 JavassistProxyFactory 的代码实现非常类似，此处就不重复解析了。感兴趣的胖友，自己研究下。

# 总结

参考和推荐如下文章：

- 祖大俊 [《Mybatis3.4.x技术内幕（二十一）：参数设置、结果封装、级联查询、延迟加载原理分析》](https://my.oschina.net/zudajun/blog/747283)
- 无忌 [《MyBatis源码解读之延迟加载》](https://my.oschina.net/wenjinglian/blog/1857581)
- 徐郡明 [《MyBatis 技术内幕》](https://item.jd.com/12125531.html) 的 [「3.3.5 嵌套查询&延迟加载」](http://svip.iocoder.cn/MyBatis/executor-5/#) 小节
- 徐郡明 [《MyBatis 技术内幕》](https://item.jd.com/12125531.html) 的 [「3.3 ResultSetHandler」](http://svip.iocoder.cn/MyBatis/executor-4/#) 小节
- 祖大俊 [《Mybatis3.3.x技术内幕（十四）：Mybatis之KeyGenerator》](https://my.oschina.net/zudajun/blog/673612)
- 祖大俊《Mybatis3.3.x技术内幕（十五）：Mybatis之foreach批量insert，返回主键id列表（修复Mybatis返回null的bug）》
  - 目前已经修复，参见 https://github.com/mybatis/mybatis-3/pull/324 。
- 徐郡明 [《MyBatis 技术内幕》](https://item.jd.com/12125531.html) 的 [「3.4 KeyGenerator」](http://svip.iocoder.cn/MyBatis/executor-3/#) 小节
- 祖大俊 [《Mybatis3.3.x技术内幕（六）：StatementHandler（Box stop here）》](https://my.oschina.net/zudajun/blog/668378)
- 徐郡明 [《MyBatis 技术内幕》](https://item.jd.com/12125531.html) 的 [「3.5 StatementHandler](http://svip.iocoder.cn/MyBatis/executor-2/#) 小节
- 凯伦 [《聊聊MyBatis缓存机制》](https://tech.meituan.com/mybatis_cache.html)
- 祖大俊 [《Mybatis3.3.x技术内幕（四）：五鼠闹东京之执行器Executor设计原本》](https://my.oschina.net/zudajun/blog/667214)
- 祖大俊 [《Mybatis3.3.x技术内幕（五）：Executor之doFlushStatements()》](https://my.oschina.net/zudajun/blog/668323)
- 祖大俊 [《Mybatis3.4.x技术内幕（二十二）：Mybatis一级、二级缓存原理分析》](https://my.oschina.net/zudajun/blog/747499)
- 徐郡明 [《MyBatis 技术内幕》](https://item.jd.com/12125531.html) 的 [「3.6 Executor](http://svip.iocoder.cn/MyBatis/executor-1/#) 小节