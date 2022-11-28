# 1. 概述

在前面，我们已经详细解析了 MyBatis 执行器 Executor 相关的内容，但是显然，Executor 是不适合直接暴露给用户使用的，而是需要通过 SqlSession 。

流程如下图：

> [![流程图](http://static.iocoder.cn/images/MyBatis/2020_01_04/05.png)](http://static.iocoder.cn/images/MyBatis/2020_01_04/05.png)流程图

示例代码如下：

```
// 仅仅是示例哈

// 构建 SqlSessionFactory 对象
Reader reader = Resources.getResourceAsReader("org/apache/ibatis/autoconstructor/mybatis-config.xml");
SqlSessionFactory sqlSessionFactory = new SqlSessionFactoryBuilder().build(reader);

// 获得 SqlSession 对象
SqlSession sqlSession = sqlSessionFactory.openSession();

// 获得 Mapper 对象
final AutoConstructorMapper mapper = sqlSession.getMapper(AutoConstructorMapper.class);

// 执行查询
final Object subject = mapper.getSubject(1);
```

而本文解析的类，都在 `session` 包下，整体类图如下：

> 老艿艿：省略了一部分前面已经解析过的类。

[![类图](http://static.iocoder.cn/images/MyBatis/2020_03_15/01.png)](http://static.iocoder.cn/images/MyBatis/2020_03_15/01.png)类图

- 核心是 SqlSession 。
- SqlSessionFactory ，负责创建 SqlSession 对象的工厂。
- SqlSessionFactoryBuilder ，是 SqlSessionFactory 的构建器。

下面，我们按照 SqlSessionFactoryBuilder => SqlSessionFactory => SqlSession 来详细解析。

# 2. SqlSessionFactoryBuilder

`org.apache.ibatis.session.SqlSessionFactoryBuilder` ，SqlSessionFactory 构造器。代码如下：

```
// SqlSessionFactory.java

public class SqlSessionFactoryBuilder {

    public SqlSessionFactory build(Reader reader) {
        return build(reader, null, null);
    }

    public SqlSessionFactory build(Reader reader, String environment) {
        return build(reader, environment, null);
    }

    public SqlSessionFactory build(Reader reader, Properties properties) {
        return build(reader, null, properties);
    }

    /**
     * 构造 SqlSessionFactory 对象
     *
     * @param reader Reader 对象
     * @param environment 环境
     * @param properties Properties 变量
     * @return SqlSessionFactory 对象
     */
    @SuppressWarnings("Duplicates")
    public SqlSessionFactory build(Reader reader, String environment, Properties properties) {
        try {
            // 创建 XMLConfigBuilder 对象
            XMLConfigBuilder parser = new XMLConfigBuilder(reader, environment, properties);
            // 执行 XML 解析
            // 创建 DefaultSqlSessionFactory 对象
            return build(parser.parse());
        } catch (Exception e) {
            throw ExceptionFactory.wrapException("Error building SqlSession.", e);
        } finally {
            ErrorContext.instance().reset();
            try {
                reader.close();
            } catch (IOException e) {
                // Intentionally ignore. Prefer previous error.
            }
        }
    }

    public SqlSessionFactory build(InputStream inputStream) {
        return build(inputStream, null, null);
    }

    public SqlSessionFactory build(InputStream inputStream, String environment) {
        return build(inputStream, environment, null);
    }

    public SqlSessionFactory build(InputStream inputStream, Properties properties) {
        return build(inputStream, null, properties);
    }

    public SqlSessionFactory build(InputStream inputStream, String environment, Properties properties) {
        try {
            // 创建 XMLConfigBuilder 对象
            XMLConfigBuilder parser = new XMLConfigBuilder(inputStream, environment, properties);
            // 执行 XML 解析
            // 创建 DefaultSqlSessionFactory 对象
            return build(parser.parse());
        } catch (Exception e) {
            throw ExceptionFactory.wrapException("Error building SqlSession.", e);
        } finally {
            ErrorContext.instance().reset();
            try {
                inputStream.close();
            } catch (IOException e) {
                // Intentionally ignore. Prefer previous error.
            }
        }
    }

    /**
     * 创建 DefaultSqlSessionFactory 对象
     *
     * @param config Configuration 对象
     * @return DefaultSqlSessionFactory 对象
     */
    public SqlSessionFactory build(Configuration config) {
        return new DefaultSqlSessionFactory(config);
    }

}
```

- 提供了各种 build 的重载方法，核心的套路都是解析出 Configuration 配置对象，从而创建出 DefaultSqlSessionFactory 对象。

# 3. SqlSessionFactory

`org.apache.ibatis.session.SqlSessionFactory` ，SqlSession 工厂接口。代码如下：

```
// SqlSessionFactory.java

public interface SqlSessionFactory {

    SqlSession openSession();

    SqlSession openSession(boolean autoCommit);

    SqlSession openSession(Connection connection);

    SqlSession openSession(TransactionIsolationLevel level);

    SqlSession openSession(ExecutorType execType);

    SqlSession openSession(ExecutorType execType, boolean autoCommit);

    SqlSession openSession(ExecutorType execType, TransactionIsolationLevel level);

    SqlSession openSession(ExecutorType execType, Connection connection);

    Configuration getConfiguration();

}
```

- 定义了 `#openSession(...)` 和 `#getConfiguration()` 两类方法。

## 3.1 DefaultSqlSessionFactory

`org.apache.ibatis.session.defaults.DefaultSqlSessionFactory` ，实现 SqlSessionFactory 接口，默认的 SqlSessionFactory 实现类。

### 3.1.1 构造方法

```
// DefaultSqlSessionFactory.java

private final Configuration configuration;

public DefaultSqlSessionFactory(Configuration configuration) {
    this.configuration = configuration;
}

@Override
public Configuration getConfiguration() {
    return configuration;
}
```

### 3.1.2 openSession

```
// DefaultSqlSessionFactory.java

@Override
public SqlSession openSession() {
    return openSessionFromDataSource(configuration.getDefaultExecutorType(), null, false);
}

@Override
public SqlSession openSession(boolean autoCommit) {
    return openSessionFromDataSource(configuration.getDefaultExecutorType(), null, autoCommit);
}

@Override
public SqlSession openSession(ExecutorType execType) {
    return openSessionFromDataSource(execType, null, false);
}

@Override
public SqlSession openSession(TransactionIsolationLevel level) {
    return openSessionFromDataSource(configuration.getDefaultExecutorType(), level, false);
}

@Override
public SqlSession openSession(ExecutorType execType, TransactionIsolationLevel level) {
    return openSessionFromDataSource(execType, level, false);
}

@Override
public SqlSession openSession(ExecutorType execType, boolean autoCommit) {
    return openSessionFromDataSource(execType, null, autoCommit);
}

@Override
public SqlSession openSession(Connection connection) {
    return openSessionFromConnection(configuration.getDefaultExecutorType(), connection);
}

@Override
public SqlSession openSession(ExecutorType execType, Connection connection) {
    return openSessionFromConnection(execType, connection);
}
```

- 调用 `#openSessionFromDataSource(ExecutorType execType, TransactionIsolationLevel level, boolean autoCommit)` 方法，获得 SqlSession 对象。代码如下：

  ```
  // DefaultSqlSessionFactory.java
  
  private SqlSession openSessionFromDataSource(ExecutorType execType, TransactionIsolationLevel level, boolean autoCommit) {
      Transaction tx = null;
      try {
          // 获得 Environment 对象
          final Environment environment = configuration.getEnvironment();
          // 创建 Transaction 对象
          final TransactionFactory transactionFactory = getTransactionFactoryFromEnvironment(environment);
          tx = transactionFactory.newTransaction(environment.getDataSource(), level, autoCommit);
          // 创建 Executor 对象
          final Executor executor = configuration.newExecutor(tx, execType);
          // 创建 DefaultSqlSession 对象
          return new DefaultSqlSession(configuration, executor, autoCommit);
      } catch (Exception e) {
          // 如果发生异常，则关闭 Transaction 对象
          closeTransaction(tx); // may have fetched a connection so lets call close()
          throw ExceptionFactory.wrapException("Error opening session.  Cause: " + e, e);
      } finally {
          ErrorContext.instance().reset();
      }
  }
  ```

  - DefaultSqlSession 的创建，需要 `configuration`、`executor`、`autoCommit` 三个参数。

- `#openSessionFromConnection(ExecutorType execType, Connection connection)` 方法，获得 SqlSession 对象。代码如下：

  ```
  // DefaultSqlSessionFactory.java
  
  private SqlSession openSessionFromConnection(ExecutorType execType, Connection connection) {
      try {
          // 获得是否可以自动提交
          boolean autoCommit;
          try {
              autoCommit = connection.getAutoCommit();
          } catch (SQLException e) {
              // Failover to true, as most poor drivers
              // or databases won't support transactions
              autoCommit = true;
          }
          // 获得 Environment 对象
          final Environment environment = configuration.getEnvironment();
          // 创建 Transaction 对象
          final TransactionFactory transactionFactory = getTransactionFactoryFromEnvironment(environment);
          final Transaction tx = transactionFactory.newTransaction(connection);
          // 创建 Executor 对象
          final Executor executor = configuration.newExecutor(tx, execType);
          // 创建 DefaultSqlSession 对象
          return new DefaultSqlSession(configuration, executor, autoCommit);
      } catch (Exception e) {
          throw ExceptionFactory.wrapException("Error opening session.  Cause: " + e, e);
      } finally {
          ErrorContext.instance().reset();
      }
  }
  ```

### 3.1.3 getTransactionFactoryFromEnvironment

```
// DefaultSqlSessionFactory.java

private TransactionFactory getTransactionFactoryFromEnvironment(Environment environment) {
    // 情况一，创建 ManagedTransactionFactory 对象
    if (environment == null || environment.getTransactionFactory() == null) {
        return new ManagedTransactionFactory();
    }
    // 情况二，使用 `environment` 中的
    return environment.getTransactionFactory();
}
```

### 3.1.4 closeTransaction

```
// DefaultSqlSessionFactory.java

private void closeTransaction(Transaction tx) {
    if (tx != null) {
        try {
            tx.close();
        } catch (SQLException ignore) {
            // Intentionally ignore. Prefer previous error.
        }
    }
}
```

# 4. SqlSession

`org.apache.ibatis.session.SqlSession` ，SQL Session 接口。代码如下：

```
// SqlSession.java

public interface SqlSession extends Closeable {

    <T> T selectOne(String statement);
    <T> T selectOne(String statement, Object parameter);

    <E> List<E> selectList(String statement);
    <E> List<E> selectList(String statement, Object parameter);
    <E> List<E> selectList(String statement, Object parameter, RowBounds rowBounds);

    <K, V> Map<K, V> selectMap(String statement, String mapKey);
    <K, V> Map<K, V> selectMap(String statement, Object parameter, String mapKey);
    <K, V> Map<K, V> selectMap(String statement, Object parameter, String mapKey, RowBounds rowBounds);

    <T> Cursor<T> selectCursor(String statement);
    <T> Cursor<T> selectCursor(String statement, Object parameter);
    <T> Cursor<T> selectCursor(String statement, Object parameter, RowBounds rowBounds);

    void select(String statement, Object parameter, ResultHandler handler);
    void select(String statement, ResultHandler handler);
    void select(String statement, Object parameter, RowBounds rowBounds, ResultHandler handler);

    int insert(String statement);
    int insert(String statement, Object parameter);

    int update(String statement);
    int update(String statement, Object parameter);

    int delete(String statement);
    int delete(String statement, Object parameter);

    void commit();
    void commit(boolean force);
    void rollback();
    void rollback(boolean force);

    List<BatchResult> flushStatements();

    @Override
    void close();

    void clearCache();

    Configuration getConfiguration();

    /**
     * Retrieves a mapper.
     * @param <T> the mapper type
     * @param type Mapper interface class
     * @return a mapper bound to this SqlSession
     */
    <T> T getMapper(Class<T> type);

    /**
     * Retrieves inner database connection
     * @return Connection
     */
    Connection getConnection();

}
```

- 大体接口上，和 Executor 接口是相似的。

## 4.1 DefaultSqlSession

`org.apache.ibatis.session.defaults.DefaultSqlSession` ，实现 SqlSession 接口，默认的 SqlSession 实现类。

### 4.1.1 构造方法

```
// DefaultSqlSession.java

private final Configuration configuration;
private final Executor executor;

/**
 * 是否自动提交事务
 */
private final boolean autoCommit;
/**
 * 是否发生数据变更
 */
private boolean dirty;
/**
 * Cursor 数组
 */
private List<Cursor<?>> cursorList;

public DefaultSqlSession(Configuration configuration, Executor executor, boolean autoCommit) {
    this.configuration = configuration;
    this.executor = executor;
    this.dirty = false;
    this.autoCommit = autoCommit;
}

public DefaultSqlSession(Configuration configuration, Executor executor) {
    this(configuration, executor, false);
}
```

### 4.1.2 selectList

```
// DefaultSqlSession.java

@Override
public <E> List<E> selectList(String statement) {
    return this.selectList(statement, null);
}

@Override
public <E> List<E> selectList(String statement, Object parameter) {
    return this.selectList(statement, parameter, RowBounds.DEFAULT);
}

@Override
public <E> List<E> selectList(String statement, Object parameter, RowBounds rowBounds) {
    try {
        // <1> 获得 MappedStatement 对象
        MappedStatement ms = configuration.getMappedStatement(statement);
        // <2> 执行查询
        return executor.query(ms, wrapCollection(parameter), rowBounds, Executor.NO_RESULT_HANDLER);
    } catch (Exception e) {
        throw ExceptionFactory.wrapException("Error querying database.  Cause: " + e, e);
    } finally {
        ErrorContext.instance().reset();
    }
}
```

- `<1>` 处，调用 `Configuration#getMappedStatement(String id)` 方法，获得 MappedStatement 对象。代码如下：

  ```
  // DefaultSqlSession.java
  
  /**
   * MappedStatement 映射
   *
   * KEY：`${namespace}.${id}`
   */
  protected final Map<String, MappedStatement> mappedStatements = new StrictMap<>("Mapped Statements collection");
  
  public MappedStatement getMappedStatement(String id, boolean validateIncompleteStatements) {
      // 校验，保证所有 MappedStatement 已经构造完毕
      if (validateIncompleteStatements) {
          buildAllStatements();
      }
      // 获取 MappedStatement 对象
      return mappedStatements.get(id);
  }
  
  protected void buildAllStatements() {
      if (!incompleteResultMaps.isEmpty()) {
          synchronized (incompleteResultMaps) { // 保证 incompleteResultMaps 被解析完
              // This always throws a BuilderException.
              incompleteResultMaps.iterator().next().resolve();
          }
      }
      if (!incompleteCacheRefs.isEmpty()) {
          synchronized (incompleteCacheRefs) { // 保证 incompleteCacheRefs 被解析完
              // This always throws a BuilderException.
              incompleteCacheRefs.iterator().next().resolveCacheRef();
          }
      }
      if (!incompleteStatements.isEmpty()) {
          synchronized (incompleteStatements) { // 保证 incompleteStatements 被解析完
              // This always throws a BuilderException.
              incompleteStatements.iterator().next().parseStatementNode();
          }
      }
      if (!incompleteMethods.isEmpty()) {
          synchronized (incompleteMethods) { // 保证 incompleteMethods 被解析完
              // This always throws a BuilderException.
              incompleteMethods.iterator().next().resolve();
          }
      }
  }
  ```

  - 其中，`#buildAllStatements()` 方法，是用来保证所有 MappedStatement 已经构造完毕。不过艿艿，暂时没想到，什么情况下，会出现 MappedStatement 没被正确构建的情况。猜测有可能是防御性编程。

- `<2>` 处，调用 `Executor#query(...)` 方法，执行查询。

### 4.1.3 selectOne

```
// DefaultSqlSession.java

@Override
public <T> T selectOne(String statement) {
    return this.selectOne(statement, null);
}

@Override
public <T> T selectOne(String statement, Object parameter) {
    // Popular vote was to return null on 0 results and throw exception on too many.
    List<T> list = this.selectList(statement, parameter);
    if (list.size() == 1) {
        return list.get(0);
    } else if (list.size() > 1) {
        throw new TooManyResultsException("Expected one result (or null) to be returned by selectOne(), but found: " + list.size());
    } else {
        return null;
    }
}
```

- 内部调用 `#selectList(String statement, Object parameter)` 方法，进行实现。

### 4.1.4 selectMap

`#selectMap(...)` 方法，查询结果，并基于 Map 聚合结果。代码如下：

```
// DefaultSqlSession.java

@Override
public <K, V> Map<K, V> selectMap(String statement, String mapKey) {
    return this.selectMap(statement, null, mapKey, RowBounds.DEFAULT);
}

@Override
public <K, V> Map<K, V> selectMap(String statement, Object parameter, String mapKey) {
    return this.selectMap(statement, parameter, mapKey, RowBounds.DEFAULT);
}

@Override
public <K, V> Map<K, V> selectMap(String statement, Object parameter, String mapKey, RowBounds rowBounds) {
    // <1> 执行查询
    final List<? extends V> list = selectList(statement, parameter, rowBounds);
    // <2> 创建 DefaultMapResultHandler 对象
    final DefaultMapResultHandler<K, V> mapResultHandler = new DefaultMapResultHandler<>(mapKey,
            configuration.getObjectFactory(), configuration.getObjectWrapperFactory(), configuration.getReflectorFactory());
    // <3> 创建 DefaultResultContext 对象
    final DefaultResultContext<V> context = new DefaultResultContext<>();
    // <4> 遍历查询结果
    for (V o : list) {
        // 设置 DefaultResultContext 中
        context.nextResultObject(o);
        // 使用 DefaultMapResultHandler 处理结果的当前元素
        mapResultHandler.handleResult(context);
    }
    // <5> 返回结果
    return mapResultHandler.getMappedResults();
}
```

- `<1>` 处，调用 `#selectList(String statement, Object parameter, RowBounds rowBounds)` 方法，执行查询。

- `<2>` 处，创建 DefaultMapResultHandler 对象。

- `<3>` 处，创建 DefaultResultContext 对象。

- `<4>` 处，遍历查询结果，并调用 `DefaultMapResultHandler#handleResult(context)` 方法，将结果的当前元素，聚合成 Map 。代码如下：

  ```
  // DefaultMapResultHandler.java
  
  public class DefaultMapResultHandler<K, V> implements ResultHandler<V> {
  
      /**
       * 结果，基于 Map 聚合
       */
      private final Map<K, V> mappedResults;
      /**
       * {@link #mappedResults} 的 KEY 属性名
       */
      private final String mapKey;
      private final ObjectFactory objectFactory;
      private final ObjectWrapperFactory objectWrapperFactory;
      private final ReflectorFactory reflectorFactory;
  
      @SuppressWarnings("unchecked")
      public DefaultMapResultHandler(String mapKey, ObjectFactory objectFactory, ObjectWrapperFactory objectWrapperFactory, ReflectorFactory reflectorFactory) {
          this.objectFactory = objectFactory;
          this.objectWrapperFactory = objectWrapperFactory;
          this.reflectorFactory = reflectorFactory;
          // 创建 Map 对象
          this.mappedResults = objectFactory.create(Map.class);
          this.mapKey = mapKey;
      }
  
      @Override
      public void handleResult(ResultContext<? extends V> context) {
          // 获得 KEY 对应的属性
          final V value = context.getResultObject();
          final MetaObject mo = MetaObject.forObject(value, objectFactory, objectWrapperFactory, reflectorFactory);
          // TODO is that assignment always true?
          final K key = (K) mo.getValue(mapKey);
          // 添加到 mappedResults 中
          mappedResults.put(key, value);
      }
  
      public Map<K, V> getMappedResults() {
          return mappedResults;
      }
  
  }
  ```

- `<5>` 处，返回结果。

### 4.1.5 selectCursor

```
// DefaultSqlSession.java

@Override
public <T> Cursor<T> selectCursor(String statement) {
    return selectCursor(statement, null);
}

@Override
public <T> Cursor<T> selectCursor(String statement, Object parameter) {
    return selectCursor(statement, parameter, RowBounds.DEFAULT);
}

@Override
public <T> Cursor<T> selectCursor(String statement, Object parameter, RowBounds rowBounds) {
    try {
        // <1> 获得 MappedStatement 对象
        MappedStatement ms = configuration.getMappedStatement(statement);
        // <2> 执行查询
        Cursor<T> cursor = executor.queryCursor(ms, wrapCollection(parameter), rowBounds);
        // <3> 添加 cursor 到 cursorList 中
        registerCursor(cursor);
        return cursor;
    } catch (Exception e) {
        throw ExceptionFactory.wrapException("Error querying database.  Cause: " + e, e);
    } finally {
        ErrorContext.instance().reset();
    }
}
```

- `<1>` 处，调用 `Configuration#getMappedStatement(String id)` 方法，获得 MappedStatement 对象。

- `<2>` 处，调用 `Executor#queryCursor(...)` 方法，执行查询。

- `<3>` 处，调用 `#registerCursor(Cursor<T> cursor)` 方法，添加 `cursor` 到 `cursorList` 中。代码如下：

  ```
  // DefaultSqlSession.java
  
  private <T> void registerCursor(Cursor<T> cursor) {
      if (cursorList == null) {
          cursorList = new ArrayList<>();
      }
      cursorList.add(cursor);
  }
  ```

### 4.1.6 select

`#select(..., ResultHandler handler)` 方法，执行查询，使用传入的 `handler` 方法参数，对结果进行处理。代码如下：

```
// DefaultSqlSession.java

@Override
public void select(String statement, Object parameter, ResultHandler handler) {
    select(statement, parameter, RowBounds.DEFAULT, handler);
}

@Override
public void select(String statement, ResultHandler handler) {
    select(statement, null, RowBounds.DEFAULT, handler);
}

@Override
public void select(String statement, Object parameter, RowBounds rowBounds, ResultHandler handler) {
    try {
        // 获得 MappedStatement 对象
        MappedStatement ms = configuration.getMappedStatement(statement);
        // 执行查询
        executor.query(ms, wrapCollection(parameter), rowBounds, handler);
    } catch (Exception e) {
        throw ExceptionFactory.wrapException("Error querying database.  Cause: " + e, e);
    } finally {
        ErrorContext.instance().reset();
    }
}
```

### 4.1.7 wrapCollection

在上述的查询方法中，我们都可以看到一个 `#wrapCollection(final Object object)` 方法，若参数 `object` 是 Collection、Array、Map 参数类型的情况下，包装成 Map 返回。代码如下：

```
// DefaultSqlSession.java

private Object wrapCollection(final Object object) {
    if (object instanceof Collection) {
        // 如果是集合，则添加到 collection 中
        StrictMap<Object> map = new StrictMap<>();
        map.put("collection", object);
        // 如果是 List ，则添加到 list 中
        if (object instanceof List) {
            map.put("list", object);
        }
        return map;
    } else if (object != null && object.getClass().isArray()) {
        // 如果是 Array ，则添加到 array 中
        StrictMap<Object> map = new StrictMap<>();
        map.put("array", object);
        return map;
    }
    return object;
}
```

### 4.1.8 update

```
// DefaultSqlSession.java

@Override
public int update(String statement) {
    return update(statement, null);
}

@Override
public int update(String statement, Object parameter) {
    try {
        // <1> 标记 dirty ，表示执行过写操作
        dirty = true;
        // <2> 获得 MappedStatement 对象
        MappedStatement ms = configuration.getMappedStatement(statement);
        // <3> 执行更新操作
        return executor.update(ms, wrapCollection(parameter));
    } catch (Exception e) {
        throw ExceptionFactory.wrapException("Error updating database.  Cause: " + e, e);
    } finally {
        ErrorContext.instance().reset();
    }
}
```

- `<1>` 处，标记 `dirty` ，表示执行过写操作。该参数，会在事务的提交和回滚，产生其用途。
- `<2>` 处，获得 MappedStatement 对象。
- `<3>` 处，调用 `Executor#update(MappedStatement ms, Object parameter)` 方法，执行更新操作。

### 4.1.9 insert

```
// DefaultSqlSession.java

@Override
public int insert(String statement) {
    return insert(statement, null);
}

@Override
public int insert(String statement, Object parameter) {
    return update(statement, parameter);
}
```

- 基于 `#update(...)` 方法来实现。

### 4.1.10 delete

```
// DefaultSqlSession.java

@Override
public int delete(String statement) {
    return update(statement, null);
}

@Override
public int delete(String statement, Object parameter) {
    return update(statement, parameter);
}
```

- 基于 `#update(...)` 方法来实现。

### 4.1.11 flushStatements

`#flushStatements()` 方法，提交批处理。代码如下：

```
// DefaultSqlSession.java

@Override
public List<BatchResult> flushStatements() {
    try {
        return executor.flushStatements();
    } catch (Exception e) {
        throw ExceptionFactory.wrapException("Error flushing statements.  Cause: " + e, e);
    } finally {
        ErrorContext.instance().reset();
    }
}
```

### 4.1.12 commit

```
// DefaultSqlSession.java

@Override
public void commit() {
    commit(false);
}

@Override
public void commit(boolean force) {
    try {
        // 提交事务
        executor.commit(isCommitOrRollbackRequired(force));
        // 标记 dirty 为 false
        dirty = false;
    } catch (Exception e) {
        throw ExceptionFactory.wrapException("Error committing transaction.  Cause: " + e, e);
    } finally {
        ErrorContext.instance().reset();
    }
}
```

- 其中，`#isCommitOrRollbackRequired(boolean force)` 方法，判断是否执行提交或回滚。代码如下：

  ```
  // DefaultSqlSession.java
  
  private boolean isCommitOrRollbackRequired(boolean force) {
      return (!autoCommit && dirty) || force;
  }
  ```

  - 有两种情况需要触发：
  - 1）未开启自动提交，并且数据发生写操作
  - 2）强制提交

### 4.1.13 rollback

```
// DefaultSqlSession.java

@Override
public void rollback() {
    rollback(false);
}

@Override
public void rollback(boolean force) {
    try {
        // 回滚事务
        executor.rollback(isCommitOrRollbackRequired(force));
        // 标记 dirty 为 false
        dirty = false;
    } catch (Exception e) {
        throw ExceptionFactory.wrapException("Error rolling back transaction.  Cause: " + e, e);
    } finally {
        ErrorContext.instance().reset();
    }
}
```

### 4.1.14 close

`#close()` 方法，关闭会话。代码如下：

```
// DefaultSqlSession.java

@Override
public void close() {
    try {
        // <1> 关闭执行器
        executor.close(isCommitOrRollbackRequired(false));
        // <2> 关闭所有游标
        closeCursors();
        // <3> 重置 dirty 为 false
        dirty = false;
    } finally {
        ErrorContext.instance().reset();
    }
}
```

- `<1>` 处，调用 `Executor#close(boolean forceRollback)` 方法，关闭执行器。并且，根据 `forceRollback` 参数，是否进行事务回滚。

- `<2>` 处，调用 `#closeCursors()` 方法，关闭所有游标。代码如下：

  ```
  // DefaultSqlSession.java
  
  private void closeCursors() {
      if (cursorList != null && cursorList.size() != 0) {
          for (Cursor<?> cursor : cursorList) {
              try {
                  cursor.close();
              } catch (IOException e) {
                  throw ExceptionFactory.wrapException("Error closing cursor.  Cause: " + e, e);
              }
          }
          cursorList.clear();
      }
  }
  ```

- `<3>` 处，重置 `dirty` 为 `false` 。

### 4.1.15 getConfiguration

```
// DefaultSqlSession.java

@Override
public Configuration getConfiguration() {
    return configuration;
}
```

### 4.1.16 getMapper

```
// DefaultSqlSession.java

@Override
public <T> T getMapper(Class<T> type) {
    return configuration.getMapper(type, this);
}
```

### 4.1.17 getConnection

```
// DefaultSqlSession.java

@Override
public Connection getConnection() {
    try {
        return executor.getTransaction().getConnection();
    } catch (SQLException e) {
        throw ExceptionFactory.wrapException("Error getting a new connection.  Cause: " + e, e);
    }
}
```

### 4.1.18 clearCache

```
// DefaultSqlSession.java

@Override
public void clearCache() {
    executor.clearLocalCache();
}
```

# 5. SqlSessionManager

`org.apache.ibatis.session.SqlSessionManager` ，实现 SqlSessionFactory、SqlSession 接口，SqlSession 管理器。所以，从这里已经可以看出，SqlSessionManager 是 SqlSessionFactory 和 SqlSession 的职能相加。

## 5.1 构造方法

```
// SqlSessionManager.java

private final SqlSessionFactory sqlSessionFactory;
private final SqlSession sqlSessionProxy;

/**
 * 线程变量，当前线程的 SqlSession 对象
 */
private final ThreadLocal<SqlSession> localSqlSession = new ThreadLocal<>(); // <1>

private SqlSessionManager(SqlSessionFactory sqlSessionFactory) {
    this.sqlSessionFactory = sqlSessionFactory;
    // <2> 创建 SqlSession 的代理对象
    this.sqlSessionProxy = (SqlSession) Proxy.newProxyInstance(
            SqlSessionFactory.class.getClassLoader(),
            new Class[]{SqlSession.class},
            new SqlSessionInterceptor());
}
```

- 比较有意思的有两点，我们逐条来看。
- `<1>` 处，`localSqlSession` 属性，线程变量，记录当前线程的 SqlSession 对象。
- `<2>` 处，创建 SqlSession 的代理对象，而方法的拦截器是 SqlSessionInterceptor 类。详细解析，见 [「5.6 SqlSessionInterceptor」](http://svip.iocoder.cn/MyBatis/session/#) 。

## 5.2 newInstance

`#newInstance(...)` **静态**方法，创建 SqlSessionManager 对象。代码如下：

```
// SqlSessionManager.java

public static SqlSessionManager newInstance(Reader reader) {
    return new SqlSessionManager(new SqlSessionFactoryBuilder().build(reader, null, null));
}

public static SqlSessionManager newInstance(Reader reader, String environment) {
    return new SqlSessionManager(new SqlSessionFactoryBuilder().build(reader, environment, null));
}

public static SqlSessionManager newInstance(Reader reader, Properties properties) {
    return new SqlSessionManager(new SqlSessionFactoryBuilder().build(reader, null, properties));
}

public static SqlSessionManager newInstance(InputStream inputStream) {
    return new SqlSessionManager(new SqlSessionFactoryBuilder().build(inputStream, null, null));
}

public static SqlSessionManager newInstance(InputStream inputStream, String environment) {
    return new SqlSessionManager(new SqlSessionFactoryBuilder().build(inputStream, environment, null));
}

public static SqlSessionManager newInstance(InputStream inputStream, Properties properties) {
    return new SqlSessionManager(new SqlSessionFactoryBuilder().build(inputStream, null, properties));
}

public static SqlSessionManager newInstance(SqlSessionFactory sqlSessionFactory) {
    return new SqlSessionManager(sqlSessionFactory);
}
```

- 代码比较简单，胖友自己瞅瞅。

## 5.3 startManagedSession

`#startManagedSession(...)` 方法，发起一个**可被管理的** SqlSession 。代码如下：

```
// SqlSessionManager.java

public void startManagedSession() {
    this.localSqlSession.set(openSession());
}

public void startManagedSession(boolean autoCommit) {
    this.localSqlSession.set(openSession(autoCommit));
}

public void startManagedSession(Connection connection) {
    this.localSqlSession.set(openSession(connection));
}

public void startManagedSession(TransactionIsolationLevel level) {
    this.localSqlSession.set(openSession(level));
}

public void startManagedSession(ExecutorType execType) {
    this.localSqlSession.set(openSession(execType));
}

public void startManagedSession(ExecutorType execType, boolean autoCommit) {
    this.localSqlSession.set(openSession(execType, autoCommit));
}

public void startManagedSession(ExecutorType execType, TransactionIsolationLevel level) {
    this.localSqlSession.set(openSession(execType, level));
}

public void startManagedSession(ExecutorType execType, Connection connection) {
    this.localSqlSession.set(openSession(execType, connection));
}
```

- 可能胖友很难理解“可被管理”的 SqlSession 的意思？继续往下看。

## 5.4 对 SqlSessionFactory 的实现方法

```
// SqlSessionManager.java

@Override
public SqlSession openSession() {
    return sqlSessionFactory.openSession();
}

@Override
public SqlSession openSession(boolean autoCommit) {
    return sqlSessionFactory.openSession(autoCommit);
}

@Override
public SqlSession openSession(Connection connection) {
    return sqlSessionFactory.openSession(connection);
}

@Override
public SqlSession openSession(TransactionIsolationLevel level) {
    return sqlSessionFactory.openSession(level);
}

@Override
public SqlSession openSession(ExecutorType execType) {
    return sqlSessionFactory.openSession(execType);
}

@Override
public SqlSession openSession(ExecutorType execType, boolean autoCommit) {
    return sqlSessionFactory.openSession(execType, autoCommit);
}

@Override
public SqlSession openSession(ExecutorType execType, TransactionIsolationLevel level) {
    return sqlSessionFactory.openSession(execType, level);
}

@Override
public SqlSession openSession(ExecutorType execType, Connection connection) {
    return sqlSessionFactory.openSession(execType, connection);
}
```

- 直接调用 `sqlSessionFactory` 对应的方法即可。

## 5.5 对 SqlSession 的实现方法

```
// SqlSessionManager.java

@Override
public <T> T selectOne(String statement) {
    return sqlSessionProxy.selectOne(statement);
}

@Override
public <T> T selectOne(String statement, Object parameter) {
    return sqlSessionProxy.selectOne(statement, parameter);
}

@Override
public <K, V> Map<K, V> selectMap(String statement, String mapKey) {
    return sqlSessionProxy.selectMap(statement, mapKey);
}

@Override
public <K, V> Map<K, V> selectMap(String statement, Object parameter, String mapKey) {
    return sqlSessionProxy.selectMap(statement, parameter, mapKey);
}

@Override
public <K, V> Map<K, V> selectMap(String statement, Object parameter, String mapKey, RowBounds rowBounds) {
    return sqlSessionProxy.selectMap(statement, parameter, mapKey, rowBounds);
}

@Override
public <T> Cursor<T> selectCursor(String statement) {
    return sqlSessionProxy.selectCursor(statement);
}

@Override
public <T> Cursor<T> selectCursor(String statement, Object parameter) {
    return sqlSessionProxy.selectCursor(statement, parameter);
}

@Override
public <T> Cursor<T> selectCursor(String statement, Object parameter, RowBounds rowBounds) {
    return sqlSessionProxy.selectCursor(statement, parameter, rowBounds);
}

@Override
public <E> List<E> selectList(String statement) {
    return sqlSessionProxy.selectList(statement);
}

@Override
public <E> List<E> selectList(String statement, Object parameter) {
    return sqlSessionProxy.selectList(statement, parameter);
}

@Override
public <E> List<E> selectList(String statement, Object parameter, RowBounds rowBounds) {
    return sqlSessionProxy.selectList(statement, parameter, rowBounds);
}

@Override
public void select(String statement, ResultHandler handler) {
    sqlSessionProxy.select(statement, handler);
}

@Override
public void select(String statement, Object parameter, ResultHandler handler) {
    sqlSessionProxy.select(statement, parameter, handler);
}

@Override
public void select(String statement, Object parameter, RowBounds rowBounds, ResultHandler handler) {
    sqlSessionProxy.select(statement, parameter, rowBounds, handler);
}

@Override
public int insert(String statement) {
    return sqlSessionProxy.insert(statement);
}

@Override
public int insert(String statement, Object parameter) {
    return sqlSessionProxy.insert(statement, parameter);
}

@Override
public int update(String statement) {
    return sqlSessionProxy.update(statement);
}

@Override
public int update(String statement, Object parameter) {
    return sqlSessionProxy.update(statement, parameter);
}

@Override
public int delete(String statement) {
    return sqlSessionProxy.delete(statement);
}

@Override
public int delete(String statement, Object parameter) {
    return sqlSessionProxy.delete(statement, parameter);
}

@Override
public <T> T getMapper(Class<T> type) {
    return getConfiguration().getMapper(type, this);
}

@Override
public Connection getConnection() {
    final SqlSession sqlSession = localSqlSession.get();
    if (sqlSession == null) {
        throw new SqlSessionException("Error:  Cannot get connection.  No managed session is started.");
    }
    return sqlSession.getConnection();
}

@Override
public void clearCache() {
    final SqlSession sqlSession = localSqlSession.get();
    if (sqlSession == null) {
        throw new SqlSessionException("Error:  Cannot clear the cache.  No managed session is started.");
    }
    sqlSession.clearCache();
}

@Override
public void commit() {
    final SqlSession sqlSession = localSqlSession.get();
    if (sqlSession == null) {
        throw new SqlSessionException("Error:  Cannot commit.  No managed session is started.");
    }
    sqlSession.commit();
}

@Override
public void commit(boolean force) {
    final SqlSession sqlSession = localSqlSession.get();
    if (sqlSession == null) {
        throw new SqlSessionException("Error:  Cannot commit.  No managed session is started.");
    }
    sqlSession.commit(force);
}

@Override
public void rollback() {
    final SqlSession sqlSession = localSqlSession.get();
    if (sqlSession == null) {
        throw new SqlSessionException("Error:  Cannot rollback.  No managed session is started.");
    }
    sqlSession.rollback();
}

@Override
public void rollback(boolean force) {
    final SqlSession sqlSession = localSqlSession.get();
    if (sqlSession == null) {
        throw new SqlSessionException("Error:  Cannot rollback.  No managed session is started.");
    }
    sqlSession.rollback(force);
}

@Override
public List<BatchResult> flushStatements() {
    final SqlSession sqlSession = localSqlSession.get();
    if (sqlSession == null) {
        throw new SqlSessionException("Error:  Cannot rollback.  No managed session is started.");
    }
    return sqlSession.flushStatements();
}

@Override
public void close() {
    final SqlSession sqlSession = localSqlSession.get();
    if (sqlSession == null) {
        throw new SqlSessionException("Error:  Cannot close.  No managed session is started.");
    }
    try {
        sqlSession.close();
    } finally {
        localSqlSession.set(null);
    }
}
```

- 调用 `localSqlSession` 中的 SqlSession 对象，对应的方法。

## 5.6 SqlSessionInterceptor

SqlSessionInterceptor ，是 SqlSessionManager 内部类，实现 InvocationHandler 接口，实现对 `sqlSessionProxy` 的调用的拦截。代码如下：

```
// SqlSessionManager.java

private class SqlSessionInterceptor implements InvocationHandler {

    public SqlSessionInterceptor() {
        // Prevent Synthetic Access
    }

    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        // 情况一，如果 localSqlSession 中存在 SqlSession 对象，说明是自管理模式
        final SqlSession sqlSession = SqlSessionManager.this.localSqlSession.get();
        if (sqlSession != null) {
            try {
                // 直接执行方法
                return method.invoke(sqlSession, args);
            } catch (Throwable t) {
                throw ExceptionUtil.unwrapThrowable(t);
            }
        // 情况二，如果没有 SqlSession 对象，则直接创建一个
        } else {
            // 创建新的 SqlSession 对象
            try (SqlSession autoSqlSession = openSession()) { // 同时，通过 try 的语法糖，实现结束时，关闭 SqlSession 对象
                try {
                    // 执行方法
                    final Object result = method.invoke(autoSqlSession, args);
                    // 提交 SqlSession 对象
                    autoSqlSession.commit();
                    return result;
                } catch (Throwable t) {
                    // 发生异常时，回滚
                    autoSqlSession.rollback();
                    throw ExceptionUtil.unwrapThrowable(t);
                }
            }
        }
    }

}
```

- 分成两种情况，胖友直接看代码注释。hoho 。

## 5.7 嘿嘿嘿

貌似，SqlSessionManager 在实际项目中木有什么用。这里，胖友就是去理解，以及动态代理的使用。

# 6. Configuration

`org.apache.ibatis.session.Configuration` ，MyBatis 配置对象。在前面已经不断在介绍了，这里就不重复了。

# 7. MapperMethod

在前面的文章中，我们很**分块**的介绍了 MyBatis 的各个组件，那么一条 SQL 命令的完整流程是怎么样的呢？如下图所示：

> FROM 祖大俊 [《Mybatis3.3.x技术内幕（十一）：执行一个Sql命令的完整流程》](https://my.oschina.net/zudajun/blog/670373)
>
> [![流程图](http://static.iocoder.cn/images/MyBatis/2020_03_15/02.png)](http://static.iocoder.cn/images/MyBatis/2020_03_15/02.png)流程图

那么从这个图中可以看出，MapperMethod 在里面扮演了非常重要的角色，将方法转换成对应的 SqlSession 方法，并返回执行结果。实际上，在 [《精尽 MyBatis 源码分析 —— Binding 模块》](http://svip.iocoder.cn/MyBatis/binding-package) 中，我们已经介绍了 MapperMethod 类，但是因为 `#execute(SqlSession sqlSession, Object[] args)` 方法，涉及内容较多，所以没有详细解析。那么此处，让我们撸起这个方法。代码如下：

```
// MapperMethod.java

public Object execute(SqlSession sqlSession, Object[] args) {
    Object result;
    switch (command.getType()) {
        case INSERT: {
            // 转换参数
            Object param = method.convertArgsToSqlCommandParam(args);
            // 执行 INSERT 操作
            // 转换 rowCount
            result = rowCountResult(sqlSession.insert(command.getName(), param));
            break;
        }
        case UPDATE: {
            // <1.1> 转换参数
            Object param = method.convertArgsToSqlCommandParam(args);
            // <1.2> 执行更新
            // <1.3> 转换 rowCount
            result = rowCountResult(sqlSession.update(command.getName(), param));
            break;
        }
        case DELETE: {
            // 转换参数
            Object param = method.convertArgsToSqlCommandParam(args);
            // 转换 rowCount
            result = rowCountResult(sqlSession.delete(command.getName(), param));
            break;
        }
        case SELECT:
            // <2.1> 无返回，并且有 ResultHandler 方法参数，则将查询的结果，提交给 ResultHandler 进行处理
            if (method.returnsVoid() && method.hasResultHandler()) {
                executeWithResultHandler(sqlSession, args);
                result = null;
            // <2.2> 执行查询，返回列表
            } else if (method.returnsMany()) {
                result = executeForMany(sqlSession, args);
            // <2.3> 执行查询，返回 Map
            } else if (method.returnsMap()) {
                result = executeForMap(sqlSession, args);
            // <2.4> 执行查询，返回 Cursor
            } else if (method.returnsCursor()) {
                result = executeForCursor(sqlSession, args);
            // <2.5> 执行查询，返回单个对象
            } else {
                // 转换参数
                Object param = method.convertArgsToSqlCommandParam(args);
                // 查询单条
                result = sqlSession.selectOne(command.getName(), param);
                if (method.returnsOptional() &&
                        (result == null || !method.getReturnType().equals(result.getClass()))) {
                    result = Optional.ofNullable(result);
                }
            }
            break;
        case FLUSH:
            result = sqlSession.flushStatements();
            break;
        default:
            throw new BindingException("Unknown execution method for: " + command.getName());
    }
    // 返回结果为 null ，并且返回类型为基本类型，则抛出 BindingException 异常
    if (result == null && method.getReturnType().isPrimitive() && !method.returnsVoid()) {
        throw new BindingException("Mapper method '" + command.getName()
                + " attempted to return null from a method with a primitive return type (" + method.getReturnType() + ").");
    }
    // 返回结果
    return result;
}
```

- `INSERT` 命令类型

  - `<1.1>` 处，调用 `MethodSignature#convertArgsToSqlCommandParam(args)` 方法，转换参数。

  - `<1.2>` 处，调用 `SqlSession#insert(String statement, Object parameter)` 方法，执行 INSERT 操作。

  - `<1.3>` 处，调用 `#rowCountResult(int rowCount)` 方法，将返回的行变更数，转换成方法实际要返回的类型。代码如下：

    ```
    // MapperMethod.java
    
    private Object rowCountResult(int rowCount) {
        final Object result;
        if (method.returnsVoid()) { // Void 情况，不用返回
            result = null;
        } else if (Integer.class.equals(method.getReturnType()) || Integer.TYPE.equals(method.getReturnType())) { // Int
            result = rowCount;
        } else if (Long.class.equals(method.getReturnType()) || Long.TYPE.equals(method.getReturnType())) { // Long
            result = (long) rowCount;
        } else if (Boolean.class.equals(method.getReturnType()) || Boolean.TYPE.equals(method.getReturnType())) { // Boolean
            result = rowCount > 0;
        } else {
            throw new BindingException("Mapper method '" + command.getName() + "' has an unsupported return type: " + method.getReturnType());
        }
        return result;
    }
    ```

    - 根据方法的返回类型，做相应的转化处理。

- `UPDATE` 命令类型，同 `INSERT` 命令类型。

- `DELETE` 命令类型，同 `INSERT` 命令类型。

- ```
  SELECT
  ```

   

  命令类型

  - `<2.1>` 处，无返回，并且有 ResultHandler 方法参数，则调用 `#executeWithResultHandler(SqlSession sqlSession, Object[] args)` 方法，将查询的结果，提交给 ResultHandler 进行处理。详细解析，见 [「7.1 executeWithResultHandler」](http://svip.iocoder.cn/MyBatis/session/#) 。
  - `<2.2>` 处，调用 `#executeForMany(SqlSession sqlSession, Object[] args)` 方法，执行查询，返回列表。详细解析，见 [「7.2 executeForMany」](http://svip.iocoder.cn/MyBatis/session/#) 。
  - `<2.3>` 处，调用 `#executeForMap(SqlSession sqlSession, Object[] args)` 方法，执行查询，返回 Map。详细解析，见 [「7.3 executeForMap」](http://svip.iocoder.cn/MyBatis/session/#) 。
  - `<2.4>` 处，调用 `#executeForCursor(SqlSession sqlSession, Object[] args)` 方法，执行查询，返回 Map。详细解析，见 [「7.4 executeForCursor」](http://svip.iocoder.cn/MyBatis/session/#) 。
  - `<2.5>` 处，执行查询，返回单个对象。此处是，对 SqlSession 的 [「4.1.3 selectOne」](http://svip.iocoder.cn/MyBatis/session/#) 方法的封装。

## 7.1 executeWithResultHandler

```
// MapperMethod.java

private void executeWithResultHandler(SqlSession sqlSession, Object[] args) {
    // 获得 MappedStatement 对象
    MappedStatement ms = sqlSession.getConfiguration().getMappedStatement(command.getName());
    if (!StatementType.CALLABLE.equals(ms.getStatementType()) // 校验存储过程的情况。不符合，抛出 BindingException 异常
            && void.class.equals(ms.getResultMaps().get(0).getType())) {
        throw new BindingException("method " + command.getName()
                + " needs either a @ResultMap annotation, a @ResultType annotation,"
                + " or a resultType attribute in XML so a ResultHandler can be used as a parameter.");
    }
    // 转换参数
    Object param = method.convertArgsToSqlCommandParam(args);
    // 执行 SELECT 操作
    if (method.hasRowBounds()) {
        RowBounds rowBounds = method.extractRowBounds(args);
        sqlSession.select(command.getName(), param, rowBounds, method.extractResultHandler(args));
    } else {
        sqlSession.select(command.getName(), param, method.extractResultHandler(args));
    }
}
```

- 对 SqlSession 的 [「4.1.6 select」](http://svip.iocoder.cn/MyBatis/session/#) 方法的封装。

## 7.2 executeForMany

```
// MapperMethod.java

private <E> Object executeForMany(SqlSession sqlSession, Object[] args) {
    List<E> result;
    // 转换参数
    Object param = method.convertArgsToSqlCommandParam(args);
    // 执行 SELECT 操作
    if (method.hasRowBounds()) {
        RowBounds rowBounds = method.extractRowBounds(args);
        result = sqlSession.selectList(command.getName(), param, rowBounds);
    } else {
        result = sqlSession.selectList(command.getName(), param);
    }
    // issue #510 Collections & arrays support
    // 封装 Array 或 Collection 结果
    if (!method.getReturnType().isAssignableFrom(result.getClass())) {
        if (method.getReturnType().isArray()) { // 情况一，Array
            return convertToArray(result);
        } else {
            return convertToDeclaredCollection(sqlSession.getConfiguration(), result); // 情况二，Collection
        }
    }
    // 直接返回的结果
    return result; // 情况三，默认
}
```

- 对 SqlSession 的 [「4.1.2 selectList」](http://svip.iocoder.cn/MyBatis/session/#) 方法的封装。

- 情况一，Array 。代码如下：

  ```
  // MapperMethod.java
  
  private <E> Object convertToArray(List<E> list) {
      Class<?> arrayComponentType = method.getReturnType().getComponentType();
      Object array = Array.newInstance(arrayComponentType, list.size());
      if (arrayComponentType.isPrimitive()) {
          for (int i = 0; i < list.size(); i++) {
              Array.set(array, i, list.get(i));
          }
          return array;
      } else {
          return list.toArray((E[]) array);
      }
  }
  ```

- 情况二，Collection 。代码如下：

  ```
  // MapperMethod.java
  
  private <E> Object convertToDeclaredCollection(Configuration config, List<E> list) {
      Object collection = config.getObjectFactory().create(method.getReturnType());
      MetaObject metaObject = config.newMetaObject(collection);
      metaObject.addAll(list);
      return collection;
  }
  ```

## 7.3 executeForMap

```
// MapperMethod.java

private <K, V> Map<K, V> executeForMap(SqlSession sqlSession, Object[] args) {
    Map<K, V> result;
    // 转换参数
    Object param = method.convertArgsToSqlCommandParam(args);
    // 执行 SELECT 操作
    if (method.hasRowBounds()) {
        RowBounds rowBounds = method.extractRowBounds(args);
        result = sqlSession.<K, V>selectMap(command.getName(), param, method.getMapKey(), rowBounds);
    } else {
        result = sqlSession.<K, V>selectMap(command.getName(), param, method.getMapKey());
    }
    return result;
}
```

- 对 SqlSession 的 [「4.1.4 selectMap」](http://svip.iocoder.cn/MyBatis/session/#) 方法的封装。

## 7.4 executeForCursor

```
// MapperMethod.java

private <T> Cursor<T> executeForCursor(SqlSession sqlSession, Object[] args) {
    Cursor<T> result;
    // 转换参数
    Object param = method.convertArgsToSqlCommandParam(args);
    // 执行 SELECT 操作
    if (method.hasRowBounds()) {
        RowBounds rowBounds = method.extractRowBounds(args);
        result = sqlSession.selectCursor(command.getName(), param, rowBounds);
    } else {
        result = sqlSession.selectCursor(command.getName(), param);
    }
    return result;
}
```

# 总结

参考和推荐如下文章：

- 祖大俊 [《Mybatis3.3.x技术内幕（一）：SqlSession和SqlSessionFactory列传》](https://my.oschina.net/zudajun/blog/665956)
- 祖大俊 [《Mybatis3.3.x技术内幕（十一）：执行一个Sql命令的完整流程》](https://my.oschina.net/zudajun/blog/670373)
- 田小波 [《MyBatis 源码分析 - SQL 的执行过程》](https://www.tianxiaobo.com/2018/08/17/MyBatis-源码分析-SQL-的执行过程/)
- 无忌 [《MyBatis 源码解读之 SqlSession》](https://my.oschina.net/wenjinglian/blog/1631901)
- 徐郡明 [《MyBatis 技术内幕》](https://item.jd.com/12125531.html) 的 [「3.7 接口层」](http://svip.iocoder.cn/MyBatis/session/#) 小节