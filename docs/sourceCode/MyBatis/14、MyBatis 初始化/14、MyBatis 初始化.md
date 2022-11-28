# 1、加载 mybatis-config

## 1. 概述

从本文开始，我们来分享 MyBatis 初始化的流程。在 [《精尽 MyBatis 源码分析 —— 项目结构一览》](http://svip.iocoder.cn/MyBatis/intro) 中，我们简单介绍这个流程如下：

> 在 MyBatis 初始化过程中，会加载 `mybatis-config.xml` 配置文件、映射配置文件以及 Mapper 接口中的注解信息，解析后的配置信息会形成相应的对象并保存到 Configuration 对象中。例如：
>
> - `<resultMap>`节点(即 ResultSet 的映射规则) 会被解析成 ResultMap 对象。
> - `<result>` 节点(即属性映射)会被解析成 ResultMapping 对象。
>
> 之后，利用该 Configuration 对象创建 SqlSessionFactory对象。待 MyBatis 初始化之后，开发人员可以通过初始化得到 SqlSessionFactory 创建 SqlSession 对象并完成数据库操作。

- 对应 `builder` 模块，为配置**解析过程**
- 对应 `mapping` 模块，主要为 SQL 操作解析后的**映射**。

因为整个 MyBatis 的初始化流程涉及代码颇多，所以拆分成三篇文章：

- 加载 `mybatis-config.xml` 配置文件。
- 加载 Mapper 映射配置文件。
- 加载 Mapper 接口中的注解信息。

本文就主要分享第一部分「加载 `mybatis-config.xml` 配置文件」。

------

MyBatis 的初始化流程的**入口**是 SqlSessionFactoryBuilder 的 `#build(Reader reader, String environment, Properties properties)` 方法，代码如下：

> SqlSessionFactoryBuilder 中，build 方法有多种重载方式。这里就选取一个。

```
// SqlSessionFactoryBuilder.java

/**
 * 构造 SqlSessionFactory 对象
 *
 * @param reader Reader 对象
 * @param environment 环境
 * @param properties Properties 变量
 * @return SqlSessionFactory 对象
 */
public SqlSessionFactory build(Reader reader, String environment, Properties properties) {
    try {
        // <1> 创建 XMLConfigBuilder 对象
        XMLConfigBuilder parser = new XMLConfigBuilder(reader, environment, properties);
        // <2> 执行 XML 解析
        // <3> 创建 DefaultSqlSessionFactory 对象
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
```

- `<1>` 处，创建 XMLConfigBuilder 对象。
- `<2>` 处，调用 `XMLConfigBuilder#parse()` 方法，执行 XML 解析，返回 Configuration 对象。
- `<3>` 处，创建 DefaultSqlSessionFactory 对象。
- 本文的重点是 `<1>` 和 `<2>` 处，即 XMLConfigBuilder 类。详细解析，见 [「3. XMLConfigBuilder」](http://svip.iocoder.cn/MyBatis/builder-package-1/#) 。

## 2. BaseBuilder

`org.apache.ibatis.builder.BaseBuilder` ，基础构造器抽象类，为子类提供通用的工具类。

为什么不直接讲 XMLConfigBuilder ，而是先讲 BaseBuilder 呢？因为，BaseBuilder 是 XMLConfigBuilder 的父类，并且它还有其他的子类。如下图所示：[![BaseBuilder 类图](http://static.iocoder.cn/images/MyBatis/2020_02_10/01.png)](http://static.iocoder.cn/images/MyBatis/2020_02_10/01.png)BaseBuilder 类图

#### 2.1 构造方法

```
// BaseBuilder.java

/**
 * MyBatis Configuration 对象
 */
protected final Configuration configuration;
protected final TypeAliasRegistry typeAliasRegistry;
protected final TypeHandlerRegistry typeHandlerRegistry;

public BaseBuilder(Configuration configuration) {
    this.configuration = configuration;
    this.typeAliasRegistry = this.configuration.getTypeAliasRegistry();
    this.typeHandlerRegistry = this.configuration.getTypeHandlerRegistry();
}
```

- `configuration` 属性，MyBatis Configuration 对象。XML 和注解中解析到的配置，最终都会设置到 [`org.apache.ibatis.session.Configuration`](https://github.com/YunaiV/mybatis-3/blob/master/src/main/java/org/apache/ibatis/session/Configuration.java) 中。感兴趣的胖友，可以先点击瞅一眼。抽完之后，马上回来。

#### 2.2 parseExpression

`#parseExpression(String regex, String defaultValue)` 方法，创建正则表达式。代码如下：

```
// BaseBuilder.java

/**
 * 创建正则表达式
 *
 * @param regex 指定表达式
 * @param defaultValue 默认表达式
 * @return 正则表达式
 */
@SuppressWarnings("SameParameterValue")
protected Pattern parseExpression(String regex, String defaultValue) {
    return Pattern.compile(regex == null ? defaultValue : regex);
}
```

#### 2.3 xxxValueOf

`#xxxValueOf(...)` 方法，将字符串转换成对应的数据类型的值。代码如下：

```
// BaseBuilder.java

protected Boolean booleanValueOf(String value, Boolean defaultValue) {
    return value == null ? defaultValue : Boolean.valueOf(value);
}

protected Integer integerValueOf(String value, Integer defaultValue) {
    return value == null ? defaultValue : Integer.valueOf(value);
}

protected Set<String> stringSetValueOf(String value, String defaultValue) {
    value = (value == null ? defaultValue : value);
    return new HashSet<>(Arrays.asList(value.split(",")));
}
```

#### 2.4 resolveJdbcType

`#resolveJdbcType(String alias)` 方法，解析对应的 JdbcType 类型。代码如下：

```
// BaseBuilder.java

protected JdbcType resolveJdbcType(String alias) {
    if (alias == null) {
        return null;
    }
    try {
        return JdbcType.valueOf(alias);
    } catch (IllegalArgumentException e) {
        throw new BuilderException("Error resolving JdbcType. Cause: " + e, e);
    }
}
```

#### 2.5 resolveResultSetType

`#resolveResultSetType(String alias)` 方法，解析对应的 ResultSetType 类型。代码如下：

```
// BaseBuilder.java

protected ResultSetType resolveResultSetType(String alias) {
    if (alias == null) {
        return null;
    }
    try {
        return ResultSetType.valueOf(alias);
    } catch (IllegalArgumentException e) {
        throw new BuilderException("Error resolving ResultSetType. Cause: " + e, e);
    }
}
```

#### 2.6 resolveParameterMode

`#resolveParameterMode(String alias)` 方法，解析对应的 ParameterMode 类型。代码如下：

```
// BaseBuilder.java

protected ParameterMode resolveParameterMode(String alias) {
    if (alias == null) {
        return null;
    }
    try {
        return ParameterMode.valueOf(alias);
    } catch (IllegalArgumentException e) {
        throw new BuilderException("Error resolving ParameterMode. Cause: " + e, e);
    }
}
```

#### 2.7 createInstance

`#createInstance(String alias)` 方法，创建指定对象。代码如下：

```
// BaseBuilder.java

protected Object createInstance(String alias) {
    // <1> 获得对应的类型
    Class<?> clazz = resolveClass(alias);
    if (clazz == null) {
        return null;
    }
    try {
        // <2> 创建对象
        return resolveClass(alias).newInstance(); // 这里重复获得了一次
    } catch (Exception e) {
        throw new BuilderException("Error creating instance. Cause: " + e, e);
    }
}
```

- `<1>` 处，调用 `#resolveClass(String alias)` 方法，获得对应的类型。代码如下：

  ```
  // BaseBuilder.java
  
  protected <T> Class<? extends T> resolveClass(String alias) {
      if (alias == null) {
          return null;
      }
      try {
          return resolveAlias(alias);
      } catch (Exception e) {
          throw new BuilderException("Error resolving class. Cause: " + e, e);
      }
  }
  
  protected <T> Class<? extends T> resolveAlias(String alias) {
      return typeAliasRegistry.resolveAlias(alias);
  }
  ```

  - 从 `typeAliasRegistry` 中，通过别名或类全名，获得对应的类。

- `<2>` 处，创建对象。

#### 2.8 resolveTypeHandler

`#resolveTypeHandler(Class<?> javaType, String typeHandlerAlias)` 方法，从 `typeHandlerRegistry` 中获得或创建对应的 TypeHandler 对象。代码如下：

```
// BaseBuilder.java

protected TypeHandler<?> resolveTypeHandler(Class<?> javaType, Class<? extends TypeHandler<?>> typeHandlerType) {
    if (typeHandlerType == null) {
        return null;
    }
    // javaType ignored for injected handlers see issue #746 for full detail
    // 先获得 TypeHandler 对象
    TypeHandler<?> handler = typeHandlerRegistry.getMappingTypeHandler(typeHandlerType);
    if (handler == null) { // 如果不存在，进行创建 TypeHandler 对象
        // not in registry, create a new one
        handler = typeHandlerRegistry.getInstance(javaType, typeHandlerType);
    }
    return handler;
}
```

## 3. XMLConfigBuilder

`org.apache.ibatis.builder.xml.XMLConfigBuilder` ，继承 BaseBuilder 抽象类，XML 配置构建器，主要负责解析 mybatis-config.xml 配置文件。即对应 [《MyBatis 文档 —— XML 映射配置文件》](http://www.mybatis.org/mybatis-3/zh/configuration.html) 。

#### 3.1 构造方法

```
// XMLConfigBuilder.java

/**
 * 是否已解析
 */
private boolean parsed;
/**
 * 基于 Java XPath 解析器
 */
private final XPathParser parser;
/**
 * 环境
 */
private String environment;
/**
 * ReflectorFactory 对象
 */
private final ReflectorFactory localReflectorFactory = new DefaultReflectorFactory();

public XMLConfigBuilder(Reader reader) {
    this(reader, null, null);
}

public XMLConfigBuilder(Reader reader, String environment) {
    this(reader, environment, null);
}

public XMLConfigBuilder(Reader reader, String environment, Properties props) {
    this(new XPathParser(reader, true, props, new XMLMapperEntityResolver()), environment, props);
}

public XMLConfigBuilder(InputStream inputStream) {
    this(inputStream, null, null);
}

public XMLConfigBuilder(InputStream inputStream, String environment) {
    this(inputStream, environment, null);
}

public XMLConfigBuilder(InputStream inputStream, String environment, Properties props) {
    this(new XPathParser(inputStream, true, props, new XMLMapperEntityResolver()), environment, props);
}

private XMLConfigBuilder(XPathParser parser, String environment, Properties props) {
    // <1> 创建 Configuration 对象
    super(new Configuration());
    ErrorContext.instance().resource("SQL Mapper Configuration");
    // <2> 设置 Configuration 的 variables 属性
    this.configuration.setVariables(props);
    this.parsed = false;
    this.environment = environment;
    this.parser = parser;
}
```

- `parser` 属性，XPathParser 对象。在 [《精尽 MyBatis 源码分析 —— 解析器模块》](http://svip.iocoder.cn/MyBatis/parsing-package) 中，已经详细解析。

- `localReflectorFactory` 属性，DefaultReflectorFactory 对象。在 [《精尽 MyBatis 源码分析 —— 反射模块》](http://svip.iocoder.cn/MyBatis/reflection-package) 中，已经详细解析。

- 构造方法重载了比较多，只需要看最后一个。

  - `<1>` 处，创建 Configuration 对象。

  - `<2>` 处，设置 Configuration 对象的 `variables` 属性。代码如下：

    ```
    // Configuration.java
    
    /**
     * 变量 Properties 对象。
     *
     * 参见 {@link org.apache.ibatis.builder.xml.XMLConfigBuilder#propertiesElement(XNode context)} 方法
     */
    protected Properties variables = new Properties();
    
    public void setVariables(Properties variables) {
        this.variables = variables;
    }
    ```

#### 3.2 parse

`#parse()` 方法，解析 XML 成 Configuration 对象。代码如下：

```
// XMLConfigBuilder.java

public Configuration parse() {
    // <1.1> 若已解析，抛出 BuilderException 异常
    if (parsed) {
        throw new BuilderException("Each XMLConfigBuilder can only be used once.");
    }
    // <1.2> 标记已解析
    parsed = true;
    // <2> 解析 XML configuration 节点
    parseConfiguration(parser.evalNode("/configuration"));
    return configuration;
}
```

- `<1.1>` 处，若已解析，抛出 BuilderException 异常。
- `<1.2>` 处，标记已解析。
- `<2>` 处，调用 `XPathParser#evalNode(String expression)` 方法，获得 XML `<configuration />` 节点，后调用 `#parseConfiguration(XNode root)` 方法，解析该节点。详细解析，见 [「3.3 parseConfiguration」](http://svip.iocoder.cn/MyBatis/builder-package-1/#) 。

#### 3.3 parseConfiguration

`#parseConfiguration(XNode root)` 方法，解析 `<configuration />` 节点。代码如下：

```
// XMLConfigBuilder.java

private void parseConfiguration(XNode root) {
    try {
        //issue #117 read properties first
        // <1> 解析 <properties /> 标签
        propertiesElement(root.evalNode("properties"));
        // <2> 解析 <settings /> 标签
        Properties settings = settingsAsProperties(root.evalNode("settings"));
        // <3> 加载自定义 VFS 实现类
        loadCustomVfs(settings);
        // <4> 解析 <typeAliases /> 标签
        typeAliasesElement(root.evalNode("typeAliases"));
        // <5> 解析 <plugins /> 标签
        pluginElement(root.evalNode("plugins"));
        // <6> 解析 <objectFactory /> 标签
        objectFactoryElement(root.evalNode("objectFactory"));
        // <7> 解析 <objectWrapperFactory /> 标签
        objectWrapperFactoryElement(root.evalNode("objectWrapperFactory"));
        // <8> 解析 <reflectorFactory /> 标签
        reflectorFactoryElement(root.evalNode("reflectorFactory"));
        // <9> 赋值 <settings /> 到 Configuration 属性
        settingsElement(settings);
        // read it after objectFactory and objectWrapperFactory issue #631
        // <10> 解析 <environments /> 标签
        environmentsElement(root.evalNode("environments"));
        // <11> 解析 <databaseIdProvider /> 标签
        databaseIdProviderElement(root.evalNode("databaseIdProvider"));
        // <12> 解析 <typeHandlers /> 标签
        typeHandlerElement(root.evalNode("typeHandlers"));
        // <13> 解析 <mappers /> 标签
        mapperElement(root.evalNode("mappers"));
    } catch (Exception e) {
        throw new BuilderException("Error parsing SQL Mapper Configuration. Cause: " + e, e);
    }
}
```

- `<1>` 处，调用 `#propertiesElement(XNode context)` 方法，解析 `<properties />` 节点。详细解析，见 [「3.3.1 propertiesElement」](http://svip.iocoder.cn/MyBatis/builder-package-1/#) 。
- `<2>` 处，调用 `#settingsAsProperties(XNode context)` 方法，解析 `<settings />` 节点。详细解析，见 [「3.3.2 settingsAsProperties」](http://svip.iocoder.cn/MyBatis/builder-package-1/#) 。
- `<3>` 处，调用 `#loadCustomVfs(Properties settings)` 方法，加载自定义 VFS 实现类。详细解析，见 [「3.3.3 loadCustomVfs」](http://svip.iocoder.cn/MyBatis/builder-package-1/#) 。
- `<4>` 处，调用 `#typeAliasesElement(XNode parent)` 方法，解析 `<typeAliases />` 节点。详细解析，见 [「3.3.4 typeAliasesElement」](http://svip.iocoder.cn/MyBatis/builder-package-1/#) 。
- `<5>` 处，调用 `#typeAliasesElement(XNode parent)` 方法，解析 `<plugins />` 节点。详细解析，见 [「3.3.5 pluginElement」](http://svip.iocoder.cn/MyBatis/builder-package-1/#) 。
- `<6>` 处，调用 `#objectFactoryElement(XNode parent)` 方法，解析 `<objectFactory />` 节点。详细解析，见 [「3.3.6 pluginElement」](http://svip.iocoder.cn/MyBatis/builder-package-1/#) 。
- `<7>` 处，调用 `#objectWrapperFactoryElement(XNode parent)` 方法，解析 `<objectWrapperFactory />` 节点。详细解析，见 [「3.3.7 objectWrapperFactoryElement」](http://svip.iocoder.cn/MyBatis/builder-package-1/#) 。
- `<8>` 处，调用 `#reflectorFactoryElement(XNode parent)` 方法，解析 `<reflectorFactory />` 节点。详细解析，见 [「3.3.8 reflectorFactoryElement」](http://svip.iocoder.cn/MyBatis/builder-package-1/#) 。
- `<9>` 处，调用 `#settingsElement(Properties props)` 方法，赋值 `<settings />` 到 Configuration 属性。详细解析，见 [「3.3.9 settingsElement」](http://svip.iocoder.cn/MyBatis/builder-package-1/#) 。
- `<10>` 处，调用 `#environmentsElement(XNode context)` 方法，解析 `<environments />` 标签。详细解析，见 [「3.3.10 environmentsElement」](http://svip.iocoder.cn/MyBatis/builder-package-1/#) 。
- `<11>` 处，调用 `#databaseIdProviderElement(XNode context)` 方法，解析 `<databaseIdProvider />` 标签。详细解析，见 [「3.3.11 databaseIdProviderElement」](http://svip.iocoder.cn/MyBatis/builder-package-1/#) 。
- `<12>` 处，调用 `#typeHandlerElement(XNode context)` 方法，解析 `<typeHandlers />` 标签。详细解析，见 [「3.3.12 typeHandlerElement」](http://svip.iocoder.cn/MyBatis/builder-package-1/#) 。
- `<13>` 处，调用 `#mapperElement(XNode context)` 方法，解析 `<mappers />` 标签。详细解析，见 [「3.3.13 mapperElement」](http://svip.iocoder.cn/MyBatis/builder-package-1/#) 。

###### 3.3.1 propertiesElement

`#propertiesElement(XNode context)` 方法，解析 `<properties />` 节点。大体逻辑如下：

1. 解析 `<properties />` 标签，成 Properties 对象。
2. 覆盖 `configuration` 中的 Properties 对象到上面的结果。
3. 设置结果到 `parser` 和 `configuration` 中。

代码如下：

```
// XMLConfigBuilder.java

private void propertiesElement(XNode context) throws Exception {
    if (context != null) {
        // 读取子标签们，为 Properties 对象
        Properties defaults = context.getChildrenAsProperties();
        // 读取 resource 和 url 属性
        String resource = context.getStringAttribute("resource");
        String url = context.getStringAttribute("url");
        if (resource != null && url != null) { // resource 和 url 都存在的情况下，抛出 BuilderException 异常
            throw new BuilderException("The properties element cannot specify both a URL and a resource based property file reference.  Please specify one or the other.");
        }
        // 读取本地 Properties 配置文件到 defaults 中。
        if (resource != null) {
            defaults.putAll(Resources.getResourceAsProperties(resource));
            // 读取远程 Properties 配置文件到 defaults 中。
        } else if (url != null) {
            defaults.putAll(Resources.getUrlAsProperties(url));
        }
        // 覆盖 configuration 中的 Properties 对象到 defaults 中。
        Properties vars = configuration.getVariables();
        if (vars != null) {
            defaults.putAll(vars);
        }
        // 设置 defaults 到 parser 和 configuration 中。
        parser.setVariables(defaults);
        configuration.setVariables(defaults);
    }
}
```

###### 3.3.2 settingsAsProperties

`#settingsElement(Properties props)` 方法，将 `<setting />` 标签解析为 Properties 对象。代码如下：

```
// XMLConfigBuilder.java

private Properties settingsAsProperties(XNode context) {
    // 将子标签，解析成 Properties 对象
    if (context == null) {
        return new Properties();
    }
    Properties props = context.getChildrenAsProperties();
    // Check that all settings are known to the configuration class
    // 校验每个属性，在 Configuration 中，有相应的 setting 方法，否则抛出 BuilderException 异常
    MetaClass metaConfig = MetaClass.forClass(Configuration.class, localReflectorFactory);
    for (Object key : props.keySet()) {
        if (!metaConfig.hasSetter(String.valueOf(key))) {
            throw new BuilderException("The setting " + key + " is not known.  Make sure you spelled it correctly (case sensitive).");
        }
    }
    return props;
}
```

###### 3.3.3 loadCustomVfs

`#loadCustomVfs(Properties settings)` 方法，加载自定义 VFS 实现类。代码如下：

```
// XMLConfigBuilder.java

private void loadCustomVfs(Properties props) throws ClassNotFoundException {
    // 获得 vfsImpl 属性
    String value = props.getProperty("vfsImpl");
    if (value != null) {
        // 使用 , 作为分隔符，拆成 VFS 类名的数组
        String[] clazzes = value.split(",");
        // 遍历 VFS 类名的数组
        for (String clazz : clazzes) {
            if (!clazz.isEmpty()) {
                // 获得 VFS 类
                @SuppressWarnings("unchecked")
                Class<? extends VFS> vfsImpl = (Class<? extends VFS>) Resources.classForName(clazz);
                // 设置到 Configuration 中
                configuration.setVfsImpl(vfsImpl);
            }
        }
    }
}

// Configuration.java

/**
 * VFS 实现类
 */
protected Class<? extends VFS> vfsImpl;

public void setVfsImpl(Class<? extends VFS> vfsImpl) {
    if (vfsImpl != null) {
        // 设置 vfsImpl 属性
        this.vfsImpl = vfsImpl;
        // 添加到 VFS 中的自定义 VFS 类的集合
        VFS.addImplClass(this.vfsImpl);
    }
}
```

###### 3.3.4 typeAliasesElement

`#typeAliasesElement(XNode parent)` 方法，解析 `<typeAliases />` 标签，将配置类注册到 `typeAliasRegistry` 中。代码如下：

```
// XMLConfigBuilder.java

private void typeAliasesElement(XNode parent) {
    if (parent != null) {
        // 遍历子节点
        for (XNode child : parent.getChildren()) {
            // 指定为包的情况下，注册包下的每个类
            if ("package".equals(child.getName())) {
                String typeAliasPackage = child.getStringAttribute("name");
                configuration.getTypeAliasRegistry().registerAliases(typeAliasPackage);
            // 指定为类的情况下，直接注册类和别名
            } else {
                String alias = child.getStringAttribute("alias");
                String type = child.getStringAttribute("type");
                try {
                    Class<?> clazz = Resources.classForName(type); // 获得类是否存在
                    // 注册到 typeAliasRegistry 中
                    if (alias == null) {
                        typeAliasRegistry.registerAlias(clazz);
                    } else {
                        typeAliasRegistry.registerAlias(alias, clazz);
                    }
                } catch (ClassNotFoundException e) { // 若类不存在，则抛出 BuilderException 异常
                    throw new BuilderException("Error registering typeAlias for '" + alias + "'. Cause: " + e, e);
                }
            }
        }
    }
}
```

###### 3.3.5 pluginElement

`#pluginElement(XNode parent)` 方法，解析 `<plugins />` 标签，添加到 `Configuration#interceptorChain` 中。代码如下：

```
// XMLConfigBuilder.java

private void pluginElement(XNode parent) throws Exception {
    if (parent != null) {
        // 遍历 <plugins /> 标签
        for (XNode child : parent.getChildren()) {
            String interceptor = child.getStringAttribute("interceptor");
            Properties properties = child.getChildrenAsProperties();
            // <1> 创建 Interceptor 对象，并设置属性
            Interceptor interceptorInstance = (Interceptor) resolveClass(interceptor).newInstance();
            interceptorInstance.setProperties(properties);
            // <2> 添加到 configuration 中
            configuration.addInterceptor(interceptorInstance);
        }
    }
}
```

- `<1>` 处，创建 Interceptor 对象，并设置属性。关于 Interceptor 类，后续文章，详细解析。

- `<2>` 处，调用 `Configuration#addInterceptor(Interceptor interceptor)` 方法，添加到 `configuration` 中。代码如下：

  ```
  // Configuration.java
  
  /**
   * 拦截器链
   */
  protected final InterceptorChain interceptorChain = new InterceptorChain();
  
  public void addInterceptor(Interceptor interceptor) {
      interceptorChain.addInterceptor(interceptor);
  }
  ```

  - 关于 InterceptorChain 类，后续文章，详细解析。

###### 3.3.6 objectFactoryElement

`#objectFactoryElement(XNode parent)` 方法，解析 `<objectFactory />` 节点。代码如下：

```
// XMLConfigBuilder.java

private void objectFactoryElement(XNode context) throws Exception {
    if (context != null) {
        // 获得 ObjectFactory 的实现类
        String type = context.getStringAttribute("type");
        // 获得 Properties 属性
        Properties properties = context.getChildrenAsProperties();
        // <1> 创建 ObjectFactory 对象，并设置 Properties 属性
        ObjectFactory factory = (ObjectFactory) resolveClass(type).newInstance();
        factory.setProperties(properties);
        // <2> 设置 Configuration 的 objectFactory 属性
        configuration.setObjectFactory(factory);
    }
}
```

- `<1>` 处，创建 ObjectFactory 对象，并设置 Properties 属性。

- `<2>` 处，调用 `Configuration#setObjectFactory(ObjectFactory objectFactory)` 方法，设置 Configuration 的 `objectFactory` 属性。代码如下：

  ```
  // Configuration.java
  
  /**
   * ObjectFactory 对象
   */
  protected ObjectFactory objectFactory = new DefaultObjectFactory();
  
  public void setObjectFactory(ObjectFactory objectFactory) {
      this.objectFactory = objectFactory;
  }
  ```

###### 3.3.7 objectWrapperFactoryElement

`#objectWrapperFactoryElement(XNode context)` 方法，解析 `<objectWrapperFactory />` 节点。代码如下：

```
// XMLConfigBuilder.java

private void objectWrapperFactoryElement(XNode context) throws Exception {
    if (context != null) {
        // 获得 ObjectFactory 的实现类
        String type = context.getStringAttribute("type");
        // <1> 创建 ObjectWrapperFactory 对象
        ObjectWrapperFactory factory = (ObjectWrapperFactory) resolveClass(type).newInstance();
        // 设置 Configuration 的 objectWrapperFactory 属性
        configuration.setObjectWrapperFactory(factory);
    }
}
```

- `<1>` 处，创建 ObjectWrapperFactory 对象。

- `<2>` 处，调用 `Configuration#setObjectWrapperFactory(ObjectWrapperFactory objectWrapperFactory)` 方法，设置 Configuration 的 `objectWrapperFactory` 属性。代码如下：

  ```
  // Configuration.java
  
  /**
   * ObjectWrapperFactory 对象
   */
  protected ObjectWrapperFactory objectWrapperFactory = new DefaultObjectWrapperFactory();
  
  public void setObjectWrapperFactory(ObjectWrapperFactory objectWrapperFactory) {
      this.objectWrapperFactory = objectWrapperFactory;
  }
  ```

###### 3.3.8 reflectorFactoryElement

`#reflectorFactoryElement(XNode parent)` 方法，解析 `<reflectorFactory />` 节点。代码如下：

```
// XMLConfigBuilder.java

private void reflectorFactoryElement(XNode context) throws Exception {
    if (context != null) {
        // 获得 ReflectorFactory 的实现类
        String type = context.getStringAttribute("type");
        // 创建 ReflectorFactory 对象
        ReflectorFactory factory = (ReflectorFactory) resolveClass(type).newInstance();
        // 设置 Configuration 的 reflectorFactory 属性
        configuration.setReflectorFactory(factory);
    }
}
```

- `<1>` 处，创建 ReflectorFactory 对象。

- `<2>` 处，调用 `Configuration#setReflectorFactory(ReflectorFactory reflectorFactory)` 方法，设置 Configuration 的 `reflectorFactory` 属性。代码如下：

  ```
  // Configuration.java
  
  /**
   * ReflectorFactory 对象
   */
  protected ReflectorFactory reflectorFactory = new DefaultReflectorFactory();
  
  public void setReflectorFactory(ReflectorFactory reflectorFactory) {
      this.reflectorFactory = reflectorFactory;
  }
  ```

###### 3.3.9 settingsElement

`#settingsElement(Properties props)` 方法，赋值 `<settings />` 到 Configuration 属性。代码如下：

```
// XMLConfigBuilder.java

private void settingsElement(Properties props) throws Exception {
    configuration.setAutoMappingBehavior(AutoMappingBehavior.valueOf(props.getProperty("autoMappingBehavior", "PARTIAL")));
    configuration.setAutoMappingUnknownColumnBehavior(AutoMappingUnknownColumnBehavior.valueOf(props.getProperty("autoMappingUnknownColumnBehavior", "NONE")));
    configuration.setCacheEnabled(booleanValueOf(props.getProperty("cacheEnabled"), true));
    configuration.setProxyFactory((ProxyFactory) createInstance(props.getProperty("proxyFactory")));
    configuration.setLazyLoadingEnabled(booleanValueOf(props.getProperty("lazyLoadingEnabled"), false));
    configuration.setAggressiveLazyLoading(booleanValueOf(props.getProperty("aggressiveLazyLoading"), false));
    configuration.setMultipleResultSetsEnabled(booleanValueOf(props.getProperty("multipleResultSetsEnabled"), true));
    configuration.setUseColumnLabel(booleanValueOf(props.getProperty("useColumnLabel"), true));
    configuration.setUseGeneratedKeys(booleanValueOf(props.getProperty("useGeneratedKeys"), false));
    configuration.setDefaultExecutorType(ExecutorType.valueOf(props.getProperty("defaultExecutorType", "SIMPLE")));
    configuration.setDefaultStatementTimeout(integerValueOf(props.getProperty("defaultStatementTimeout"), null));
    configuration.setDefaultFetchSize(integerValueOf(props.getProperty("defaultFetchSize"), null));
    configuration.setMapUnderscoreToCamelCase(booleanValueOf(props.getProperty("mapUnderscoreToCamelCase"), false));
    configuration.setSafeRowBoundsEnabled(booleanValueOf(props.getProperty("safeRowBoundsEnabled"), false));
    configuration.setLocalCacheScope(LocalCacheScope.valueOf(props.getProperty("localCacheScope", "SESSION")));
    configuration.setJdbcTypeForNull(JdbcType.valueOf(props.getProperty("jdbcTypeForNull", "OTHER")));
    configuration.setLazyLoadTriggerMethods(stringSetValueOf(props.getProperty("lazyLoadTriggerMethods"), "equals,clone,hashCode,toString"));
    configuration.setSafeResultHandlerEnabled(booleanValueOf(props.getProperty("safeResultHandlerEnabled"), true));
    configuration.setDefaultScriptingLanguage(resolveClass(props.getProperty("defaultScriptingLanguage")));
    @SuppressWarnings("unchecked")
    Class<? extends TypeHandler> typeHandler = resolveClass(props.getProperty("defaultEnumTypeHandler"));
    configuration.setDefaultEnumTypeHandler(typeHandler);
    configuration.setCallSettersOnNulls(booleanValueOf(props.getProperty("callSettersOnNulls"), false));
    configuration.setUseActualParamName(booleanValueOf(props.getProperty("useActualParamName"), true));
    configuration.setReturnInstanceForEmptyRow(booleanValueOf(props.getProperty("returnInstanceForEmptyRow"), false));
    configuration.setLogPrefix(props.getProperty("logPrefix"));
    @SuppressWarnings("unchecked")
    Class<? extends Log> logImpl = resolveClass(props.getProperty("logImpl"));
    configuration.setLogImpl(logImpl);
    configuration.setConfigurationFactory(resolveClass(props.getProperty("configurationFactory")));
}
```

- 属性比较多，瞟一眼就行。

###### 3.3.10 environmentsElement

`#environmentsElement(XNode context)` 方法，解析 `<environments />` 标签。代码如下：

```
// XMLConfigBuilder.java

private void environmentsElement(XNode context) throws Exception {
    if (context != null) {
        // <1> environment 属性非空，从 default 属性获得
        if (environment == null) {
            environment = context.getStringAttribute("default");
        }
        // 遍历 XNode 节点
        for (XNode child : context.getChildren()) {
            // <2> 判断 environment 是否匹配
            String id = child.getStringAttribute("id");
            if (isSpecifiedEnvironment(id)) {
                // <3> 解析 `<transactionManager />` 标签，返回 TransactionFactory 对象
                TransactionFactory txFactory = transactionManagerElement(child.evalNode("transactionManager"));
                // <4> 解析 `<dataSource />` 标签，返回 DataSourceFactory 对象
                DataSourceFactory dsFactory = dataSourceElement(child.evalNode("dataSource"));
                DataSource dataSource = dsFactory.getDataSource();
                // <5> 创建 Environment.Builder 对象
                Environment.Builder environmentBuilder = new Environment.Builder(id)
                        .transactionFactory(txFactory)
                        .dataSource(dataSource);
                // <6> 构造 Environment 对象，并设置到 configuration 中
                configuration.setEnvironment(environmentBuilder.build());
            }
        }
    }
}
```

- `<1>` 处，若 `environment` 属性非空，从 `default` 属性种获得 `environment` 属性。

- `<2>` 处，遍历 XNode 节点，获得其 `id` 属性，后调用 `#isSpecifiedEnvironment(String id)` 方法，判断 `environment` 和 `id` 是否匹配。代码如下：

  ```
  // XMLConfigBuilder.java
  
  private boolean isSpecifiedEnvironment(String id) {
      if (environment == null) {
          throw new BuilderException("No environment specified.");
      } else if (id == null) {
          throw new BuilderException("Environment requires an id attribute.");
      } else if (environment.equals(id)) { // 相等
          return true;
      }
      return false;
  }
  ```

- `<3>` 处，调用 `#transactionManagerElement(XNode context)` 方法，解析 `<transactionManager />` 标签，返回 TransactionFactory 对象。代码如下：

  ```
  // XMLConfigBuilder.java
  
  private TransactionFactory transactionManagerElement(XNode context) throws Exception {
      if (context != null) {
          // 获得 TransactionFactory 的类
          String type = context.getStringAttribute("type");
          // 获得 Properties 属性
          Properties props = context.getChildrenAsProperties();
          // 创建 TransactionFactory 对象，并设置属性
          TransactionFactory factory = (TransactionFactory) resolveClass(type).newInstance();
          factory.setProperties(props);
          return factory;
      }
      throw new BuilderException("Environment declaration requires a TransactionFactory.");
  }
  ```

- `<4>` 处，调用 `#dataSourceElement(XNode context)` 方法，解析 `<dataSource />` 标签，返回 DataSourceFactory 对象，而后返回 DataSource 对象。代码如下：

  ```
  // XMLConfigBuilder.java
  
  private DataSourceFactory dataSourceElement(XNode context) throws Exception {
      if (context != null) {
          // 获得 DataSourceFactory 的类
          String type = context.getStringAttribute("type");
          // 获得 Properties 属性
          Properties props = context.getChildrenAsProperties();
          // 创建 DataSourceFactory 对象，并设置属性
          DataSourceFactory factory = (DataSourceFactory) resolveClass(type).newInstance();
          factory.setProperties(props);
          return factory;
      }
      throw new BuilderException("Environment declaration requires a DataSourceFactory.");
  }
  ```

- `<5>` 处，创建 Environment.Builder 对象。

- `<6>` 处，构造 Environment 对象，并设置到 `configuration` 中。代码如下：

  ```
  // Configuration.java
  
  /**
   * DB Environment 对象
   */
  protected Environment environment;
  
  public void setEnvironment(Environment environment) {
      this.environment = environment;
  }
  ```

####### 3.3.10.1 Environment

`org.apache.ibatis.mapping.Environment` ，DB 环境。代码如下：

```
// Environment.java

public final class Environment {

    /**
     * 环境变好
     */
    private final String id;
    /**
     * TransactionFactory 对象
     */
    private final TransactionFactory transactionFactory;
    /**
     * DataSource 对象
     */
    private final DataSource dataSource;

    public Environment(String id, TransactionFactory transactionFactory, DataSource dataSource) {
        if (id == null) {
            throw new IllegalArgumentException("Parameter 'id' must not be null");
        }
        if (transactionFactory == null) {
            throw new IllegalArgumentException("Parameter 'transactionFactory' must not be null");
        }
        this.id = id;
        if (dataSource == null) {
            throw new IllegalArgumentException("Parameter 'dataSource' must not be null");
        }
        this.transactionFactory = transactionFactory;
        this.dataSource = dataSource;
    }

    /**
     * 构造器
     */
    public static class Builder {

        /**
         * 环境变好
         */
        private String id;
        /**
         * TransactionFactory 对象
         */
        private TransactionFactory transactionFactory;
        /**
         * DataSource 对象
         */
        private DataSource dataSource;

        public Builder(String id) {
            this.id = id;
        }

        public Builder transactionFactory(TransactionFactory transactionFactory) {
            this.transactionFactory = transactionFactory;
            return this;
        }

        public Builder dataSource(DataSource dataSource) {
            this.dataSource = dataSource;
            return this;
        }

        public String id() {
            return this.id;
        }

        public Environment build() {
            return new Environment(this.id, this.transactionFactory, this.dataSource);
        }

    }

    public String getId() {
        return this.id;
    }

    public TransactionFactory getTransactionFactory() {
        return this.transactionFactory;
    }

    public DataSource getDataSource() {
        return this.dataSource;
    }

}
```

###### 3.3.11 databaseIdProviderElement

`#databaseIdProviderElement(XNode context)` 方法，解析 `<databaseIdProvider />` 标签。代码如下：

```
// XMLConfigBuilder.java

private void databaseIdProviderElement(XNode context) throws Exception {
    DatabaseIdProvider databaseIdProvider = null;
    if (context != null) {
        // <1> 获得 DatabaseIdProvider 的类
        String type = context.getStringAttribute("type");
        // awful patch to keep backward compatibility 保持兼容
        if ("VENDOR".equals(type)) {
            type = "DB_VENDOR";
        }
        // <2> 获得 Properties 对象
        Properties properties = context.getChildrenAsProperties();
        // <3> 创建 DatabaseIdProvider 对象，并设置对应的属性
        databaseIdProvider = (DatabaseIdProvider) resolveClass(type).newInstance();
        databaseIdProvider.setProperties(properties);
    }
    Environment environment = configuration.getEnvironment();
    if (environment != null && databaseIdProvider != null) {
        // <4> 获得对应的 databaseId 编号
        String databaseId = databaseIdProvider.getDatabaseId(environment.getDataSource());
        // <5> 设置到 configuration 中
        configuration.setDatabaseId(databaseId);
    }
}
```

- 不了解的胖友，可以先看看 [《MyBatis 文档 —— XML 映射配置文件 —— databaseIdProvider 数据库厂商标识》](http://www.mybatis.org/mybatis-3/zh/configuration.html#databaseIdProvider) 。

- `<1>` 处，获得 DatabaseIdProvider 的类。

- `<2>` 处，获得 Properties 对象。

- `<3>` 处，创建 DatabaseIdProvider 对象，并设置对应的属性。

- `<4>` 处，调用 `DatabaseIdProvider#getDatabaseId(DataSource dataSource)` 方法，获得对应的 `databaseId` **标识**。

- `<5>` 处，设置到 `configuration` 中。代码如下：

  ```
  // Configuration.java
  
  /**
   * 数据库标识
   */
  protected String databaseId;
  
  public void setDatabaseId(String databaseId) {
      this.databaseId = databaseId;
  }
  ```

####### 3.3.11.1 DatabaseIdProvider

`org.apache.ibatis.mapping.DatabaseIdProvider` ，数据库标识提供器接口。代码如下：

```
public interface DatabaseIdProvider {

    /**
     * 设置属性
     *
     * @param p Properties 对象
     */
    void setProperties(Properties p);

    /**
     * 获得数据库标识
     *
     * @param dataSource 数据源
     * @return 数据库标识
     * @throws SQLException 当 DB 发生异常时
     */
    String getDatabaseId(DataSource dataSource) throws SQLException;

}
```

####### 3.3.11.2 VendorDatabaseIdProvider

`org.apache.ibatis.mapping.VendorDatabaseIdProvider` ，实现 DatabaseIdProvider 接口，供应商数据库标识提供器实现类。

**① 构造方法**

```
// VendorDatabaseIdProvider.java

/**
 * Properties 对象
 */
private Properties properties;

@Override
public String getDatabaseId(DataSource dataSource) {
    if (dataSource == null) {
        throw new NullPointerException("dataSource cannot be null");
    }
    try {
        return getDatabaseName(dataSource);
    } catch (Exception e) {
        log.error("Could not get a databaseId from dataSource", e);
    }
    return null;
}

@Override
public void setProperties(Properties p) {
    this.properties = p;
}
```

**② 获得数据库标识**

`#getDatabaseId(DataSource dataSource)` 方法，代码如下：

```
// VendorDatabaseIdProvider.java

@Override
public String getDatabaseId(DataSource dataSource) {
    if (dataSource == null) {
        throw new NullPointerException("dataSource cannot be null");
    }
    try {
        // 获得数据库标识
        return getDatabaseName(dataSource);
    } catch (Exception e) {
        log.error("Could not get a databaseId from dataSource", e);
    }
    return null;
}

private String getDatabaseName(DataSource dataSource) throws SQLException {
    // <1> 获得数据库产品名
    String productName = getDatabaseProductName(dataSource);
    if (this.properties != null) {
        for (Map.Entry<Object, Object> property : properties.entrySet()) {
            // 如果产品名包含 KEY ，则返回对应的  VALUE
            if (productName.contains((String) property.getKey())) {
                return (String) property.getValue();
            }
        }
        // no match, return null
        return null;
    }
    // <3> 不存在 properties ，则直接返回 productName
    return productName;
}
```

- `<1>` 处，调用 `#getDatabaseProductName(DataSource dataSource)` 方法，获得数据库产品名。代码如下：

  ```
  // VendorDatabaseIdProvider.java
  
  private String getDatabaseProductName(DataSource dataSource) throws SQLException {
      try (Connection con = dataSource.getConnection()) {
          // 获得数据库连接
          DatabaseMetaData metaData = con.getMetaData();
          // 获得数据库产品名
          return metaData.getDatabaseProductName();
      }
  }
  ```

  - 通过从 Connection 获得数据库产品名。

- `<2>` 处，如果 `properties` 非空，则从 `properties` 中匹配 `KEY` ？若成功，则返回 `VALUE` ，否则，返回 `null` 。

- `<3>` 处，如果 `properties` 为空，则直接返回 `productName` 。

###### 3.3.12 typeHandlerElement

`#typeHandlerElement(XNode parent)` 方法，解析 `<typeHandlers />` 标签。代码如下：

```
// XMLConfigBuilder.java

private void typeHandlerElement(XNode parent) throws Exception {
    if (parent != null) {
        // 遍历子节点
        for (XNode child : parent.getChildren()) {
            // <1> 如果是 package 标签，则扫描该包
            if ("package".equals(child.getName())) {
                String typeHandlerPackage = child.getStringAttribute("name");
                typeHandlerRegistry.register(typeHandlerPackage);
            // <2> 如果是 typeHandler 标签，则注册该 typeHandler 信息
            } else {
                // 获得 javaType、jdbcType、handler
                String javaTypeName = child.getStringAttribute("javaType");
                String jdbcTypeName = child.getStringAttribute("jdbcType");
                String handlerTypeName = child.getStringAttribute("handler");
                Class<?> javaTypeClass = resolveClass(javaTypeName);
                JdbcType jdbcType = resolveJdbcType(jdbcTypeName);
                Class<?> typeHandlerClass = resolveClass(handlerTypeName); // 非空
                // 注册 typeHandler
                if (javaTypeClass != null) {
                    if (jdbcType == null) {
                        typeHandlerRegistry.register(javaTypeClass, typeHandlerClass);
                    } else {
                        typeHandlerRegistry.register(javaTypeClass, jdbcType, typeHandlerClass);
                    }
                } else {
                    typeHandlerRegistry.register(typeHandlerClass);
                }
            }
        }
    }
}
```

- 遍历子节点，分别处理 `<1>` 是 `<package />` 和 `<2>` 是 `<typeHandler />` 两种标签的情况。逻辑比较简单，最终都是注册到 `typeHandlerRegistry` 中。

###### 3.3.13 mapperElement

`#mapperElement(XNode context)` 方法，解析 `<mappers />` 标签。代码如下：

```
// XMLConfigBuilder.java

private void mapperElement(XNode parent) throws Exception {
    if (parent != null) {
        // <0> 遍历子节点
        for (XNode child : parent.getChildren()) {
            // <1> 如果是 package 标签，则扫描该包
            if ("package".equals(child.getName())) {
                // 获得包名
                String mapperPackage = child.getStringAttribute("name");
                // 添加到 configuration 中
                configuration.addMappers(mapperPackage);
            // 如果是 mapper 标签，
            } else {
                // 获得 resource、url、class 属性
                String resource = child.getStringAttribute("resource");
                String url = child.getStringAttribute("url");
                String mapperClass = child.getStringAttribute("class");
                // <2> 使用相对于类路径的资源引用
                if (resource != null && url == null && mapperClass == null) {
                    ErrorContext.instance().resource(resource);
                    // 获得 resource 的 InputStream 对象
                    InputStream inputStream = Resources.getResourceAsStream(resource);
                    // 创建 XMLMapperBuilder 对象
                    XMLMapperBuilder mapperParser = new XMLMapperBuilder(inputStream, configuration, resource, configuration.getSqlFragments());
                    // 执行解析
                    mapperParser.parse();
                // <3> 使用完全限定资源定位符（URL）
                } else if (resource == null && url != null && mapperClass == null) {
                    ErrorContext.instance().resource(url);
                    // 获得 url 的 InputStream 对象
                    InputStream inputStream = Resources.getUrlAsStream(url);
                    // 创建 XMLMapperBuilder 对象
                    XMLMapperBuilder mapperParser = new XMLMapperBuilder(inputStream, configuration, url, configuration.getSqlFragments());
                    // 执行解析
                    mapperParser.parse();
                // <4> 使用映射器接口实现类的完全限定类名
                } else if (resource == null && url == null && mapperClass != null) {
                    // 获得 Mapper 接口
                    Class<?> mapperInterface = Resources.classForName(mapperClass);
                    // 添加到 configuration 中
                    configuration.addMapper(mapperInterface);
                } else {
                    throw new BuilderException("A mapper element may only specify a url, resource or class, but not more than one.");
                }
            }
        }
    }
}
```

- `<0>` 处，遍历子节点，处理每一个节点。根据节点情况，会分成 `<1>`、`<2>`、`<3>`、`<4>` 种情况，并且第一个是处理 `<package />` 标签，后三个是处理 `<mapper />` 标签。

- `<1>` 处，如果是 `<package />` 标签，则获得 `name` 报名，并调用 `Configuration#addMappers(String packageName)` 方法，扫描该包下的所有 Mapper 接口。代码如下：

  ```
  // Configuration.java
  
  /**
   * MapperRegistry 对象
   */
  protected final MapperRegistry mapperRegistry = new MapperRegistry(this);
  
  public void addMappers(String packageName) {
      // 扫描该包下所有的 Mapper 接口，并添加到 mapperRegistry 中
      mapperRegistry.addMappers(packageName);
  }
  ```

- `<4>` 处，如果是 `mapperClass` 非空，则是使用映射器接口实现类的完全限定类名，则获得 Mapper 接口，并调用 `Configuration#addMapper(Class<T> type)` 方法，直接添加到 `configuration` 中。代码如下：

  ```
  // Configuration.java
  
  public <T> void addMapper(Class<T> type) {
      mapperRegistry.addMapper(type);
  }
  ```

  - 实际上，`<1>` 和 `<4>` 是相似情况，差别在于前者需要扫描，才能获取到所有的 Mapper 接口，而后者明确知道是哪个 Mapper 接口。

- `<2>` 处，如果是 `resource` 非空，则是使用相对于类路径的资源引用，则需要创建 XMLMapperBuilder 对象，并调用 `XMLMapperBuilder#parse()` 方法，执行解析 Mapper XML 配置。执行之后，我们就能知道这个 Mapper XML 配置对应的 Mapper 接口。关于 XMLMapperBuilder 类，我们放在下一篇博客中，详细解析。

- `<3>` 处，如果是 `url` 非空，则是使用完全限定资源定位符（URL），情况和 `<2>` 是类似的。

# 2、加载 Mapper 映射配置文件

本文接 [《精尽 MyBatis 源码分析 —— MyBatis 初始化（一）之加载 mybatis-config》](http://svip.iocoder.cn/MyBatis/builder-package-1) 一文，来分享 MyBatis 初始化的第二步，**加载 Mapper 映射配置文件**。而这个步骤的入口是 XMLMapperBuilder 。下面，我们一起来看看它的代码实现。

> FROM [《Mybatis3.3.x技术内幕（八）：Mybatis初始化流程（上）》](https://my.oschina.net/zudajun/blog/668738)
>
> [![解析](http://static.iocoder.cn/images/MyBatis/2020_02_13/01.png)](http://static.iocoder.cn/images/MyBatis/2020_02_13/01.png)解析

- 上图，就是 Mapper 映射配置文件的解析结果。

## 2. XMLMapperBuilder

`org.apache.ibatis.builder.xml.XMLMapperBuilder` ，继承 BaseBuilder 抽象类，Mapper XML 配置构建器，主要负责解析 Mapper 映射配置文件。

#### 2.1 构造方法

```
// XMLMapperBuilder.java

/**
 * 基于 Java XPath 解析器
 */
private final XPathParser parser;
/**
 * Mapper 构造器助手
 */
private final MapperBuilderAssistant builderAssistant;
/**
 * 可被其他语句引用的可重用语句块的集合
 *
 * 例如：<sql id="userColumns"> ${alias}.id,${alias}.username,${alias}.password </sql>
 */
private final Map<String, XNode> sqlFragments;
/**
 * 资源引用的地址
 */
private final String resource;

public XMLMapperBuilder(InputStream inputStream, Configuration configuration, String resource, Map<String, XNode> sqlFragments, String namespace) {
    this(inputStream, configuration, resource, sqlFragments);
    this.builderAssistant.setCurrentNamespace(namespace);
}

public XMLMapperBuilder(InputStream inputStream, Configuration configuration, String resource, Map<String, XNode> sqlFragments) {
    this(new XPathParser(inputStream, true, configuration.getVariables(), new XMLMapperEntityResolver()),
            configuration, resource, sqlFragments);
}

private XMLMapperBuilder(XPathParser parser, Configuration configuration, String resource, Map<String, XNode> sqlFragments) {
    super(configuration);
    // 创建 MapperBuilderAssistant 对象
    this.builderAssistant = new MapperBuilderAssistant(configuration, resource);
    this.parser = parser;
    this.sqlFragments = sqlFragments;
    this.resource = resource;
}
```

- `builderAssistant` 属性，MapperBuilderAssistant 对象，是 XMLMapperBuilder 和 MapperAnnotationBuilder 的小助手，提供了一些公用的方法，例如创建 ParameterMap、MappedStatement 对象等等。关于 MapperBuilderAssistant 类，可见 [「3. MapperBuilderAssistant」](http://svip.iocoder.cn/MyBatis/builder-package-2/#) 。

#### 2.2 parse

`#parse()` 方法，解析 Mapper XML 配置文件。代码如下：

```
// XMLMapperBuilder.java

public void parse() {
    // <1> 判断当前 Mapper 是否已经加载过
    if (!configuration.isResourceLoaded(resource)) {
        // <2> 解析 `<mapper />` 节点
        configurationElement(parser.evalNode("/mapper"));
        // <3> 标记该 Mapper 已经加载过
        configuration.addLoadedResource(resource);
        // <4> 绑定 Mapper
        bindMapperForNamespace();
    }

    // <5> 解析待定的 <resultMap /> 节点
    parsePendingResultMaps();
    // <6> 解析待定的 <cache-ref /> 节点
    parsePendingCacheRefs();
    // <7> 解析待定的 SQL 语句的节点
    parsePendingStatements();
}
```

- `<1>` 处，调用 `Configuration#isResourceLoaded(String resource)` 方法，判断当前 Mapper 是否已经加载过。代码如下：

  ```
  // Configuration.java
  
  /**
   * 已加载资源( Resource )集合
   */
  protected final Set<String> loadedResources = new HashSet<>();
  
  public boolean isResourceLoaded(String resource) {
      return loadedResources.contains(resource);
  }
  ```

- `<3>` 处，调用 `Configuration#addLoadedResource(String resource)` 方法，标记该 Mapper 已经加载过。代码如下：

  ```
  // Configuration.java
  
  public void addLoadedResource(String resource) {
      loadedResources.add(resource);
  }
  ```

- `<2>` 处，调用 `#configurationElement(XNode context)` 方法，解析 `<mapper />` 节点。详细解析，见 [「2.3 configurationElement」](http://svip.iocoder.cn/MyBatis/builder-package-2/#) 。

- `<4>` 处，调用 `#bindMapperForNamespace()` 方法，绑定 Mapper 。详细解析，

- `<5>`、`<6>`、`<7>` 处，解析对应的**待定**的节点。详细解析，见 [「2.5 parsePendingXXX」](http://svip.iocoder.cn/MyBatis/builder-package-2/#) 。

#### 2.3 configurationElement

`#configurationElement(XNode context)` 方法，解析 `<mapper />` 节点。代码如下：

```
// XMLMapperBuilder.java

private void configurationElement(XNode context) {
    try {
        // <1> 获得 namespace 属性
        String namespace = context.getStringAttribute("namespace");
        if (namespace == null || namespace.equals("")) {
            throw new BuilderException("Mapper's namespace cannot be empty");
        }
        // <1> 设置 namespace 属性
        builderAssistant.setCurrentNamespace(namespace);
        // <2> 解析 <cache-ref /> 节点
        cacheRefElement(context.evalNode("cache-ref"));
        // <3> 解析 <cache /> 节点
        cacheElement(context.evalNode("cache"));
        // 已废弃！老式风格的参数映射。内联参数是首选,这个元素可能在将来被移除，这里不会记录。
        parameterMapElement(context.evalNodes("/mapper/parameterMap"));
        // <4> 解析 <resultMap /> 节点们
        resultMapElements(context.evalNodes("/mapper/resultMap"));
        // <5> 解析 <sql /> 节点们
        sqlElement(context.evalNodes("/mapper/sql"));
        // <6> 解析 <select /> <insert /> <update /> <delete /> 节点们
        buildStatementFromContext(context.evalNodes("select|insert|update|delete"));
    } catch (Exception e) {
        throw new BuilderException("Error parsing Mapper XML. The XML location is '" + resource + "'. Cause: " + e, e);
    }
}
```

- `<1>` 处，获得 `namespace` 属性，并设置到 MapperAnnotationBuilder 中。
- `<2>` 处，调用 `#cacheRefElement(XNode context)` 方法，解析 `<cache-ref />` 节点。详细解析，见 [「2.3.1 cacheElement」](http://svip.iocoder.cn/MyBatis/builder-package-2/#) 。
- `<3>` 处，调用 `#cacheElement(XNode context)` 方法，解析 `cache />` 标签。详细解析，见 [「2.3.2 cacheElement」](http://svip.iocoder.cn/MyBatis/builder-package-2/#) 。
- `<4>` 处，调用 `#resultMapElements(List<XNode> list)` 方法，解析 `<resultMap />` 节点们。详细解析，见 [「2.3.3 resultMapElements」](http://svip.iocoder.cn/MyBatis/builder-package-2/#) 。
- `<5>` 处，调用 `#sqlElement(List<XNode> list)` 方法，解析 `<sql />` 节点们。详细解析，见 [「2.3.4 sqlElement」](http://svip.iocoder.cn/MyBatis/builder-package-2/#) 。
- `<6>` 处，调用 `#buildStatementFromContext(List<XNode> list)` 方法，解析 `<select />`、`<insert />`、`<update />`、`<delete />` 节点们。详细解析，见 [「2.3.5 buildStatementFromContext」](http://svip.iocoder.cn/MyBatis/builder-package-2/#) 。

###### 2.3.1 cacheElement

`#cacheRefElement(XNode context)` 方法，解析 `<cache-ref />` 节点。代码如下：

```
// XMLMapperBuilder.java

private void cacheRefElement(XNode context) {
    if (context != null) {
        // <1> 获得指向的 namespace 名字，并添加到 configuration 的 cacheRefMap 中
        configuration.addCacheRef(builderAssistant.getCurrentNamespace(), context.getStringAttribute("namespace"));
        // <2> 创建 CacheRefResolver 对象，并执行解析
        CacheRefResolver cacheRefResolver = new CacheRefResolver(builderAssistant, context.getStringAttribute("namespace"));
        try {
            cacheRefResolver.resolveCacheRef();
        } catch (IncompleteElementException e) {
            // <3> 解析失败，添加到 configuration 的 incompleteCacheRefs 中
            configuration.addIncompleteCacheRef(cacheRefResolver);
        }
    }
}
```

- 示例如下：

  ```
  <cache-ref namespace="com.someone.application.data.SomeMapper"/>
  ```

- `<1>` 处，获得指向的 `namespace` 名字，并调用 `Configuration#addCacheRef(String namespace, String referencedNamespace)` 方法，添加到 `configuration` 的 `cacheRefMap` 中。代码如下：

  ```
  // Configuration.java
  
  /**
   * A map holds cache-ref relationship. The key is the namespace that
   * references a cache bound to another namespace and the value is the
   * namespace which the actual cache is bound to.
   *
   * Cache 指向的映射
   *
   * @see #addCacheRef(String, String)
   * @see org.apache.ibatis.builder.xml.XMLMapperBuilder#cacheRefElement(XNode)
   */
  protected final Map<String, String> cacheRefMap = new HashMap<>();
  
  public void addCacheRef(String namespace, String referencedNamespace) {
      cacheRefMap.put(namespace, referencedNamespace);
  }
  ```

- `<2>` 处，创建 CacheRefResolver 对象，并调用 `CacheRefResolver#resolveCacheRef()` 方法，执行解析。关于 CacheRefResolver ，在 [「2.3.1.1 CacheRefResolver」](http://svip.iocoder.cn/MyBatis/builder-package-2/#) 详细解析。

- `<3>` 处，解析失败，因为此处指向的 Cache 对象可能未初始化，则先调用 `Configuration#addIncompleteCacheRef(CacheRefResolver incompleteCacheRef)` 方法，添加到 `configuration` 的 `incompleteCacheRefs` 中。代码如下：

  ```
  // Configuration.java
  
  /**
   * CacheRefResolver 集合
   */
  protected final Collection<CacheRefResolver> incompleteCacheRefs = new LinkedList<>();
  
  public void addIncompleteCacheRef(CacheRefResolver incompleteCacheRef) {
      incompleteCacheRefs.add(incompleteCacheRef);
  }
  ```

####### 2.3.1.1 CacheRefResolver

`org.apache.ibatis.builder.CacheRefResolver` ，Cache 指向解析器。代码如下：

```
// CacheRefResolver.java

public class CacheRefResolver {

    private final MapperBuilderAssistant assistant;
    /**
     * Cache 指向的命名空间
     */
    private final String cacheRefNamespace;

    public CacheRefResolver(MapperBuilderAssistant assistant, String cacheRefNamespace) {
        this.assistant = assistant;
        this.cacheRefNamespace = cacheRefNamespace;
    }

    public Cache resolveCacheRef() {
        return assistant.useCacheRef(cacheRefNamespace);
    }

}
```

- 在 `#resolveCacheRef()` 方法中，会调用 `MapperBuilderAssistant#useCacheRef(String namespace)` 方法，获得指向的 Cache 对象。详细解析，见 [「3.3 useCacheRef」](http://svip.iocoder.cn/MyBatis/builder-package-2/#) 。

###### 2.3.2 cacheElement

`#cacheElement(XNode context)` 方法，解析 `cache />` 标签。代码如下：

```
// XMLMapperBuilder.java

private void cacheElement(XNode context) throws Exception {
    if (context != null) {
        // <1> 获得负责存储的 Cache 实现类
        String type = context.getStringAttribute("type", "PERPETUAL");
        Class<? extends Cache> typeClass = typeAliasRegistry.resolveAlias(type);
        // <2> 获得负责过期的 Cache 实现类
        String eviction = context.getStringAttribute("eviction", "LRU");
        Class<? extends Cache> evictionClass = typeAliasRegistry.resolveAlias(eviction);
        // <3> 获得 flushInterval、size、readWrite、blocking 属性
        Long flushInterval = context.getLongAttribute("flushInterval");
        Integer size = context.getIntAttribute("size");
        boolean readWrite = !context.getBooleanAttribute("readOnly", false);
        boolean blocking = context.getBooleanAttribute("blocking", false);
        // <4> 获得 Properties 属性
        Properties props = context.getChildrenAsProperties();
        // <5> 创建 Cache 对象
        builderAssistant.useNewCache(typeClass, evictionClass, flushInterval, size, readWrite, blocking, props);
    }
}
```

- 示例如下：

  ```
  // 使用默认缓存
  <cache eviction="FIFO" flushInterval="60000"  size="512" readOnly="true"/>
  
  // 使用自定义缓存
  <cache type="com.domain.something.MyCustomCache">
    <property name="cacheFile" value="/tmp/my-custom-cache.tmp"/>
  </cache>
  ```

- `<1>`、`<2>`、`<3>`、`<4>` 处，见代码注释即可。

- `<5>` 处，调用 `MapperBuilderAssistant#useNewCache(...)` 方法，创建 Cache 对象。详细解析，见 [「3.4 useNewCache」](http://svip.iocoder.cn/MyBatis/builder-package-2/#) 中。

###### 2.3.3 resultMapElements

> 老艿艿：开始高能，保持耐心。

整体流程如下：

> FROM [《Mybatis3.3.x技术内幕（十）：Mybatis初始化流程（下）》](https://my.oschina.net/zudajun/blog/669868)
>
> [![解析``](http://static.iocoder.cn/images/MyBatis/2020_02_13/03.png)](http://static.iocoder.cn/images/MyBatis/2020_02_13/03.png)解析``

`#resultMapElements(List<XNode> list)` 方法，解析 `<resultMap />` 节点们。代码如下：

```
// XMLMapperBuilder.java

// 解析 <resultMap /> 节点们
private void resultMapElements(List<XNode> list) throws Exception {
    // 遍历 <resultMap /> 节点们
    for (XNode resultMapNode : list) {
        try {
            // 处理单个 <resultMap /> 节点
            resultMapElement(resultMapNode);
        } catch (IncompleteElementException e) {
            // ignore, it will be retried
        }
    }
}

// 解析 <resultMap /> 节点
private ResultMap resultMapElement(XNode resultMapNode) throws Exception {
    return resultMapElement(resultMapNode, Collections.<ResultMapping>emptyList());
}

// 解析 <resultMap /> 节点
private ResultMap resultMapElement(XNode resultMapNode, List<ResultMapping> additionalResultMappings) throws Exception {
    ErrorContext.instance().activity("processing " + resultMapNode.getValueBasedIdentifier());
    // <1> 获得 id 属性
    String id = resultMapNode.getStringAttribute("id",
            resultMapNode.getValueBasedIdentifier());
    // <1> 获得 type 属性
    String type = resultMapNode.getStringAttribute("type",
            resultMapNode.getStringAttribute("ofType",
                    resultMapNode.getStringAttribute("resultType",
                            resultMapNode.getStringAttribute("javaType"))));
    // <1> 获得 extends 属性
    String extend = resultMapNode.getStringAttribute("extends");
    // <1> 获得 autoMapping 属性
    Boolean autoMapping = resultMapNode.getBooleanAttribute("autoMapping");
    // <1> 解析 type 对应的类
    Class<?> typeClass = resolveClass(type);
    Discriminator discriminator = null;
    // <2> 创建 ResultMapping 集合
    List<ResultMapping> resultMappings = new ArrayList<>();
    resultMappings.addAll(additionalResultMappings);
    // <2> 遍历 <resultMap /> 的子节点
    List<XNode> resultChildren = resultMapNode.getChildren();
    for (XNode resultChild : resultChildren) {
        // <2.1> 处理 <constructor /> 节点
        if ("constructor".equals(resultChild.getName())) {
            processConstructorElement(resultChild, typeClass, resultMappings);
        // <2.2> 处理 <discriminator /> 节点
        } else if ("discriminator".equals(resultChild.getName())) {
            discriminator = processDiscriminatorElement(resultChild, typeClass, resultMappings);
        // <2.3> 处理其它节点
        } else {
            List<ResultFlag> flags = new ArrayList<>();
            if ("id".equals(resultChild.getName())) {
                flags.add(ResultFlag.ID);
            }
            resultMappings.add(buildResultMappingFromContext(resultChild, typeClass, flags));
        }
    }
    // <3> 创建 ResultMapResolver 对象，执行解析
    ResultMapResolver resultMapResolver = new ResultMapResolver(builderAssistant, id, typeClass, extend, discriminator, resultMappings, autoMapping);
    try {
        return resultMapResolver.resolve();
    } catch (IncompleteElementException e) {
        // <4> 解析失败，添加到 configuration 中
        configuration.addIncompleteResultMap(resultMapResolver);
        throw e;
    }
}
```

- `<resultMap />` 标签的解析，是相对复杂的过程，情况比较多，所以胖友碰到不懂的，可以看看 [《MyBatis 文档 —— Mapper XML 文件》](http://www.mybatis.org/mybatis-3/zh/sqlmap-xml.html) 文档。

- `<1>` 处，获得 `id`、`type`、`extends`、`autoMapping` 属性，并解析 `type` 对应的类型。

- ```
  <2>
  ```

   

  处，创建 ResultMapping 集合，后遍历

   

  ```
  <resultMap />
  ```

   

  的

  子

  节点们，将每一个子节点解析成一个或多个 ResultMapping 对象，添加到集合中。即如下图所示：

  ![ResultMap 与 ResultMapping 的映射](http://static.iocoder.cn/images/MyBatis/2020_02_13/02.png)

  ResultMap 与 ResultMapping 的映射

  - `<2.1>` 处，调用 `#processConstructorElement(...)` 方法，处理 `<constructor />` 节点。详细解析，见 [「2.3.3.1 processConstructorElement」](http://svip.iocoder.cn/MyBatis/builder-package-2/#) 。
  - `<2.2>` 处，调用 `#processDiscriminatorElement(...)` 方法，处理 `<discriminator />` 节点。详细解析，见 [「2.3.3.2 processDiscriminatorElement」](http://svip.iocoder.cn/MyBatis/builder-package-2/#) 。
  - `<2.3>` 处，调用 `#buildResultMappingFromContext(XNode context, Class<?> resultType, List<ResultFlag> flags)` 方法，将当前子节点构建成 ResultMapping 对象，并添加到 `resultMappings` 中。详细解析，见 [「2.3.3.3 buildResultMappingFromContext」](http://svip.iocoder.cn/MyBatis/builder-package-2/#) 。🌞 这一块，和 [「2.3.3.1 processConstructorElement」](http://svip.iocoder.cn/MyBatis/builder-package-2/#) 的 `<3>` 是一致的。

- `<3>` 处，创建 ResultMapResolver 对象，执行解析。关于 ResultMapResolver ，在 「2.3.1.1 CacheRefResolver」 详细解析。

- `<4>` 处，如果解析失败，说明有依赖的信息不全，所以调用 `Configuration#addIncompleteResultMap(ResultMapResolver resultMapResolver)` 方法，添加到 Configuration 的 `incompleteResultMaps` 中。代码如下：

  ```
  // Configuration.java
  
  /**
   * ResultMapResolver 集合
   */
  protected final Collection<ResultMapResolver> incompleteResultMaps = new LinkedList<>();
  
  public void addIncompleteResultMap(ResultMapResolver resultMapResolver) {
      incompleteResultMaps.add(resultMapResolver);
  }
  ```

####### 2.3.3.1 processConstructorElement

`#processConstructorElement(XNode resultChild, Class<?> resultType, List<ResultMapping> resultMappings)` 方法，处理 `<constructor />` 节点。代码如下：

```
// XMLMapperBuilder.java

private void processConstructorElement(XNode resultChild, Class<?> resultType, List<ResultMapping> resultMappings) throws Exception {
    // <1> 遍历 <constructor /> 的子节点们
    List<XNode> argChildren = resultChild.getChildren();
    for (XNode argChild : argChildren) {
        // <2> 获得 ResultFlag 集合
        List<ResultFlag> flags = new ArrayList<>();
        flags.add(ResultFlag.CONSTRUCTOR);
        if ("idArg".equals(argChild.getName())) {
            flags.add(ResultFlag.ID);
        }
        // <3> 将当前子节点构建成 ResultMapping 对象，并添加到 resultMappings 中
        resultMappings.add(buildResultMappingFromContext(argChild, resultType, flags));
    }
}
```

- `<1>` 和 `<3>` 处，遍历 `<constructor />` 的子节点们，调用 `#buildResultMappingFromContext(XNode context, Class<?> resultType, List<ResultFlag> flags)` 方法，将当前子节点构建成 ResultMapping 对象，并添加到 `resultMappings` 中。详细解析，见 [「2.3.3.3 buildResultMappingFromContext」](http://svip.iocoder.cn/MyBatis/builder-package-2/#) 。

- `<2>` 处，我们可以看到一个 `org.apache.ibatis.mapping.ResultFlag` 枚举类，结果标识。代码如下：

  ```
  // ResultFlag.java
  
  public enum ResultFlag {
  
      /**
       * ID
       */
      ID,
      /**
       * 构造方法
       */
      CONSTRUCTOR
  
  }
  ```

  - 具体的用途，见下文。

####### 2.3.3.2 processDiscriminatorElement

`#processDiscriminatorElement(XNode context, Class<?> resultType, List<ResultMapping> resultMappings)` 方法，处理 `<constructor />` 节点。代码如下：

```
// XMLMapperBuilder.java

private Discriminator processDiscriminatorElement(XNode context, Class<?> resultType, List<ResultMapping> resultMappings) throws Exception {
    // <1> 解析各种属性
    String column = context.getStringAttribute("column");
    String javaType = context.getStringAttribute("javaType");
    String jdbcType = context.getStringAttribute("jdbcType");
    String typeHandler = context.getStringAttribute("typeHandler");
    // <1> 解析各种属性对应的类
    Class<?> javaTypeClass = resolveClass(javaType);
    Class<? extends TypeHandler<?>> typeHandlerClass = resolveClass(typeHandler);
    JdbcType jdbcTypeEnum = resolveJdbcType(jdbcType);
    // <2> 遍历 <discriminator /> 的子节点，解析成 discriminatorMap 集合
    Map<String, String> discriminatorMap = new HashMap<>();
    for (XNode caseChild : context.getChildren()) {
        String value = caseChild.getStringAttribute("value");
        String resultMap = caseChild.getStringAttribute("resultMap", processNestedResultMappings(caseChild, resultMappings)); // <2.1>
        discriminatorMap.put(value, resultMap);
    }
    // <3> 创建 Discriminator 对象
    return builderAssistant.buildDiscriminator(resultType, column, javaTypeClass, jdbcTypeEnum, typeHandlerClass, discriminatorMap);
}
```

- 可能大家对 `<discriminator />` 标签不是很熟悉，可以打开 [《MyBatis 文档 —— Mapper XML 文件》](http://www.mybatis.org/mybatis-3/zh/sqlmap-xml.html) 文档，然后下【鉴别器】。😈 当然，这块简单了解下就好，实际场景下，艿艿貌似都不知道它的存在，哈哈哈哈。

- `<1>` 处，解析各种属性以及属性对应的类。

- `<2>` 处，遍历 `<discriminator />` 的子节点，解析成 `discriminatorMap` 集合。

- `<2.1>` 处，如果是内嵌的 ResultMap 的情况，则调用 `#processNestedResultMappings(XNode context, List<ResultMapping> resultMappings)` 方法，处理**内嵌**的 ResultMap 的情况。代码如下：

  ```
  // XMLMapperBuilder.java
  
  private String processNestedResultMappings(XNode context, List<ResultMapping> resultMappings) throws Exception {
      if ("association".equals(context.getName())
              || "collection".equals(context.getName())
              || "case".equals(context.getName())) {
          if (context.getStringAttribute("select") == null) {
              // 解析，并返回 ResultMap
              ResultMap resultMap = resultMapElement(context, resultMappings);
              return resultMap.getId();
          }
      }
      return null;
  }
  ```

  - 该方法，会“递归”调用 `#resultMapElement(XNode context, List<ResultMapping> resultMappings)` 方法，处理内嵌的 ResultMap 的情况。也就是返回到 [「2.3.3 resultMapElement」](http://svip.iocoder.cn/MyBatis/builder-package-2/#) 流程。

- `<3>` 处，调用 `MapperBuilderAssistant#buildDiscriminator(...)` 方法，创建 Discriminator 对象。详细解析，见 [「3.6 buildDiscriminator」](http://svip.iocoder.cn/MyBatis/builder-package-2/#) 。

####### 2.3.3.3 buildResultMappingFromContext

`#buildResultMappingFromContext(XNode context, Class<?> resultType, List<ResultFlag> flags)` 方法，将当前节点构建成 ResultMapping 对象。代码如下：

```
// XMLMapperBuilder.java

private ResultMapping buildResultMappingFromContext(XNode context, Class<?> resultType, List<ResultFlag> flags) throws Exception {
    // <1> 获得各种属性
    String property;
    if (flags.contains(ResultFlag.CONSTRUCTOR)) {
        property = context.getStringAttribute("name");
    } else {
        property = context.getStringAttribute("property");
    }
    String column = context.getStringAttribute("column");
    String javaType = context.getStringAttribute("javaType");
    String jdbcType = context.getStringAttribute("jdbcType");
    String nestedSelect = context.getStringAttribute("select");
    String nestedResultMap = context.getStringAttribute("resultMap",
            processNestedResultMappings(context, Collections.emptyList()));
    String notNullColumn = context.getStringAttribute("notNullColumn");
    String columnPrefix = context.getStringAttribute("columnPrefix");
    String typeHandler = context.getStringAttribute("typeHandler");
    String resultSet = context.getStringAttribute("resultSet");
    String foreignColumn = context.getStringAttribute("foreignColumn");
    boolean lazy = "lazy".equals(context.getStringAttribute("fetchType", configuration.isLazyLoadingEnabled() ? "lazy" : "eager"));
    // <1> 获得各种属性对应的类
    Class<?> javaTypeClass = resolveClass(javaType);
    Class<? extends TypeHandler<?>> typeHandlerClass = resolveClass(typeHandler);
    JdbcType jdbcTypeEnum = resolveJdbcType(jdbcType);
    // <2> 构建 ResultMapping 对象
    return builderAssistant.buildResultMapping(resultType, property, column, javaTypeClass, jdbcTypeEnum, nestedSelect, nestedResultMap, notNullColumn, columnPrefix, typeHandlerClass, flags, resultSet, foreignColumn, lazy);
}
```

- `<1>` 处，解析各种属性以及属性对应的类。
- `<2>` 处，调用 `MapperBuilderAssistant#buildResultMapping(...)` 方法，构建 ResultMapping 对象。详细解析，见 [「3.5 buildResultMapping」](http://svip.iocoder.cn/MyBatis/builder-package-2/#) 。

####### 2.3.3.4 ResultMapResolver

`org.apache.ibatis.builder.ResultMapResolver`，ResultMap 解析器。代码如下：

```
// ResultMapResolver.java

public class ResultMapResolver {

    private final MapperBuilderAssistant assistant;
    /**
     * ResultMap 编号
     */
    private final String id;
    /**
     * 类型
     */
    private final Class<?> type;
    /**
     * 继承自哪个 ResultMap
     */
    private final String extend;
    /**
     * Discriminator 对象
     */
    private final Discriminator discriminator;
    /**
     * ResultMapping 集合
     */
    private final List<ResultMapping> resultMappings;
    /**
     * 是否自动匹配
     */
    private final Boolean autoMapping;

    public ResultMapResolver(MapperBuilderAssistant assistant, String id, Class<?> type, String extend, Discriminator discriminator, List<ResultMapping> resultMappings, Boolean autoMapping) {
        this.assistant = assistant;
        this.id = id;
        this.type = type;
        this.extend = extend;
        this.discriminator = discriminator;
        this.resultMappings = resultMappings;
        this.autoMapping = autoMapping;
    }

    public ResultMap resolve() {
        return assistant.addResultMap(this.id, this.type, this.extend, this.discriminator, this.resultMappings, this.autoMapping);
    }

}
```

- 在 `#resolve()` 方法中，会调用 `MapperBuilderAssistant#addResultMap(...)` 方法，创建 ResultMap 对象。详细解析，见 [「3.7 addResultMap」](http://svip.iocoder.cn/MyBatis/builder-package-2/#) 。

###### 2.3.4 sqlElement

`#sqlElement(List<XNode> list)` 方法，解析 `<sql />` 节点们。代码如下：

```
// XMLMapperBuilder.java

private void sqlElement(List<XNode> list) throws Exception {
    if (configuration.getDatabaseId() != null) {
        sqlElement(list, configuration.getDatabaseId());
    }
    sqlElement(list, null);
    // 上面两块代码，可以简写成 sqlElement(list, configuration.getDatabaseId());
}

private void sqlElement(List<XNode> list, String requiredDatabaseId) throws Exception {
    // <1> 遍历所有 <sql /> 节点
    for (XNode context : list) {
        // <2> 获得 databaseId 属性
        String databaseId = context.getStringAttribute("databaseId");
        // <3> 获得完整的 id 属性，格式为 `${namespace}.${id}` 。
        String id = context.getStringAttribute("id");
        id = builderAssistant.applyCurrentNamespace(id, false);
        // <4> 判断 databaseId 是否匹配
        if (databaseIdMatchesCurrent(id, databaseId, requiredDatabaseId)) {
            // <5> 添加到 sqlFragments 中
            sqlFragments.put(id, context);
        }
    }
}
```

- `<1>` 处，遍历所有 `<sql />` 节点，逐个处理。

- `<2>` 处，获得 `databaseId` 属性。

- `<3>` 处，获得完整的 `id` 属性，格式为 `${namespace}.${id}` 。

- `<4>` 处，调用 `#databaseIdMatchesCurrent(String id, String databaseId, String requiredDatabaseId)` 方法，判断 `databaseId` 是否匹配。代码如下：

  ```
  // XMLMapperBuilder.java
  
  private boolean databaseIdMatchesCurrent(String id, String databaseId, String requiredDatabaseId) {
      // 如果不匹配，则返回 false
      if (requiredDatabaseId != null) {
          return requiredDatabaseId.equals(databaseId);
      } else {
          // 如果未设置 requiredDatabaseId ，但是 databaseId 存在，说明还是不匹配，则返回 false
          // mmp ，写的好绕
          if (databaseId != null) {
              return false;
          }
          // skip this fragment if there is a previous one with a not null databaseId
          // 判断是否已经存在
          if (this.sqlFragments.containsKey(id)) {
              XNode context = this.sqlFragments.get(id);
              // 若存在，则判断原有的 sqlFragment 是否 databaseId 为空。因为，当前 databaseId 为空，这样两者才能匹配。
              return context.getStringAttribute("databaseId") == null;
          }
      }
      return true;
  }
  ```

- `<5>` 处，添加到 `sqlFragments` 中。因为 `sqlFragments` 是来自 Configuration 的 `sqlFragments` 属性，所以相当于也被添加了。代码如下：

  ```
  // Configuration.java
  
   /**
   * 可被其他语句引用的可重用语句块的集合
   *
   * 例如：<sql id="userColumns"> ${alias}.id,${alias}.username,${alias}.password </sql>
   */
  protected final Map<String, XNode> sqlFragments = new StrictMap<>("XML fragments parsed from previous mappers");
  ```

###### 2.3.5 buildStatementFromContext

`#buildStatementFromContext(List<XNode> list)` 方法，解析 `<select />`、`<insert />`、`<update />`、`<delete />` 节点们。代码如下：

```
// XMLMapperBuilder.java

private void buildStatementFromContext(List<XNode> list) {
    if (configuration.getDatabaseId() != null) {
        buildStatementFromContext(list, configuration.getDatabaseId());
    }
    buildStatementFromContext(list, null);
    // 上面两块代码，可以简写成 buildStatementFromContext(list, configuration.getDatabaseId());
}

private void buildStatementFromContext(List<XNode> list, String requiredDatabaseId) {
    // <1> 遍历 <select /> <insert /> <update /> <delete /> 节点们
    for (XNode context : list) {
        // <1> 创建 XMLStatementBuilder 对象，执行解析
        final XMLStatementBuilder statementParser = new XMLStatementBuilder(configuration, builderAssistant, context, requiredDatabaseId);
        try {
            statementParser.parseStatementNode();
        } catch (IncompleteElementException e) {
            // <2> 解析失败，添加到 configuration 中
            configuration.addIncompleteStatement(statementParser);
        }
    }
}
```

- `<1>` 处，遍历 `<select />`、`<insert />`、`<update />`、`<delete />` 节点们，逐个创建 XMLStatementBuilder 对象，执行解析。关于 XMLStatementBuilder 类，我们放在下篇文章，详细解析。

- `<2>` 处，解析失败，调用 `Configuration#addIncompleteStatement(XMLStatementBuilder incompleteStatement)` 方法，添加到 `configuration` 中。代码如下：

  ```
  // Configuration.java
  
  /**
   * XMLStatementBuilder 集合
   */
  protected final Collection<XMLStatementBuilder> incompleteStatements = new LinkedList<>();
  
  public void addIncompleteStatement(XMLStatementBuilder incompleteStatement) {
      incompleteStatements.add(incompleteStatement);
  }
  ```

#### 2.4 bindMapperForNamespace

`#bindMapperForNamespace()` 方法，绑定 Mapper 。代码如下：

```
// XMLMapperBuilder.java

private void bindMapperForNamespace() {
    String namespace = builderAssistant.getCurrentNamespace();
    if (namespace != null) {
        // <1> 获得 Mapper 映射配置文件对应的 Mapper 接口，实际上类名就是 namespace 。嘿嘿，这个是常识。
        Class<?> boundType = null;
        try {
            boundType = Resources.classForName(namespace);
        } catch (ClassNotFoundException e) {
            //ignore, bound type is not required
        }
        if (boundType != null) {
            // <2> 不存在该 Mapper 接口，则进行添加
            if (!configuration.hasMapper(boundType)) {
                // Spring may not know the real resource name so we set a flag
                // to prevent loading again this resource from the mapper interface
                // look at MapperAnnotationBuilder#loadXmlResource
                // <3> 标记 namespace 已经添加，避免 MapperAnnotationBuilder#loadXmlResource(...) 重复加载
                configuration.addLoadedResource("namespace:" + namespace);
                // <4> 添加到 configuration 中
                configuration.addMapper(boundType);
            }
        }
    }
}
```

- `<1>` 处，获得 Mapper 映射配置文件对应的 Mapper 接口，实际上类名就是 `namespace` 。嘿嘿，这个是常识。

- `<2>` 处，调用 `Configuration#hasMapper(Class<?> type)` 方法，判断若谷不存在该 Mapper 接口，则进行绑定。代码如下：

  ```
  // Configuration.java
  
  /**
   * MapperRegistry 对象
   */
  protected final MapperRegistry mapperRegistry = new MapperRegistry(this);
  
  public boolean hasMapper(Class<?> type) {
      return mapperRegistry.hasMapper(type);
  }
  ```

- `<3>` 处，调用 `Configuration#addLoadedResource(String resource)` 方法，标记 `namespace` 已经添加，避免 `MapperAnnotationBuilder#loadXmlResource(...)` 重复加载。代码如下：

  ```
  // MapperAnnotationBuilder.java
  
  private void loadXmlResource() {
      // Spring may not know the real resource name so we check a flag
      // to prevent loading again a resource twice
      // this flag is set at XMLMapperBuilder#bindMapperForNamespace
      if (!configuration.isResourceLoaded("namespace:" + type.getName())) {
          // ... 省略创建 XMLMapperBuilder ，进行解析的代码
      }
  }
  ```

- `<4>` 处，调用 `Configuration#addMapper(Class<T> type)` 方法，添加到 `configuration` 的 `mapperRegistry` 中。代码如下：

  ```
  // Configuration.java
  
  public <T> void addMapper(Class<T> type) {
      mapperRegistry.addMapper(type);
  }
  ```

#### 2.5 parsePendingXXX

有三个 parsePendingXXX 方法，代码如下：

```
// XMLMapperBuilder.java

private void parsePendingResultMaps() {
    // 获得 ResultMapResolver 集合，并遍历进行处理
    Collection<ResultMapResolver> incompleteResultMaps = configuration.getIncompleteResultMaps();
    synchronized (incompleteResultMaps) {
        Iterator<ResultMapResolver> iter = incompleteResultMaps.iterator();
        while (iter.hasNext()) {
            try {
                // 执行解析
                iter.next().resolve();
                // 移除
                iter.remove();
            } catch (IncompleteElementException e) {
                // ResultMap is still missing a resource...
                // 解析失败，不抛出异常
            }
        }
    }
}

private void parsePendingCacheRefs() {
    // 获得 CacheRefResolver 集合，并遍历进行处理
    Collection<CacheRefResolver> incompleteCacheRefs = configuration.getIncompleteCacheRefs();
    synchronized (incompleteCacheRefs) {
        Iterator<CacheRefResolver> iter = incompleteCacheRefs.iterator();
        while (iter.hasNext()) {
            try {
                // 执行解析
                iter.next().resolveCacheRef();
                // 移除
                iter.remove();
            } catch (IncompleteElementException e) {
                // Cache ref is still missing a resource...
            }
        }
    }
}

private void parsePendingStatements() {
    // 获得 XMLStatementBuilder 集合，并遍历进行处理
    Collection<XMLStatementBuilder> incompleteStatements = configuration.getIncompleteStatements();
    synchronized (incompleteStatements) {
        Iterator<XMLStatementBuilder> iter = incompleteStatements.iterator();
        while (iter.hasNext()) {
            try {
                // 执行解析
                iter.next().parseStatementNode();
                // 移除
                iter.remove();
            } catch (IncompleteElementException e) {
                // Statement is still missing a resource...
            }
        }
    }
}
```

- 三个方法的逻辑思路基本一致：1）获得对应的集合；2）遍历集合，执行解析；3）执行成功，则移除出集合；4）执行失败，忽略异常。
- 当然，实际上，此处还是可能有执行解析失败的情况，但是随着每一个 Mapper 配置文件对应的 XMLMapperBuilder 执行一次这些方法，逐步逐步就会被全部解析完。😈

## 3. MapperBuilderAssistant

`org.apache.ibatis.builder.MapperBuilderAssistant` ，继承 BaseBuilder 抽象类，Mapper 构造器的小助手，提供了一些公用的方法，例如创建 ParameterMap、MappedStatement 对象等等。

#### 3.1 构造方法

```
// MapperBuilderAssistant.java

/**
 * 当前 Mapper 命名空间
 */
private String currentNamespace;
/**
 * 资源引用的地址
 */
private final String resource;
/**
 * 当前 Cache 对象
 */
private Cache currentCache;
/**
 * 是否未解析成功 Cache 引用
 */
private boolean unresolvedCacheRef; // issue #676

public MapperBuilderAssistant(Configuration configuration, String resource) {
    super(configuration);
    ErrorContext.instance().resource(resource);
    this.resource = resource;
}
```

- 实际上，😈 如果要不是为了 XMLMapperBuilder 和 MapperAnnotationBuilder 都能调用到这个公用方法，可能都不需要这个类。

#### 3.2 setCurrentNamespace

`#setCurrentNamespace(String currentNamespace)` 方法，设置 `currentNamespace` 属性。代码如下：

```
// MapperBuilderAssistant.java

public void setCurrentNamespace(String currentNamespace) {
    // 如果传入的 currentNamespace 参数为空，抛出 BuilderException 异常
    if (currentNamespace == null) {
        throw new BuilderException("The mapper element requires a namespace attribute to be specified.");
    }

    // 如果当前已经设置，并且还和传入的不相等，抛出 BuilderException 异常
    if (this.currentNamespace != null && !this.currentNamespace.equals(currentNamespace)) {
        throw new BuilderException("Wrong namespace. Expected '"
                + this.currentNamespace + "' but found '" + currentNamespace + "'.");
    }

    // 设置
    this.currentNamespace = currentNamespace;
}
```

#### 3.3 useCacheRef

`#useCacheRef(String namespace)` 方法，获得指向的 Cache 对象。如果获得不到，则抛出 IncompleteElementException 异常。代码如下：

```
// MapperBuilderAssistant.java

public Cache useCacheRef(String namespace) {
    if (namespace == null) {
        throw new BuilderException("cache-ref element requires a namespace attribute.");
    }
    try {
        unresolvedCacheRef = true; // 标记未解决
        // <1> 获得 Cache 对象
        Cache cache = configuration.getCache(namespace);
        // 获得不到，抛出 IncompleteElementException 异常
        if (cache == null) {
            throw new IncompleteElementException("No cache for namespace '" + namespace + "' could be found.");
        }
        // 记录当前 Cache 对象
        currentCache = cache;
        unresolvedCacheRef = false; // 标记已解决
        return cache;
    } catch (IllegalArgumentException e) {
        throw new IncompleteElementException("No cache for namespace '" + namespace + "' could be found.", e);
    }
}
```

- `<1>` 处，调用 `Configuration#getCache(String id)` 方法，获得 Cache 对象。代码如下：

  ```
  // Configuration.java
  
  /**
   * Cache 对象集合
   *
   * KEY：命名空间 namespace
   */
  protected final Map<String, Cache> caches = new StrictMap<>("Caches collection");
  
  public Cache getCache(String id) {
      return caches.get(id);
  }
  ```

#### 3.4 useNewCache

`#useNewCache(Class<? extends Cache> typeClass, Class<? extends Cache> evictionClass, Long flushInterval, Integer size, boolean readWrite, boolean blocking, Properties props)` 方法，创建 Cache 对象。代码如下：

```
// MapperBuilderAssistant.java

public Cache useNewCache(Class<? extends Cache> typeClass,
                         Class<? extends Cache> evictionClass,
                         Long flushInterval,
                         Integer size,
                         boolean readWrite,
                         boolean blocking,
                         Properties props) {
    // <1> 创建 Cache 对象
    Cache cache = new CacheBuilder(currentNamespace)
            .implementation(valueOrDefault(typeClass, PerpetualCache.class))
            .addDecorator(valueOrDefault(evictionClass, LruCache.class))
            .clearInterval(flushInterval)
            .size(size)
            .readWrite(readWrite)
            .blocking(blocking)
            .properties(props)
            .build();
    // <2> 添加到 configuration 的 caches 中
    configuration.addCache(cache);
    // <3> 赋值给 currentCache
    currentCache = cache;
    return cache;
}
```

- `<1>` 处，创建 Cache 对象。关于 CacheBuilder 类，详细解析，见 [「3.4.1 CacheBuilder」](http://svip.iocoder.cn/MyBatis/builder-package-2/#) 。

- `<2>` 处，调用 `Configuration#addCache(Cache cache)` 方法，添加到 `configuration` 的 `caches` 中。代码如下：

  ```
  // Configuration.java
  
  public void addCache(Cache cache) {
      caches.put(cache.getId(), cache);
  }
  ```

- `<3>` 处，赋值给 `currentCache` 。

###### 3.4.1 CacheBuilder

`org.apache.ibatis.mapping.CacheBuilder` ，Cache 构造器。基于装饰者设计模式，进行 Cache 对象的构造。代码比较简单，但是有点略长，胖友直接点击 [链接](https://github.com/YunaiV/mybatis-3/blob/master/src/main/java/org/apache/ibatis/mapping/CacheBuilder.java) 查看，已经添加了完整的注释。

#### 3.5 buildResultMapping

`#buildResultMapping(Class<?> resultType, String property, String column,Class<?> javaType, JdbcType jdbcType, String nestedSelect, String nestedResultMap, String notNullColumn, String columnPrefix, Class<? extends TypeHandler<?>> typeHandler, List<ResultFlag> flags, String resultSet, String foreignColumn, boolean lazy)` 方法，构造 ResultMapping 对象。代码如下：

```
// MapperBuilderAssistant.java

public ResultMapping buildResultMapping(
        Class<?> resultType,
        String property,
        String column,
        Class<?> javaType,
        JdbcType jdbcType,
        String nestedSelect,
        String nestedResultMap,
        String notNullColumn,
        String columnPrefix,
        Class<? extends TypeHandler<?>> typeHandler,
        List<ResultFlag> flags,
        String resultSet,
        String foreignColumn,
        boolean lazy) {
    // <1> 解析对应的 Java Type 类和 TypeHandler 对象
    Class<?> javaTypeClass = resolveResultJavaType(resultType, property, javaType);
    TypeHandler<?> typeHandlerInstance = resolveTypeHandler(javaTypeClass, typeHandler);
    // <2> 解析组合字段名称成 ResultMapping 集合。涉及「关联的嵌套查询」
    List<ResultMapping> composites = parseCompositeColumnName(column);
    // <3> 创建 ResultMapping 对象
    return new ResultMapping.Builder(configuration, property, column, javaTypeClass)
            .jdbcType(jdbcType)
            .nestedQueryId(applyCurrentNamespace(nestedSelect, true)) // <3.1>
            .nestedResultMapId(applyCurrentNamespace(nestedResultMap, true)) // <3.1>
            .resultSet(resultSet)
            .typeHandler(typeHandlerInstance)
            .flags(flags == null ? new ArrayList<>() : flags)
            .composites(composites)
            .notNullColumns(parseMultipleColumnNames(notNullColumn)) // <3.2>
            .columnPrefix(columnPrefix)
            .foreignColumn(foreignColumn)
            .lazy(lazy)
}
```

- `<1>` 处，解析对应的 Java Type 类和 TypeHandler 对象。

- `<2>` 处，调用 `#parseCompositeColumnName(String columnName)` 方法，解析组合字段名称成 ResultMapping 集合。详细解析，见 [「3.5.1 parseCompositeColumnName」](http://svip.iocoder.cn/MyBatis/builder-package-2/#) 中。

- ```
  <3>
  ```

   

  处，创建 ResultMapping 对象。

  - `<3.1>` 处，调用 `#applyCurrentNamespace(String base, boolean isReference)` 方法，拼接命名空间。详细解析，见 [「3.5.2 applyCurrentNamespace」](http://svip.iocoder.cn/MyBatis/builder-package-2/#) 。
  - `<3.2>` 处，调用 `#parseMultipleColumnNames(String notNullColumn)` 方法，将字符串解析成集合。详细解析，见 [「3.5.3 parseMultipleColumnNames」](http://svip.iocoder.cn/MyBatis/builder-package-2/#) 。
  - 关于 ResultMapping 类，在 [「3.5.4 ResultMapping」](http://svip.iocoder.cn/MyBatis/builder-package-2/#) 中详细解析。

###### 3.5.1 parseCompositeColumnName

`#parseCompositeColumnName(String columnName)` 方法，解析组合字段名称成 ResultMapping 集合。代码如下：

```
// MapperBuilderAssistant.java

private List<ResultMapping> parseCompositeColumnName(String columnName) {
    List<ResultMapping> composites = new ArrayList<>();
    // 分词，解析其中的 property 和 column 的组合对
    if (columnName != null && (columnName.indexOf('=') > -1 || columnName.indexOf(',') > -1)) {
        StringTokenizer parser = new StringTokenizer(columnName, "{}=, ", false);
        while (parser.hasMoreTokens()) {
            String property = parser.nextToken();
            String column = parser.nextToken();
            // 创建 ResultMapping 对象
            ResultMapping complexResultMapping = new ResultMapping.Builder(
                    configuration, property, column, configuration.getTypeHandlerRegistry().getUnknownTypeHandler()).build();
            // 添加到 composites 中
            composites.add(complexResultMapping);
        }
    }
    return composites;
}
```

- 对于这种情况，官方文档说明如下：

  > FROM [《MyBatis 文档 —— Mapper XML 文件》](http://www.mybatis.org/mybatis-3/zh/sqlmap-xml.html) 的 「关联的嵌套查询」 小节
  >
  > 来自数据库的列名,或重命名的列标签。这和通常传递给 resultSet.getString(columnName)方法的字符串是相同的。 column 注 意 : 要 处 理 复 合 主 键 , 你 可 以 指 定 多 个 列 名 通 过 column= ” {prop1=col1,prop2=col2} ” 这种语法来传递给嵌套查询语 句。这会引起 prop1 和 prop2 以参数对象形式来设置给目标嵌套查询语句。

- 😈 不用理解太细，如果胖友和我一样，基本用不到这个特性。

###### 3.5.2 applyCurrentNamespace

`#applyCurrentNamespace(String base, boolean isReference)` 方法，拼接命名空间。代码如下：

```
// MapperBuilderAssistant.java

public String applyCurrentNamespace(String base, boolean isReference) {
    if (base == null) {
        return null;
    }
    if (isReference) {
        // is it qualified with any namespace yet?
        if (base.contains(".")) {
            return base;
        }
    } else {
        // is it qualified with this namespace yet?
        if (base.startsWith(currentNamespace + ".")) {
            return base;
        }
        if (base.contains(".")) {
            throw new BuilderException("Dots are not allowed in element names, please remove it from " + base);
        }
    }
    // 拼接 currentNamespace + base
    return currentNamespace + "." + base;
}
```

- 通过这样的方式，生成**唯一**在的标识。

###### 3.5.3 parseMultipleColumnNames

`#parseMultipleColumnNames(String notNullColumn)` 方法，将字符串解析成集合。代码如下：

```
// MapperBuilderAssistant.java

private Set<String> parseMultipleColumnNames(String columnName) {
    Set<String> columns = new HashSet<>();
    if (columnName != null) {
        // 多个字段，使用 ，分隔
        if (columnName.indexOf(',') > -1) {
            StringTokenizer parser = new StringTokenizer(columnName, "{}, ", false);
            while (parser.hasMoreTokens()) {
                String column = parser.nextToken();
                columns.add(column);
            }
        } else {
            columns.add(columnName);
        }
    }
    return columns;
}
```

###### 3.5.4 ResultMapping

`org.apache.ibatis.mapping.ResultMapping` ，ResultMap 中的每一条结果字段的映射。代码比较简单，但是有点略长，胖友直接点击 [链接](https://github.com/YunaiV/mybatis-3/blob/master/src/main/java/org/apache/ibatis/mapping/ResultMapping.java) 查看，已经添加了完整的注释。

#### 3.6 buildDiscriminator

`#buildDiscriminator(Class<?> resultType, String column, Class<?> javaType, JdbcType jdbcType, Class<? extends TypeHandler<?>> typeHandler, Map<String, String> discriminatorMap)` 方法，构建 Discriminator 对象。代码如下：

```
// MapperBuilderAssistant.java

public Discriminator buildDiscriminator(
        Class<?> resultType,
        String column,
        Class<?> javaType,
        JdbcType jdbcType,
        Class<? extends TypeHandler<?>> typeHandler,
        Map<String, String> discriminatorMap) {
    // 构建 ResultMapping 对象
    ResultMapping resultMapping = buildResultMapping(
            resultType,
            null,
            column,
            javaType,
            jdbcType,
            null,
            null,
            null,
            null,
            typeHandler,
            new ArrayList<ResultFlag>(),
            null,
            null,
            false);
    // 创建 namespaceDiscriminatorMap 映射
    Map<String, String> namespaceDiscriminatorMap = new HashMap<>();
    for (Map.Entry<String, String> e : discriminatorMap.entrySet()) {
        String resultMap = e.getValue();
        resultMap = applyCurrentNamespace(resultMap, true); // 生成 resultMap 标识
        namespaceDiscriminatorMap.put(e.getKey(), resultMap);
    }
    // 构建 Discriminator 对象
    return new Discriminator.Builder(configuration, resultMapping, namespaceDiscriminatorMap).build();
}
```

- 简单看看就好，`<discriminator />` 平时用的很少。

###### 3.6.1 Discriminator

`org.apache.ibatis.mapping.Discriminator` ，鉴别器，代码比较简单，胖友直接点击 [链接](https://github.com/YunaiV/mybatis-3/blob/master/src/main/java/org/apache/ibatis/mapping/Discriminator.java) 查看，已经添加了完整的注释。

#### 3.7 addResultMap

`#addResultMap(String id, Class<?> type, String extend, Discriminator discriminator, List<ResultMapping> resultMappings, Boolean autoMapping)` 方法，创建 ResultMap 对象，并添加到 Configuration 中。代码如下：

```
// MapperBuilderAssistant.java

public ResultMap addResultMap(
        String id,
        Class<?> type,
        String extend,
        Discriminator discriminator,
        List<ResultMapping> resultMappings,
        Boolean autoMapping) {
    // <1> 获得 ResultMap 编号，即格式为 `${namespace}.${id}` 。
    id = applyCurrentNamespace(id, false);
    // <2.1> 获取完整的 extend 属性，即格式为 `${namespace}.${extend}` 。从这里的逻辑来看，貌似只能自己 namespace 下的 ResultMap 。
    extend = applyCurrentNamespace(extend, true);

    // <2.2> 如果有父类，则将父类的 ResultMap 集合，添加到 resultMappings 中。
    if (extend != null) {
        // <2.2.1> 获得 extend 对应的 ResultMap 对象。如果不存在，则抛出 IncompleteElementException 异常
        if (!configuration.hasResultMap(extend)) {
            throw new IncompleteElementException("Could not find a parent resultmap with id '" + extend + "'");
        }
        ResultMap resultMap = configuration.getResultMap(extend);
        // 获取 extend 的 ResultMap 对象的 ResultMapping 集合，并移除 resultMappings
        List<ResultMapping> extendedResultMappings = new ArrayList<>(resultMap.getResultMappings());
        extendedResultMappings.removeAll(resultMappings);
        // Remove parent constructor if this resultMap declares a constructor.
        // 判断当前的 resultMappings 是否有构造方法，如果有，则从 extendedResultMappings 移除所有的构造类型的 ResultMapping 们
        boolean declaresConstructor = false;
        for (ResultMapping resultMapping : resultMappings) {
            if (resultMapping.getFlags().contains(ResultFlag.CONSTRUCTOR)) {
                declaresConstructor = true;
                break;
            }
        }
        if (declaresConstructor) {
            extendedResultMappings.removeIf(resultMapping -> resultMapping.getFlags().contains(ResultFlag.CONSTRUCTOR));
        }
        // 将 extendedResultMappings 添加到 resultMappings 中
        resultMappings.addAll(extendedResultMappings);
    }
    // <3> 创建 ResultMap 对象
    ResultMap resultMap = new ResultMap.Builder(configuration, id, type, resultMappings, autoMapping)
            .discriminator(discriminator)
            .build();
    // <4> 添加到 configuration 中
    configuration.addResultMap(resultMap);
    return resultMap;
}
```

- `<1>` 处，获得 ResultMap 编号，即格式为 `${namespace}.${id}` 。

- `<2.1>` 处，获取完整的 extend 属性，即格式为 `${namespace}.${extend}` 。**从这里的逻辑来看，貌似只能自己 namespace 下的 ResultMap** 。

- `<2.2>` 处，如果有父类，则将父类的 ResultMap 集合，添加到 `resultMappings` 中。逻辑有些绕，胖友耐心往下看。

  - `<2.2.1>` 处，获得 `extend` 对应的 ResultMap 对象。如果不存在，则抛出 IncompleteElementException 异常。代码如下：

    ```
    // Configuration.java
    
    /**
     * ResultMap 的映射
     *
     * KEY：`${namespace}.${id}`
     */
    protected final Map<String, ResultMap> resultMaps = new StrictMap<>("Result Maps collection");
    
    public ResultMap getResultMap(String id) {
        return resultMaps.get(id);
    }
    
    public boolean hasResultMap(String id) {
        return resultMaps.containsKey(id);
    }
    ```

    - x

- `<3>` 处，创建 ResultMap 对象。详细解析，见 [「3.7.1 ResultMap」](http://svip.iocoder.cn/MyBatis/builder-package-2/#) 。

- `<4>` 处， 调用 `Configuration#addResultMap(ResultMap rm)` 方法，添加到 Configuration 的 `resultMaps` 中。代码如下：

  ```
  // Configuration.java
  
  public void addResultMap(ResultMap rm) {
      // <1> 添加到 resultMaps 中
      resultMaps.put(rm.getId(), rm);
      // 遍历全局的 ResultMap 集合，若其拥有 Discriminator 对象，则判断是否强制标记为有内嵌的 ResultMap
      checkLocallyForDiscriminatedNestedResultMaps(rm);
      // 若传入的 ResultMap 不存在内嵌 ResultMap 并且有 Discriminator ，则判断是否需要强制表位有内嵌的 ResultMap
      checkGloballyForDiscriminatedNestedResultMaps(rm);
  }
  ```

  - `<1>` 处，添加到 `resultMaps` 中。

  - `<2>` 处，调用 `#checkLocallyForDiscriminatedNestedResultMaps(ResultMap rm)` 方法，遍历全局的 ResultMap 集合，若其拥有 Discriminator 对象，则判断是否强制标记为有内嵌的 ResultMap 。代码如下：

    ```
    // Configuration.java
    
    // Slow but a one time cost. A better solution is welcome.
    protected void checkGloballyForDiscriminatedNestedResultMaps(ResultMap rm) {
        // 如果传入的 ResultMap 有内嵌的 ResultMap
        if (rm.hasNestedResultMaps()) {
            // 遍历全局的 ResultMap 集合
            for (Map.Entry<String, ResultMap> entry : resultMaps.entrySet()) {
                Object value = entry.getValue();
                if (value != null) {
                    ResultMap entryResultMap = (ResultMap) value;
                    // 判断遍历的全局的 entryResultMap 不存在内嵌 ResultMap 并且有 Discriminator
                    if (!entryResultMap.hasNestedResultMaps() && entryResultMap.getDiscriminator() != null) {
                        // 判断是否 Discriminator 的 ResultMap 集合中，使用了传入的 ResultMap 。
                        // 如果是，则标记为有内嵌的 ResultMap
                        Collection<String> discriminatedResultMapNames = entryResultMap.getDiscriminator().getDiscriminatorMap().values();
                        if (discriminatedResultMapNames.contains(rm.getId())) {
                            entryResultMap.forceNestedResultMaps();
                        }
                    }
                }
            }
        }
    }
    ```

    - 逻辑有点绕，胖友耐心看下去。。。

  - `<3>` 处，调用 `#checkLocallyForDiscriminatedNestedResultMaps(ResultMap rm)` 方法，若传入的 ResultMap 不存在内嵌 ResultMap 并且有 Discriminator ，则判断是否需要强制表位有内嵌的 ResultMap 。代码如下：

    ```
    // Configuration.java
    
    // Slow but a one time cost. A better solution is welcome.
    protected void checkLocallyForDiscriminatedNestedResultMaps(ResultMap rm) {
        // 如果传入的 ResultMap 不存在内嵌 ResultMap 并且有 Discriminator
        if (!rm.hasNestedResultMaps() && rm.getDiscriminator() != null) {
            // 遍历传入的 ResultMap 的 Discriminator 的 ResultMap 集合
            for (Map.Entry<String, String> entry : rm.getDiscriminator().getDiscriminatorMap().entrySet()) {
                String discriminatedResultMapName = entry.getValue();
                if (hasResultMap(discriminatedResultMapName)) {
                    // 如果引用的 ResultMap 存在内嵌 ResultMap ，则标记传入的 ResultMap 存在内嵌 ResultMap
                    ResultMap discriminatedResultMap = resultMaps.get(discriminatedResultMapName);
                    if (discriminatedResultMap.hasNestedResultMaps()) {
                        rm.forceNestedResultMaps();
                        break;
                    }
                }
            }
        }
    }
    ```

    - 逻辑有点绕，胖友耐心看下去。。。整体逻辑，和 `#checkGloballyForDiscriminatedNestedResultMaps(ResultMap rm)` 方法是**类似**的，互为“倒影”。

###### 3.7.1 ResultMap

`org.apache.ibatis.mapping.ResultMap` ，结果集，例如 `<resultMap />` 解析后的对象。代码比较简单，但是有点略长，胖友直接点击 [链接](https://github.com/YunaiV/mybatis-3/blob/master/src/main/java/org/apache/ibatis/mapping/ResultMap.java) 查看，已经添加了完整的注释。



# 3、加载 Statement 配置

## 1. 概述

本文接 [《精尽 MyBatis 源码分析 —— MyBatis 初始化（二）之加载 Mapper 映射配置文件》](http://svip.iocoder.cn/MyBatis/builder-package-2) 一文，来分享 MyBatis 初始化的第三步，**加载 Statement 配置**。而这个步骤的入口是 XMLStatementBuilder 。下面，我们一起来看看它的代码实现。

在 [《精尽 MyBatis 源码分析 —— MyBatis 初始化（二）之加载 Mapper 映射配置文件》](http://svip.iocoder.cn/MyBatis/builder-package-2) 的 [「2.3.5 buildStatementFromContext」](http://svip.iocoder.cn/MyBatis/builder-package-3/#) 中，我们已经看到对 XMLStatementBuilder 的调用代码。代码如下：

```
// XMLMapperBuilder.java

private void buildStatementFromContext(List<XNode> list) {
    if (configuration.getDatabaseId() != null) {
        buildStatementFromContext(list, configuration.getDatabaseId());
    }
    buildStatementFromContext(list, null);
    // 上面两块代码，可以简写成 buildStatementFromContext(list, configuration.getDatabaseId());
}

private void buildStatementFromContext(List<XNode> list, String requiredDatabaseId) {
    // <1> 遍历 <select /> <insert /> <update /> <delete /> 节点们
    for (XNode context : list) {
        // <1> 创建 XMLStatementBuilder 对象，执行解析
        final XMLStatementBuilder statementParser = new XMLStatementBuilder(configuration, builderAssistant, context, requiredDatabaseId);
        try {
            statementParser.parseStatementNode();
        } catch (IncompleteElementException e) {
            // <2> 解析失败，添加到 configuration 中
            configuration.addIncompleteStatement(statementParser);
        }
    }
}
```

## 2. XMLStatementBuilder

`org.apache.ibatis.builder.xml.XMLStatementBuilder` ，继承 BaseBuilder 抽象类，Statement XML 配置构建器，主要负责解析 Statement 配置，即 `<select />`、`<insert />`、`<update />`、`<delete />` 标签。

#### 2.1 构造方法

```
// XMLStatementBuilder.java

private final MapperBuilderAssistant builderAssistant;
/**
 * 当前 XML 节点，例如：<select />、<insert />、<update />、<delete /> 标签
 */
private final XNode context;
/**
 * 要求的 databaseId
 */
private final String requiredDatabaseId;

public XMLStatementBuilder(Configuration configuration, MapperBuilderAssistant builderAssistant, XNode context, String databaseId) {
    super(configuration);
    this.builderAssistant = builderAssistant;
    this.context = context;
    this.requiredDatabaseId = databaseId;
}
```

#### 2.2 parseStatementNode

`#parseStatementNode()` 方法，执行 Statement 解析。代码如下：

```
// XMLStatementBuilder.java

public void parseStatementNode() {
    // <1> 获得 id 属性，编号。
    String id = context.getStringAttribute("id");
    // <2> 获得 databaseId ， 判断 databaseId 是否匹配
    String databaseId = context.getStringAttribute("databaseId");
    if (!databaseIdMatchesCurrent(id, databaseId, this.requiredDatabaseId)) {
        return;
    }

    // <3> 获得各种属性
    Integer fetchSize = context.getIntAttribute("fetchSize");
    Integer timeout = context.getIntAttribute("timeout");
    String parameterMap = context.getStringAttribute("parameterMap");
    String parameterType = context.getStringAttribute("parameterType");
    Class<?> parameterTypeClass = resolveClass(parameterType);
    String resultMap = context.getStringAttribute("resultMap");
    String resultType = context.getStringAttribute("resultType");
    String lang = context.getStringAttribute("lang");

    // <4> 获得 lang 对应的 LanguageDriver 对象
    LanguageDriver langDriver = getLanguageDriver(lang);

    // <5> 获得 resultType 对应的类
    Class<?> resultTypeClass = resolveClass(resultType);
    // <6> 获得 resultSet 对应的枚举值
    String resultSetType = context.getStringAttribute("resultSetType");
    ResultSetType resultSetTypeEnum = resolveResultSetType(resultSetType);
    // <7> 获得 statementType 对应的枚举值
    StatementType statementType = StatementType.valueOf(context.getStringAttribute("statementType", StatementType.PREPARED.toString()));

    // <8> 获得 SQL 对应的 SqlCommandType 枚举值
    String nodeName = context.getNode().getNodeName();
    SqlCommandType sqlCommandType = SqlCommandType.valueOf(nodeName.toUpperCase(Locale.ENGLISH));
    // <9> 获得各种属性
    boolean isSelect = sqlCommandType == SqlCommandType.SELECT;
    boolean flushCache = context.getBooleanAttribute("flushCache", !isSelect);
    boolean useCache = context.getBooleanAttribute("useCache", isSelect);
    boolean resultOrdered = context.getBooleanAttribute("resultOrdered", false);

    // Include Fragments before parsing
    // <10> 创建 XMLIncludeTransformer 对象，并替换 <include /> 标签相关的内容
    XMLIncludeTransformer includeParser = new XMLIncludeTransformer(configuration, builderAssistant);
    includeParser.applyIncludes(context.getNode());

    // Parse selectKey after includes and remove them.
    // <11> 解析 <selectKey /> 标签
    processSelectKeyNodes(id, parameterTypeClass, langDriver);

    // Parse the SQL (pre: <selectKey> and <include> were parsed and removed)
    // <12> 创建 SqlSource
    SqlSource sqlSource = langDriver.createSqlSource(configuration, context, parameterTypeClass);
    // <13> 获得 KeyGenerator 对象
    String resultSets = context.getStringAttribute("resultSets");
    String keyProperty = context.getStringAttribute("keyProperty");
    String keyColumn = context.getStringAttribute("keyColumn");
    KeyGenerator keyGenerator;
    // <13.1> 优先，从 configuration 中获得 KeyGenerator 对象。如果存在，意味着是 <selectKey /> 标签配置的
    String keyStatementId = id + SelectKeyGenerator.SELECT_KEY_SUFFIX;
    keyStatementId = builderAssistant.applyCurrentNamespace(keyStatementId, true);
    if (configuration.hasKeyGenerator(keyStatementId)) {
        keyGenerator = configuration.getKeyGenerator(keyStatementId);
    // <13.2> 其次，根据标签属性的情况，判断是否使用对应的 Jdbc3KeyGenerator 或者 NoKeyGenerator 对象
    } else {
        keyGenerator = context.getBooleanAttribute("useGeneratedKeys", // 优先，基于 useGeneratedKeys 属性判断
                configuration.isUseGeneratedKeys() && SqlCommandType.INSERT.equals(sqlCommandType)) // 其次，基于全局的 useGeneratedKeys 配置 + 是否为插入语句类型
                ? Jdbc3KeyGenerator.INSTANCE : NoKeyGenerator.INSTANCE;
    }

    // 创建 MappedStatement 对象
    builderAssistant.addMappedStatement(id, sqlSource, statementType, sqlCommandType,
            fetchSize, timeout, parameterMap, parameterTypeClass, resultMap, resultTypeClass,
            resultSetTypeEnum, flushCache, useCache, resultOrdered,
            keyGenerator, keyProperty, keyColumn, databaseId, langDriver, resultSets);
}
```

- `<1>` 处，获得 `id` 属性，编号。
- `<2>` 处，获得 `databaseId` 属性，并调用 `#databaseIdMatchesCurrent(String id, String databaseId, String requiredDatabaseId)` 方法，判断 `databaseId` 是否匹配。详细解析，见 [「2.3 databaseIdMatchesCurrent」](http://svip.iocoder.cn/MyBatis/builder-package-3/#) 。
- `<3>` 处，获得各种属性。
- `<4>` 处，调用 `#getLanguageDriver(String lang)` 方法，获得 `lang` 对应的 LanguageDriver 对象。详细解析，见 [「2.4 getLanguageDriver」](http://svip.iocoder.cn/MyBatis/builder-package-3/#) 。
- `<5>` 处，获得 `resultType` 对应的类。
- `<6>` 处，获得 `resultSet` 对应的枚举值。关于 `org.apache.ibatis.mapping.ResultSetType` 枚举类，点击[查看](https://github.com/YunaiV/mybatis-3/blob/master/src/main/java/org/apache/ibatis/mapping/ResultSetType.java)。一般情况下，不会设置该值。它是基于 `java.sql.ResultSet` 结果集的几种模式，感兴趣的话，可以看看 [《ResultSet 的 Type 属性》](http://jinguo.iteye.com/blog/365373) 。
- `<7>` 处，获得 `statementType` 对应的枚举值。关于 `org.apache.ibatis.mapping.StatementType` 枚举类，点击[查看](https://github.com/YunaiV/mybatis-3/blob/master/src/main/java/org/apache/ibatis/mapping/StatementType.java)。
- `<8>` 处，获得 SQL 对应的 SqlCommandType 枚举值。
- `<9>` 处，获得各种属性。
- `<10>` 处，创建 XMLIncludeTransformer 对象，并调用 `XMLIncludeTransformer#applyIncludes(Node source)` 方法，替换 `<include />` 标签相关的内容。详细解析，见 [「3. XMLIncludeTransformer」](http://svip.iocoder.cn/MyBatis/builder-package-3/#) 。
- `<11>` 处，调用 `#processSelectKeyNodes(String id, Class<?> parameterTypeClass, LanguageDriver langDriver)` 方法，解析 `<selectKey />` 标签。详细解析，见 [「2.5 processSelectKeyNodes」](http://svip.iocoder.cn/MyBatis/builder-package-3/#) 。
- `<12>` 处，调用 `LanguageDriver#createSqlSource(Configuration configuration, XNode script, Class<?> parameterType)` 方法，创建 SqlSource 对象。详细解析，见后续文章。
- `<13>` 处，获得 KeyGenerator 对象。分成 `<13.1>` 和 `<13.2>` 两种情况。具体的，胖友耐心看下代码注释。
- `<14>` 处，调用 `MapperBuilderAssistant#addMappedStatement(...)` 方法，创建 MappedStatement 对象。详细解析，见 [「4.1 addMappedStatement」](http://svip.iocoder.cn/MyBatis/builder-package-3/#) 中。

#### 2.3 databaseIdMatchesCurrent

`#databaseIdMatchesCurrent(String id, String databaseId, String requiredDatabaseId)` 方法，判断 `databaseId` 是否匹配。代码如下：

```
// XMLStatementBuilder.java

private boolean databaseIdMatchesCurrent(String id, String databaseId, String requiredDatabaseId) {
    // 如果不匹配，则返回 false
    if (requiredDatabaseId != null) {
        return requiredDatabaseId.equals(databaseId);
    } else {
        // 如果未设置 requiredDatabaseId ，但是 databaseId 存在，说明还是不匹配，则返回 false
        // mmp ，写的好绕
        if (databaseId != null) {
            return false;
        }
        // skip this statement if there is a previous one with a not null databaseId
        // 判断是否已经存在
        id = builderAssistant.applyCurrentNamespace(id, false);
        if (this.configuration.hasStatement(id, false)) {
            MappedStatement previous = this.configuration.getMappedStatement(id, false); // issue #2
            // 若存在，则判断原有的 sqlFragment 是否 databaseId 为空。因为，当前 databaseId 为空，这样两者才能匹配。
            return previous.getDatabaseId() == null;
        }
    }
    return true;
}
```

- 代码比较简单，胖友自己瞅瞅就得。从逻辑上，和我们在 XMLMapperBuilder 看到的同名方法 `#databaseIdMatchesCurrent(String id, String databaseId, String requiredDatabaseId)` 方法是一致的。

#### 2.4 getLanguageDriver

`#getLanguageDriver(String lang)` 方法，获得 `lang` 对应的 LanguageDriver 对象。代码如下：

```
// XMLStatementBuilder.java

private LanguageDriver getLanguageDriver(String lang) {
    // 解析 lang 对应的类
    Class<? extends LanguageDriver> langClass = null;
    if (lang != null) {
        langClass = resolveClass(lang);
    }
    // 获得 LanguageDriver 对象
    return builderAssistant.getLanguageDriver(langClass);
}
```

- 调用 `MapperBuilderAssistant#getLanguageDriver(lass<? extends LanguageDriver> langClass)` 方法，获得 LanguageDriver 对象。代码如下：

  ```
  // MapperBuilderAssistant.java
  
  public LanguageDriver getLanguageDriver(Class<? extends LanguageDriver> langClass) {
      // 获得 langClass 类
      if (langClass != null) {
          configuration.getLanguageRegistry().register(langClass);
      } else { // 如果为空，则使用默认类
          langClass = configuration.getLanguageRegistry().getDefaultDriverClass();
      }
      // 获得 LanguageDriver 对象
      return configuration.getLanguageRegistry().getDriver(langClass);
  }
  ```

  - 关于 `org.apache.ibatis.scripting.LanguageDriverRegistry` 类，我们在后续的文章，详细解析。

#### 2.5 processSelectKeyNodes

`#processSelectKeyNodes(String id, Class<?> parameterTypeClass, LanguageDriver langDriver)` 方法，解析 `<selectKey />` 标签。代码如下：

```
// XMLStatementBuilder.java

private void processSelectKeyNodes(String id, Class<?> parameterTypeClass, LanguageDriver langDriver) {
    // <1> 获得 <selectKey /> 节点们
    List<XNode> selectKeyNodes = context.evalNodes("selectKey");
    // <2> 执行解析 <selectKey /> 节点们
    if (configuration.getDatabaseId() != null) {
        parseSelectKeyNodes(id, selectKeyNodes, parameterTypeClass, langDriver, configuration.getDatabaseId());
    }
    parseSelectKeyNodes(id, selectKeyNodes, parameterTypeClass, langDriver, null);
    // <3> 移除 <selectKey /> 节点们
    removeSelectKeyNodes(selectKeyNodes);
}
```

- `<1>` 处，获得 `<selectKey />` 节点们。

- `<2>` 处，调用 `#parseSelectKeyNodes(...)` 方法，执行解析 `<selectKey />` 节点们。详细解析，见 [「2.5.1 parseSelectKeyNodes」](http://svip.iocoder.cn/MyBatis/builder-package-3/#) 。

- `<3>` 处，调用 `#removeSelectKeyNodes(List<XNode> selectKeyNodes)` 方法，移除 `<selectKey />` 节点们。代码如下：

  ```
  // XMLStatementBuilder.java
  
  private void removeSelectKeyNodes(List<XNode> selectKeyNodes) {
      for (XNode nodeToHandle : selectKeyNodes) {
          nodeToHandle.getParent().getNode().removeChild(nodeToHandle.getNode());
      }
  }
  ```

###### 2.5.1 parseSelectKeyNodes

`#parseSelectKeyNodes(String parentId, List<XNode> list, Class<?> parameterTypeClass, LanguageDriver langDriver, String skRequiredDatabaseId)` 方法，执行解析 `<selectKey />` 子节点们。代码如下：

```
// XMLStatementBuilder.java

private void parseSelectKeyNodes(String parentId, List<XNode> list, Class<?> parameterTypeClass, LanguageDriver langDriver, String skRequiredDatabaseId) {
    // <1> 遍历 <selectKey /> 节点们
    for (XNode nodeToHandle : list) {
        // <2> 获得完整 id ，格式为 `${id}!selectKey`
        String id = parentId + SelectKeyGenerator.SELECT_KEY_SUFFIX;
        // <3> 获得 databaseId ， 判断 databaseId 是否匹配
        String databaseId = nodeToHandle.getStringAttribute("databaseId");
        if (databaseIdMatchesCurrent(id, databaseId, skRequiredDatabaseId)) {
            // <4> 执行解析单个 <selectKey /> 节点
            parseSelectKeyNode(id, nodeToHandle, parameterTypeClass, langDriver, databaseId);
        }
    }
}
```

- `<1>` 处，遍历 `<selectKey />` 节点们，逐个处理。
- `<2>` 处，获得完整 `id` 编号，格式为 `${id}!selectKey` 。这里很重要，最终解析的 `<selectKey />` 节点，会创建成一个 MappedStatement 对象。而该对象的编号，就是 `id` 。
- `<3>` 处，获得 `databaseId` ，并调用 `#databaseIdMatchesCurrent(String id, String databaseId, String requiredDatabaseId)` 方法，判断 `databaseId` 是否匹配。😈 通过此处，我们可以看到，即使有多个 `<selectionKey />` 节点，但是最终只会有一个节点被解析，就是符合的 `databaseId` 对应的。因为不同的数据库实现不同，对于获取主键的方式也会不同。
- `<4>` 处，调用 `#parseSelectKeyNode(...)` 方法，执行解析**单个** `<selectKey />` 节点。详细解析，见 [「2.5.2 parseSelectKeyNode」](http://svip.iocoder.cn/MyBatis/builder-package-3/#) 。

###### 2.5.2 parseSelectKeyNode

`#parseSelectKeyNode(...)` 方法，执行解析**单个** `<selectKey />` 节点。代码如下：

```
// XMLStatementBuilder.java

private void parseSelectKeyNode(String id, XNode nodeToHandle, Class<?> parameterTypeClass, LanguageDriver langDriver, String databaseId) {
    // <1.1> 获得各种属性和对应的类
    String resultType = nodeToHandle.getStringAttribute("resultType");
    Class<?> resultTypeClass = resolveClass(resultType);
    StatementType statementType = StatementType.valueOf(nodeToHandle.getStringAttribute("statementType", StatementType.PREPARED.toString()));
    String keyProperty = nodeToHandle.getStringAttribute("keyProperty");
    String keyColumn = nodeToHandle.getStringAttribute("keyColumn");
    boolean executeBefore = "BEFORE".equals(nodeToHandle.getStringAttribute("order", "AFTER"));

    // defaults
    // <1.2> 创建 MappedStatement 需要用到的默认值
    boolean useCache = false;
    boolean resultOrdered = false;
    KeyGenerator keyGenerator = NoKeyGenerator.INSTANCE;
    Integer fetchSize = null;
    Integer timeout = null;
    boolean flushCache = false;
    String parameterMap = null;
    String resultMap = null;
    ResultSetType resultSetTypeEnum = null;

    // <1.3> 创建 SqlSource 对象
    SqlSource sqlSource = langDriver.createSqlSource(configuration, nodeToHandle, parameterTypeClass);
    SqlCommandType sqlCommandType = SqlCommandType.SELECT;

    // <1.4> 创建 MappedStatement 对象
    builderAssistant.addMappedStatement(id, sqlSource, statementType, sqlCommandType,
            fetchSize, timeout, parameterMap, parameterTypeClass, resultMap, resultTypeClass,
            resultSetTypeEnum, flushCache, useCache, resultOrdered,
            keyGenerator, keyProperty, keyColumn, databaseId, langDriver, null);

    // <2.1> 获得 SelectKeyGenerator 的编号，格式为 `${namespace}.${id}`
    id = builderAssistant.applyCurrentNamespace(id, false);
    // <2.2> 获得 MappedStatement 对象
    MappedStatement keyStatement = configuration.getMappedStatement(id, false);
    // <2.3> 创建 SelectKeyGenerator 对象，并添加到 configuration 中
    configuration.addKeyGenerator(id, new SelectKeyGenerator(keyStatement, executeBefore));
}
```

- `<1.1>` 处理，获得各种属性和对应的类。

- `<1.2>` 处理，创建 MappedStatement 需要用到的默认值。

- `<1.3>` 处理，调用 `LanguageDriver#createSqlSource(Configuration configuration, XNode script, Class<?> parameterType)` 方法，创建 SqlSource 对象。详细解析，见后续文章。

- `<1.4>` 处理，调用 `MapperBuilderAssistant#addMappedStatement(...)` 方法，创建 MappedStatement 对象。

- `<2.1>` 处理，获得 SelectKeyGenerator 的编号，格式为 `${namespace}.${id}` 。

- `<2.2>` 处理，获得 MappedStatement 对象。该对象，实际就是 `<1.4>` 处创建的 MappedStatement 对象。

- `<2.3>` 处理，调用 `Configuration#addKeyGenerator(String id, KeyGenerator keyGenerator)` 方法，创建 SelectKeyGenerator 对象，并添加到 `configuration` 中。代码如下：

  ```
  // Configuration.java
  
  /**
   * KeyGenerator 的映射
   *
   * KEY：在 {@link #mappedStatements} 的 KEY 的基础上，跟上 {@link SelectKeyGenerator#SELECT_KEY_SUFFIX}
   */
  protected final Map<String, KeyGenerator> keyGenerators = new StrictMap<>("Key Generators collection");
  
  public void addKeyGenerator(String id, KeyGenerator keyGenerator) {
      keyGenerators.put(id, keyGenerator);
  }
  ```

## 3. XMLIncludeTransformer

`org.apache.ibatis.builder.xml.XMLIncludeTransformer` ，XML `<include />` 标签的转换器，负责将 SQL 中的 `<include />` 标签转换成对应的 `<sql />` 的内容。

#### 3.1 构造方法

```
// XMLIncludeTransformer.java

private final Configuration configuration;
private final MapperBuilderAssistant builderAssistant;

public XMLIncludeTransformer(Configuration configuration, MapperBuilderAssistant builderAssistant) {
    this.configuration = configuration;
    this.builderAssistant = builderAssistant;
}
```

#### 3.2 applyIncludes

`#applyIncludes(Node source)` 方法，将 `<include />` 标签，替换成引用的 `<sql />` 。代码如下：

```
// XMLIncludeTransformer.java

public void applyIncludes(Node source) {
    // <1> 创建 variablesContext ，并将 configurationVariables 添加到其中
    Properties variablesContext = new Properties();
    Properties configurationVariables = configuration.getVariables();
    if (configurationVariables != null) {
        variablesContext.putAll(configurationVariables);
    }
    // <2> 处理 <include />
    applyIncludes(source, variablesContext, false);
}
```

- `<1>` 处，创建 `variablesContext` ，并将 `configurationVariables` 添加到其中。这里的目的是，避免 `configurationVariables` 被下面使用时候，可能被修改。实际上，从下面的实现上，不存在这个情况。
- `<2>` 处，调用 `#applyIncludes(Node source, final Properties variablesContext, boolean included)` 方法，处理 `<include />` 。

------

`#applyIncludes(Node source, final Properties variablesContext, boolean included)` 方法，使用递归的方式，将 `<include />` 标签，替换成引用的 `<sql />` 。代码如下：

```
// XMLIncludeTransformer.java

private void applyIncludes(Node source, final Properties variablesContext, boolean included) {
    // <1> 如果是 <include /> 标签
    if (source.getNodeName().equals("include")) {
        // <1.1> 获得 <sql /> 对应的节点
        Node toInclude = findSqlFragment(getStringAttribute(source, "refid"), variablesContext);
        // <1.2> 获得包含 <include /> 标签内的属性
        Properties toIncludeContext = getVariablesContext(source, variablesContext);
        // <1.3> 递归调用 #applyIncludes(...) 方法，继续替换。注意，此处是 <sql /> 对应的节点
        applyIncludes(toInclude, toIncludeContext, true);
        if (toInclude.getOwnerDocument() != source.getOwnerDocument()) { // 这个情况，艿艿暂时没调试出来
            toInclude = source.getOwnerDocument().importNode(toInclude, true);
        }
        // <1.4> 将 <include /> 节点替换成 <sql /> 节点
        source.getParentNode().replaceChild(toInclude, source); // 注意，这是一个奇葩的 API ，前者为 newNode ，后者为 oldNode
        // <1.4> 将 <sql /> 子节点添加到 <sql /> 节点前面
        while (toInclude.hasChildNodes()) {
            toInclude.getParentNode().insertBefore(toInclude.getFirstChild(), toInclude); // 这里有个点，一定要注意，卡了艿艿很久。当子节点添加到其它节点下面后，这个子节点会不见了，相当于是“移动操作”
        }
        // <1.4> 移除 <include /> 标签自身
        toInclude.getParentNode().removeChild(toInclude);
    // <2> 如果节点类型为 Node.ELEMENT_NODE
    } else if (source.getNodeType() == Node.ELEMENT_NODE) {
        // <2.1> 如果在处理 <include /> 标签中，则替换其上的属性，例如 <sql id="123" lang="${cpu}"> 的情况，lang 属性是可以被替换的
        if (included && !variablesContext.isEmpty()) {
            // replace variables in attribute values
            NamedNodeMap attributes = source.getAttributes();
            for (int i = 0; i < attributes.getLength(); i++) {
                Node attr = attributes.item(i);
                attr.setNodeValue(PropertyParser.parse(attr.getNodeValue(), variablesContext));
            }
        }
        // <2.2> 遍历子节点，递归调用 #applyIncludes(...) 方法，继续替换
        NodeList children = source.getChildNodes();
        for (int i = 0; i < children.getLength(); i++) {
            applyIncludes(children.item(i), variablesContext, included);
        }
    // <3> 如果在处理 <include /> 标签中，并且节点类型为 Node.TEXT_NODE ，并且变量非空
    // 则进行变量的替换，并修改原节点 source
    } else if (included && source.getNodeType() == Node.TEXT_NODE
            && !variablesContext.isEmpty()) {
        // replace variables in text node
        source.setNodeValue(PropertyParser.parse(source.getNodeValue(), variablesContext));
    }
}
```

- 这是个有**自递归逻辑**的方法，所以理解起来会有点绕，实际上还是蛮简单的。为了更好的解释，我们假设示例如下：

  ```
  // mybatis-config.xml
  
  <properties>
      <property name="cpu" value="16c" />
      <property name="target_sql" value="123" />
  </properties>
  
  // Mapper.xml
  
  <sql id="123" lang="${cpu}">
      ${cpu}
      aoteman
      qqqq
  </sql>
  
  <select id="testForInclude">
      SELECT * FROM subject
      <include refid="${target_sql}" />
  </select>
  ```

- 方法参数 `included` ，是否**正在**处理 `<include />` 标签中。😈 一脸懵逼？不要方，继续往下看。

- 在上述示例的

   

  ```
  <select />
  ```

   

  节点进入这个方法时，会首先进入

   

  ```
  <2>
  ```

   

  这块逻辑。

  - `<2.1>` 处，因为 不满足 `included` 条件，初始传入是 `false` ，所以跳过。

  - ```
    <2.2>
    ```

     

    处，遍历子节点，

    递归

    调用

     

    ```
    #applyIncludes(...)
    ```

     

    方法，继续替换。如图所示：

    ![子节点](http://static.iocoder.cn/images/MyBatis/2020_02_16/01.png)

    子节点

    - 子节点 `[0]` 和 `[2]` ，执行该方法时，不满足 `<1>`、`<2>`、`<3>` 任一一种情况，所以可以忽略。虽然说，满足 `<3>` 的节点类型为 `Node.TEXT_NODE` ，但是 `included` 此时为 `false` ，所以不满足。
    - 子节点 `[1]` ，执行该方法时，满足 `<1>` 的情况，所以走起。

- 在子节点

   

  ```
  [1]
  ```

   

  ，即

   

  ```
  <include />
  ```

   

  节点进入

   

  ```
  <1>
  ```

   

  这块逻辑：

  - `<1.1>` 处，调用 `#findSqlFragment(String refid, Properties variables)` 方法，获得 `<sql />` 对应的节点，即上述示例看到的，`<sql id="123" lang="${cpu}"> ... </>` 。详细解析，见 [「3.3 findSqlFragment」](http://svip.iocoder.cn/MyBatis/builder-package-3/#) 。
  - `<1.2>` 处，调用 `#getVariablesContext(Node node, Properties inheritedVariablesContext)` 方法，获得包含 `<include />` 标签内的属性 Properties 对象。详细解析，见 [「3.4 getVariablesContext」](http://svip.iocoder.cn/MyBatis/builder-package-3/#) 。
  - `<1.3>` 处，**递归**调用 `#applyIncludes(...)` 方法，继续替换。注意，此处是 `<sql />` 对应的节点，并且 `included` 参数为 `true` 。详细的结果，见 😈😈😈 处。
  - `<1.4>` 处，将处理好的 `<sql />` 节点，替换掉 `<include />` 节点。逻辑有丢丢绕，胖友耐心看下注释，好好思考。

- 😈😈😈 在

   

  ```
  <sql />
  ```

   

  节点，会进入

   

  ```
  <2>
  ```

   

  这块逻辑：

  - `<2.1>` 处，因为 `included` 为 `true` ，所以能满足这块逻辑，会进行执行。如 `<sql id="123" lang="${cpu}">` 的情况，`lang` 属性是可以被替换的。

  - ```
    <2.2>
    ```

     

    处，遍历子节点，

    递归

    调用

     

    ```
    #applyIncludes(...)
    ```

     

    方法，继续替换。如图所示：

    ![子节点](http://static.iocoder.cn/images/MyBatis/2020_02_16/02.png)

    子节点

    - 子节点 `[0]` ，执行该方法时，满足 `<3>` 的情况，所以可以使用变量 Properteis 对象，进行替换，并修改原节点。

其实，整理一下，逻辑也不会很绕。耐心耐心耐心。

#### 3.3 findSqlFragment

`#findSqlFragment(String refid, Properties variables)` 方法，获得对应的 `<sql />` 节点。代码如下：

```
// XMLIncludeTransformer.java

private Node findSqlFragment(String refid, Properties variables) {
    // 因为 refid 可能是动态变量，所以进行替换
    refid = PropertyParser.parse(refid, variables);
    // 获得完整的 refid ，格式为 "${namespace}.${refid}"
    refid = builderAssistant.applyCurrentNamespace(refid, true);
    try {
        // 获得对应的 <sql /> 节点
        XNode nodeToInclude = configuration.getSqlFragments().get(refid);
        // 获得 Node 节点，进行克隆
        return nodeToInclude.getNode().cloneNode(true);
    } catch (IllegalArgumentException e) {
        throw new IncompleteElementException("Could not find SQL statement to include with refid '" + refid + "'", e);
    }
}

private String getStringAttribute(Node node, String name) {
    return node.getAttributes().getNamedItem(name).getNodeValue();
}
```

- 比较简单，胖友瞅瞅注释。

#### 3.4 getVariablesContext

`#getVariablesContext(Node node, Properties inheritedVariablesContext)` 方法，获得包含 `<include />` 标签内的属性 Properties 对象。代码如下：

```
// XMLIncludeTransformer.java

private Properties getVariablesContext(Node node, Properties inheritedVariablesContext) {
    // 获得 <include /> 标签的属性集合
    Map<String, String> declaredProperties = null;
    NodeList children = node.getChildNodes();
    for (int i = 0; i < children.getLength(); i++) {
        Node n = children.item(i);
        if (n.getNodeType() == Node.ELEMENT_NODE) {
            String name = getStringAttribute(n, "name");
            // Replace variables inside
            String value = PropertyParser.parse(getStringAttribute(n, "value"), inheritedVariablesContext);
            if (declaredProperties == null) {
                declaredProperties = new HashMap<>();
            }
            if (declaredProperties.put(name, value) != null) { // 如果重复定义，抛出异常
                throw new BuilderException("Variable " + name + " defined twice in the same include definition");
            }
        }
    }
    // 如果 <include /> 标签内没有属性，直接使用 inheritedVariablesContext 即可
    if (declaredProperties == null) {
        return inheritedVariablesContext;
    // 如果 <include /> 标签内有属性，则创建新的 newProperties 集合，将 inheritedVariablesContext + declaredProperties 合并
    } else {
        Properties newProperties = new Properties();
        newProperties.putAll(inheritedVariablesContext);
        newProperties.putAll(declaredProperties);
        return newProperties;
    }
}
```

- 比较简单，胖友瞅瞅注释。

- 如下是 `<include />` 标签内有属性的示例：

  ```
  <sql id="userColumns"> ${alias}.id,${alias}.username,${alias}.password </sql>
  
  <select id="selectUsers" resultType="map">
    select
      <include refid="userColumns"><property name="alias" value="t1"/></include>,
      <include refid="userColumns"><property name="alias" value="t2"/></include>
    from some_table t1
      cross join some_table t2
  </select>
  ```

## 4. MapperBuilderAssistant

#### 4.1 addMappedStatement

`#addMappedStatement(String id, SqlSource sqlSource, StatementType statementType, SqlCommandType sqlCommandType, Integer fetchSize, Integer timeout, String parameterMap, Class<?> parameterType, String resultMap, Class<?> resultType, ResultSetType resultSetType, boolean flushCache, boolean useCache, boolean resultOrdered, KeyGenerator keyGenerator, String keyProperty, String keyColumn, String databaseId, LanguageDriver lang, String resultSets)` 方法，构建 MappedStatement 对象。代码如下：

```
// MapperBuilderAssistant.java

public MappedStatement addMappedStatement(
        String id,
        SqlSource sqlSource,
        StatementType statementType,
        SqlCommandType sqlCommandType,
        Integer fetchSize,
        Integer timeout,
        String parameterMap,
        Class<?> parameterType,
        String resultMap,
        Class<?> resultType,
        ResultSetType resultSetType,
        boolean flushCache,
        boolean useCache,
        boolean resultOrdered,
        KeyGenerator keyGenerator,
        String keyProperty,
        String keyColumn,
        String databaseId,
        LanguageDriver lang,
        String resultSets) {

    // <1> 如果只想的 Cache 未解析，抛出 IncompleteElementException 异常
    if (unresolvedCacheRef) {
        throw new IncompleteElementException("Cache-ref not yet resolved");
    }

    // <2> 获得 id 编号，格式为 `${namespace}.${id}`
    id = applyCurrentNamespace(id, false);
    boolean isSelect = sqlCommandType == SqlCommandType.SELECT;

    // <3> 创建 MappedStatement.Builder 对象
    MappedStatement.Builder statementBuilder = new MappedStatement.Builder(configuration, id, sqlSource, sqlCommandType)
            .resource(resource)
            .fetchSize(fetchSize)
            .timeout(timeout)
            .statementType(statementType)
            .keyGenerator(keyGenerator)
            .keyProperty(keyProperty)
            .keyColumn(keyColumn)
            .databaseId(databaseId)
            .lang(lang)
            .resultOrdered(resultOrdered)
            .resultSets(resultSets)
            .resultMaps(getStatementResultMaps(resultMap, resultType, id)) // <3.1> 获得 ResultMap 集合
            .resultSetType(resultSetType)
            .flushCacheRequired(valueOrDefault(flushCache, !isSelect))
            .useCache(valueOrDefault(useCache, isSelect))
            .cache(currentCache);

    // <3.2> 获得 ParameterMap ，并设置到 MappedStatement.Builder 中
    ParameterMap statementParameterMap = getStatementParameterMap(parameterMap, parameterType, id);
    if (statementParameterMap != null) {
        statementBuilder.parameterMap(statementParameterMap);
    }

    // <4> 创建 MappedStatement 对象
    MappedStatement statement = statementBuilder.build();
    // <5> 添加到 configuration 中
    configuration.addMappedStatement(statement);
    return statement;
}
```

- `<1>` 处，如果只想的 Cache 未解析，抛出 IncompleteElementException 异常。

- `<2>` 处，获得 `id` 编号，格式为 `${namespace}.${id}` 。

- ```
  <3>
  ```

   

  处，创建 MappedStatement.Builder 对象。详细解析，见

   

  「4.1.3 MappedStatement」

   

  。

  - `<3.1>` 处，调用 `#getStatementResultMaps(...)` 方法，获得 ResultMap 集合。详细解析，见 [「4.1.3 getStatementResultMaps」](http://svip.iocoder.cn/MyBatis/builder-package-3/#) 。
  - `<3.2>` 处，调用 `#getStatementParameterMap(...)` 方法，获得 ParameterMap ，并设置到 MappedStatement.Builder 中。详细解析，见 [4.1.4 getStatementResultMaps」](http://svip.iocoder.cn/MyBatis/builder-package-3/#) 。

- `<4>` 处，创建 MappedStatement 对象。详细解析，见 [「4.1.1 MappedStatement」](http://svip.iocoder.cn/MyBatis/builder-package-3/#) 。

- `<5>` 处，调用 `Configuration#addMappedStatement(statement)` 方法，添加到 `configuration` 中。代码如下：

  ```
  // Configuration.java
  
  /**
   * MappedStatement 映射
   *
   * KEY：`${namespace}.${id}`
   */
  protected final Map<String, MappedStatement> mappedStatements = new StrictMap<>("Mapped Statements collection");
  
  public void addMappedStatement(MappedStatement ms) {
      mappedStatements.put(ms.getId(), ms);
  }
  ```

###### 4.1.1 MappedStatement

`org.apache.ibatis.mapping.MappedStatement` ，映射的语句，每个 `<select />`、`<insert />`、`<update />`、`<delete />` 对应一个 MappedStatement 对象。代码比较简单，但是有点略长，胖友直接点击 [链接](https://github.com/YunaiV/mybatis-3/blob/master/src/main/java/org/apache/ibatis/mapping/MappedStatement.java) 查看，已经添加了完整的注释。

另外，比较特殊的是，`<selectKey />` 解析后，也会对应一个 MappedStatement 对象。

在另外，MappedStatement 有一个非常重要的方法 `#getBoundSql(Object parameterObject)` 方法，获得 BoundSql 对象。代码如下：

```
// MappedStatement.java

public BoundSql getBoundSql(Object parameterObject) {
    // 获得 BoundSql 对象
    BoundSql boundSql = sqlSource.getBoundSql(parameterObject);
    // 忽略，因为 <parameterMap /> 已经废弃，参见 http://www.mybatis.org/mybatis-3/zh/sqlmap-xml.html 文档
    List<ParameterMapping> parameterMappings = boundSql.getParameterMappings();
    if (parameterMappings == null || parameterMappings.isEmpty()) {
        boundSql = new BoundSql(configuration, boundSql.getSql(), parameterMap.getParameterMappings(), parameterObject);
    }

    // check for nested result maps in parameter mappings (issue #30)
    // 判断传入的参数中，是否有内嵌的结果 ResultMap 。如果有，则修改 hasNestedResultMaps 为 true
    // 存储过程相关，暂时无视
    for (ParameterMapping pm : boundSql.getParameterMappings()) {
        String rmId = pm.getResultMapId();
        if (rmId != null) {
            ResultMap rm = configuration.getResultMap(rmId);
            if (rm != null) {
                hasNestedResultMaps |= rm.hasNestedResultMaps();
            }
        }
    }

    return boundSql;
}
```

- 需要结合后续文章 [《精尽 MyBatis 源码分析 —— SQL 初始化（下）之 SqlSource》](http://svip.iocoder.cn/MyBatis/scripting-2) 。胖友可以看完那篇文章后，再回过头看这个方法。

###### 4.1.2 ParameterMap

`org.apache.ibatis.mapping.ParameterMap` ，参数集合，对应 `paramType=""` 或 `paramMap=""` 标签属性。代码比较简单，但是有点略长，胖友直接点击 [链接](https://github.com/YunaiV/mybatis-3/blob/master/src/main/java/org/apache/ibatis/mapping/ParameterMap.java) 查看，已经添加了完整的注释。

###### 4.1.3 getStatementResultMaps

`#getStatementResultMaps(...)` 方法，获得 ResultMap 集合。代码如下：

```
// MapperBuilderAssistant.java

private List<ResultMap> getStatementResultMaps(
        String resultMap,
        Class<?> resultType,
        String statementId) {
    // 获得 resultMap 的编号
    resultMap = applyCurrentNamespace(resultMap, true);

    // 创建 ResultMap 集合
    List<ResultMap> resultMaps = new ArrayList<>();
    // 如果 resultMap 非空，则获得 resultMap 对应的 ResultMap 对象(们）
    if (resultMap != null) {
        String[] resultMapNames = resultMap.split(",");
        for (String resultMapName : resultMapNames) {
            try {
                resultMaps.add(configuration.getResultMap(resultMapName.trim())); // 从 configuration 中获得
            } catch (IllegalArgumentException e) {
                throw new IncompleteElementException("Could not find result map " + resultMapName, e);
            }
        }
    // 如果 resultType 非空，则创建 ResultMap 对象
    } else if (resultType != null) {
        ResultMap inlineResultMap = new ResultMap.Builder(
                configuration,
                statementId + "-Inline",
                resultType,
                new ArrayList<>(),
                null).build();
        resultMaps.add(inlineResultMap);
    }
    return resultMaps;
}
```

- 整体代码比较简单，胖友自己看下。
- 比较奇怪的是，方法参数 `resultMap` 存在使用逗号分隔的情况。这个出现在使用存储过程的时候，参见 [《mybatis调用存储过程返回多个结果集》](https://blog.csdn.net/sinat_25295611/article/details/75103358) 。

###### 4.1.4 getStatementResultMaps

`#getStatementParameterMap(...)` 方法，获得 ParameterMap 对象。代码如下：

```
// MapperBuilderAssistant.java

private ParameterMap getStatementParameterMap(
        String parameterMapName,
        Class<?> parameterTypeClass,
        String statementId) {
    // 获得 ParameterMap 的编号，格式为 `${namespace}.${parameterMapName}`
    parameterMapName = applyCurrentNamespace(parameterMapName, true);
    ParameterMap parameterMap = null;
    // <2> 如果 parameterMapName 非空，则获得 parameterMapName 对应的 ParameterMap 对象
    if (parameterMapName != null) {
        try {
            parameterMap = configuration.getParameterMap(parameterMapName);
        } catch (IllegalArgumentException e) {
            throw new IncompleteElementException("Could not find parameter map " + parameterMapName, e);
        }
    // <1> 如果 parameterTypeClass 非空，则创建 ParameterMap 对象
    } else if (parameterTypeClass != null) {
        List<ParameterMapping> parameterMappings = new ArrayList<>();
        parameterMap = new ParameterMap.Builder(
                configuration,
                statementId + "-Inline",
                parameterTypeClass,
                parameterMappings).build();
    }
    return parameterMap;
}
```

- 主要看 `<1>` 处，如果 `parameterTypeClass` 非空，则创建 ParameterMap 对象。
- 关于 `<2>` 处，MyBatis 官方不建议使用 `parameterMap` 的方式。

# 4、加载注解配置

## 1. 概述

本文接 [《精尽 MyBatis 源码分析 —— MyBatis 初始化（三）之加载 Statement 配置》](http://svip.iocoder.cn/MyBatis/builder-package-3) 一文，来分享 MyBatis 初始化的第 2 步的一部分，**加载注解配置**。而这个部分的入口是 MapperAnnotationBuilder 。下面，我们一起来看看它的代码实现。

在 [《精尽 MyBatis 源码分析 —— MyBatis 初始化（二）之加载 Mapper 映射配置文件》](http://svip.iocoder.cn/MyBatis/builder-package-2) 的 [「3.3.13 mapperElement」](http://svip.iocoder.cn/MyBatis/builder-package-4/#) 中，我们已经看到对 MapperAnnotationBuilder 的调用代码。代码如下：

```
// Configuration.java
    
public <T> void addMapper(Class<T> type) {
    mapperRegistry.addMapper(type);
}

// MapperRegistry.java

public <T> void addMapper(Class<T> type) {
    // 判断，必须是接口。
    if (type.isInterface()) {
        // 已经添加过，则抛出 BindingException 异常
        if (hasMapper(type)) {
            throw new BindingException("Type " + type + " is already known to the MapperRegistry.");
        }
        boolean loadCompleted = false;
        try {
            // 添加到 knownMappers 中
            knownMappers.put(type, new MapperProxyFactory<>(type));
            // It's important that the type is added before the parser is run
            // otherwise the binding may automatically be attempted by the
            // mapper parser. If the type is already known, it won't try.
            // 解析 Mapper 的注解配置 <====== 😈 看我 😈 =====>
            MapperAnnotationBuilder parser = new MapperAnnotationBuilder(config, type);
            parser.parse();
            // 标记加载完成
            loadCompleted = true;
        } finally {
            // 若加载未完成，从 knownMappers 中移除
            if (!loadCompleted) {
                knownMappers.remove(type);
            }
        }
    }
}
```

- 调用处，请看 😈 处。

## 2. MapperAnnotationBuilder

`org.apache.ibatis.builder.annotation.MapperAnnotationBuilder` ，Mapper 注解构造器，负责解析 Mapper 接口上的注解。

#### 2.1 构造方法

```
// MapperAnnotationBuilder.java

/**
 * SQL 操作注解集合
 */
private static final Set<Class<? extends Annotation>> SQL_ANNOTATION_TYPES = new HashSet<>();
/**
 * SQL 操作提供者注解集合
 */
private static final Set<Class<? extends Annotation>> SQL_PROVIDER_ANNOTATION_TYPES = new HashSet<>();

private final Configuration configuration;
private final MapperBuilderAssistant assistant;
/**
 * Mapper 接口类
 */
private final Class<?> type;

static {
    SQL_ANNOTATION_TYPES.add(Select.class);
    SQL_ANNOTATION_TYPES.add(Insert.class);
    SQL_ANNOTATION_TYPES.add(Update.class);
    SQL_ANNOTATION_TYPES.add(Delete.class);

    SQL_PROVIDER_ANNOTATION_TYPES.add(SelectProvider.class);
    SQL_PROVIDER_ANNOTATION_TYPES.add(InsertProvider.class);
    SQL_PROVIDER_ANNOTATION_TYPES.add(UpdateProvider.class);
    SQL_PROVIDER_ANNOTATION_TYPES.add(DeleteProvider.class);
}

public MapperAnnotationBuilder(Configuration configuration, Class<?> type) {
    // 创建 MapperBuilderAssistant 对象
    String resource = type.getName().replace('.', '/') + ".java (best guess)";
    this.assistant = new MapperBuilderAssistant(configuration, resource);
    this.configuration = configuration;
    this.type = type;
}
```

- 比较简单，胖友扫一眼。

#### 2.2 parse

`#parse()` 方法，解析注解。代码如下：

```
// MapperAnnotationBuilder.java

public void parse() {
    // <1> 判断当前 Mapper 接口是否应加载过。
    String resource = type.toString();
    if (!configuration.isResourceLoaded(resource)) {
        // <2> 加载对应的 XML Mapper
        loadXmlResource();
        // <3> 标记该 Mapper 接口已经加载过
        configuration.addLoadedResource(resource);
        // <4> 设置 namespace 属性
        assistant.setCurrentNamespace(type.getName());
        // <5> 解析 @CacheNamespace 注解
        parseCache();
        // <6> 解析 @CacheNamespaceRef 注解
        parseCacheRef();
        // <7> 遍历每个方法，解析其上的注解
        Method[] methods = type.getMethods();
        for (Method method : methods) {
            try {
                // issue #237
                if (!method.isBridge()) {
                    // <7.1> 执行解析
                    parseStatement(method);
                }
            } catch (IncompleteElementException e) {
                // <7.2> 解析失败，添加到 configuration 中
                configuration.addIncompleteMethod(new MethodResolver(this, method));
            }
        }
    }
    // <8> 解析待定的方法
    parsePendingMethods();
}
```

- `<1>` 处，调用 `Configuration#isResourceLoaded(String resource)` 方法，判断当前 Mapper 接口是否应加载过。

- `<2>` 处，调用 `#loadXmlResource()` 方法，加载对应的 XML Mapper 。详细解析，见 [「2.3 loadXmlResource」](http://svip.iocoder.cn/MyBatis/builder-package-4/#) 。

- `<3>` 处，调用 `Configuration#addLoadedResource(String resource)` 方法，标记该 Mapper 接口已经加载过。

- `<4>` 处，调用 `MapperBuilderAssistant#setCurrentNamespace(String currentNamespace)` 方法，设置 `namespace` 属性。

- `<5>` 处，调用 `#parseCache()` 方法，`@CacheNamespace` 注解。详细解析，见 [「2.4 parseCache」](http://svip.iocoder.cn/MyBatis/builder-package-4/#) 。

- `<6>` 处，调用 `#parseCacheRef()` 方法，`@CacheNamespaceRef` 注解。详细解析，见 [「2.5 parseCacheRef」](http://svip.iocoder.cn/MyBatis/builder-package-4/#) 。

- `<7>` 处，遍历每个方法，解析其上的注解。

  - `<7.1>` 处，调用 `#parseStatement(Method method)` 方法，执行解析每个方法的注解。详细解析，见 [「2.6 parseStatement」](http://svip.iocoder.cn/MyBatis/builder-package-4/#) 。

  - `<7.2>` 处，解析失败，调用 `Configuration#addIncompleteMethod(MethodResolver builder)` 方法，添加到 `configuration` 中。代码如下：

    ```
    // Configuration.java
    
    /**
     * 未完成的 MethodResolver 集合
     */
    protected final Collection<MethodResolver> incompleteMethods = new LinkedList<>();
    
    public void addIncompleteMethod(MethodResolver builder) {
        incompleteMethods.add(builder);
    }
    ```

    - 关于 MethodResolver 类，详细解析，见 [「2.7 MethodResolver」](http://svip.iocoder.cn/MyBatis/builder-package-4/#) 。

- `<8>` 处，调用 `#parsePendingMethods()` 方法，解析待定的方法。详细解析，见 [「2.8 parsePendingMethods」](http://svip.iocoder.cn/MyBatis/builder-package-4/#) 。

#### 2.3 loadXmlResource

`#loadXmlResource()` 方法，加载对应的 XML Mapper 。代码如下：

```
// MapperAnnotationBuilder.java

private void loadXmlResource() {
    // Spring may not know the real resource name so we check a flag
    // to prevent loading again a resource twice
    // this flag is set at XMLMapperBuilder#bindMapperForNamespace
    // <1> 判断 Mapper XML 是否已经加载过，如果加载过，就不加载了。
    // 此处，是为了避免和 XMLMapperBuilder#parse() 方法冲突，重复解析
    if (!configuration.isResourceLoaded("namespace:" + type.getName())) {
        // <2> 获得 InputStream 对象
        String xmlResource = type.getName().replace('.', '/') + ".xml";
        // #1347
        InputStream inputStream = type.getResourceAsStream("/" + xmlResource);
        if (inputStream == null) {
            // Search XML mapper that is not in the module but in the classpath.
            try {
                inputStream = Resources.getResourceAsStream(type.getClassLoader(), xmlResource);
            } catch (IOException e2) {
                // ignore, resource is not required
            }
        }
        // <2> 创建 XMLMapperBuilder 对象，执行解析
        if (inputStream != null) {
            XMLMapperBuilder xmlParser = new XMLMapperBuilder(inputStream, assistant.getConfiguration(), xmlResource, configuration.getSqlFragments(), type.getName());
            xmlParser.parse();
        }
    }
}
```

- `<1>` 处，判断 Mapper XML 是否已经加载过，如果加载过，就不加载了。此处，是为了避免和 `XMLMapperBuilder#parse()` 方法冲突，重复解析。
- `<2>` 处，获得 InputStream 对象，然后创建 XMLMapperBuilder 对象，最后调用 `XMLMapperBuilder#parse()` 方法，执行解析。
- 这里，如果是先解析 Mapper 接口，那就会达到再解析对应的 Mapper XML 的情况。

#### 2.4 parseCache

`#parseCache()` 方法，解析 `@CacheNamespace` 注解。代码如下：

```
// MapperAnnotationBuilder.java

private void parseCache() {
    // <1> 获得类上的 @CacheNamespace 注解
    CacheNamespace cacheDomain = type.getAnnotation(CacheNamespace.class);
    if (cacheDomain != null) {
        // <2> 获得各种属性
        Integer size = cacheDomain.size() == 0 ? null : cacheDomain.size();
        Long flushInterval = cacheDomain.flushInterval() == 0 ? null : cacheDomain.flushInterval();
        // <3> 获得 Properties 属性
        Properties props = convertToProperties(cacheDomain.properties());
        // <4> 创建 Cache 对象
        assistant.useNewCache(cacheDomain.implementation(), cacheDomain.eviction(), flushInterval, size, cacheDomain.readWrite(), cacheDomain.blocking(), props);
    }
}
```

- `<1>` 处，获得**类**上的 `@CacheNamespace` 注解。

- `<2>` 处，获得各种属性。

- `<3>` 处，调用 `#convertToProperties(Property[] properties)` 方法，将 `@Property` 注解数组，转换成 Properties 对象。代码如下：

  ```
  // MapperAnnotationBuilder.java
  
  private Properties convertToProperties(Property[] properties) {
      if (properties.length == 0) {
          return null;
      }
      Properties props = new Properties();
      for (Property property : properties) {
          props.setProperty(property.name(),
                  PropertyParser.parse(property.value(), configuration.getVariables())); // 替换
      }
      return props;
  }
  ```

- `<4>` 处，调用 `MapperBuilderAssistant#useNewCache(...)` 方法，创建 Cache 对象。在 [《精尽 MyBatis 源码分析 —— MyBatis 初始化（二）之加载 Mapper 映射配置文件》](http://svip.iocoder.cn/MyBatis/builder-package-2) 的 [「3.4 useNewCache」](http://svip.iocoder.cn/MyBatis/builder-package-4/#) 中，已经详细解析。

#### 2.5 parseCacheRef

`#parseCacheRef()` 方法，解析 `@CacheNamespaceRef` 注解。代码如下：

```
// MapperAnnotationBuilder.java

private void parseCacheRef() {
    // 获得类上的 @CacheNamespaceRef 注解
    CacheNamespaceRef cacheDomainRef = type.getAnnotation(CacheNamespaceRef.class);
    if (cacheDomainRef != null) {
        // <2> 获得各种属性
        Class<?> refType = cacheDomainRef.value();
        String refName = cacheDomainRef.name();
        // <2> 校验，如果 refType 和 refName 都为空，则抛出 BuilderException 异常
        if (refType == void.class && refName.isEmpty()) {
            throw new BuilderException("Should be specified either value() or name() attribute in the @CacheNamespaceRef");
        }
        // <2> 校验，如果 refType 和 refName 都不为空，则抛出 BuilderException 异常
        if (refType != void.class && !refName.isEmpty()) {
            throw new BuilderException("Cannot use both value() and name() attribute in the @CacheNamespaceRef");
        }
        // <2> 获得最终的 namespace 属性
        String namespace = (refType != void.class) ? refType.getName() : refName;
        // <3> 获得指向的 Cache 对象
        assistant.useCacheRef(namespace);
    }
}
```

- `<1>` 处，获得**类**上的 `@CacheNamespaceRef` 注解。
- `<2>` 处，获得各种属性，进行校验，最终获得 `namespace` 属性。
- `<3>` 处，调用 `MapperBuilderAssistant#useCacheRef(String namespace)` 方法，获得指向的 Cache 对象。在 [《精尽 MyBatis 源码分析 —— MyBatis 初始化（二）之加载 Mapper 映射配置文件》](http://svip.iocoder.cn/MyBatis/builder-package-2) 的 [「3.3 useCacheRef」](http://svip.iocoder.cn/MyBatis/builder-package-4/#) 中，已经详细解析。

#### 2.6 parseStatement

`#parseStatement(Method method)` 方法，解析方法上的 SQL 操作相关的注解。代码如下：

```
// MapperAnnotationBuilder.java

void parseStatement(Method method) {
    // <1> 获得参数的类型
    Class<?> parameterTypeClass = getParameterType(method);
    // <2> 获得 LanguageDriver 对象
    LanguageDriver languageDriver = getLanguageDriver(method);
    // <3> 获得 SqlSource 对象
    SqlSource sqlSource = getSqlSourceFromAnnotations(method, parameterTypeClass, languageDriver);
    if (sqlSource != null) {
        // <4> 获得各种属性
        Options options = method.getAnnotation(Options.class);
        final String mappedStatementId = type.getName() + "." + method.getName();
        Integer fetchSize = null;
        Integer timeout = null;
        StatementType statementType = StatementType.PREPARED;
        ResultSetType resultSetType = null;
        SqlCommandType sqlCommandType = getSqlCommandType(method);
        boolean isSelect = sqlCommandType == SqlCommandType.SELECT;
        boolean flushCache = !isSelect;
        boolean useCache = isSelect;

        // <5> 获得 KeyGenerator 对象
        KeyGenerator keyGenerator;
        String keyProperty = null;
        String keyColumn = null;
        if (SqlCommandType.INSERT.equals(sqlCommandType) || SqlCommandType.UPDATE.equals(sqlCommandType)) { // 有
            // first check for SelectKey annotation - that overrides everything else
            // <5.1> 如果有 @SelectKey 注解，则进行处理
            SelectKey selectKey = method.getAnnotation(SelectKey.class);
            if (selectKey != null) {
                keyGenerator = handleSelectKeyAnnotation(selectKey, mappedStatementId, getParameterType(method), languageDriver);
                keyProperty = selectKey.keyProperty();
            // <5.2> 如果无 @Options 注解，则根据全局配置处理
            } else if (options == null) {
                keyGenerator = configuration.isUseGeneratedKeys() ? Jdbc3KeyGenerator.INSTANCE : NoKeyGenerator.INSTANCE;
            // <5.3> 如果有 @Options 注解，则使用该注解的配置处理
            } else {
                keyGenerator = options.useGeneratedKeys() ? Jdbc3KeyGenerator.INSTANCE : NoKeyGenerator.INSTANCE;
                keyProperty = options.keyProperty();
                keyColumn = options.keyColumn();
            }
        // <5.4> 无
        } else {
            keyGenerator = NoKeyGenerator.INSTANCE;
        }

        // <6> 初始化各种属性
        if (options != null) {
            if (FlushCachePolicy.TRUE.equals(options.flushCache())) {
                flushCache = true;
            } else if (FlushCachePolicy.FALSE.equals(options.flushCache())) {
                flushCache = false;
            }
            useCache = options.useCache();
            fetchSize = options.fetchSize() > -1 || options.fetchSize() == Integer.MIN_VALUE ? options.fetchSize() : null; //issue #348
            timeout = options.timeout() > -1 ? options.timeout() : null;
            statementType = options.statementType();
            resultSetType = options.resultSetType();
        }

        // <7> 获得 resultMapId 编号字符串
        String resultMapId = null;
        // <7.1> 如果有 @ResultMap 注解，使用该注解为 resultMapId 属性
        ResultMap resultMapAnnotation = method.getAnnotation(ResultMap.class);
        if (resultMapAnnotation != null) {
            String[] resultMaps = resultMapAnnotation.value();
            StringBuilder sb = new StringBuilder();
            for (String resultMap : resultMaps) {
                if (sb.length() > 0) {
                    sb.append(",");
                }
                sb.append(resultMap);
            }
            resultMapId = sb.toString();
        // <7.2> 如果无 @ResultMap 注解，解析其它注解，作为 resultMapId 属性
        } else if (isSelect) {
            resultMapId = parseResultMap(method);
        }

        // 构建 MappedStatement 对象
        assistant.addMappedStatement(
                mappedStatementId,
                sqlSource,
                statementType,
                sqlCommandType,
                fetchSize,
                timeout,
                // ParameterMapID
                null,
                parameterTypeClass,
                resultMapId,
                getReturnType(method), // 获得返回类型
                resultSetType,
                flushCache,
                useCache,
                // TODO gcode issue #577
                false,
                keyGenerator,
                keyProperty,
                keyColumn,
                // DatabaseID
                null,
                languageDriver,
                // ResultSets
                options != null ? nullOrEmpty(options.resultSets()) : null);
    }
}
```

- `<1>` 处，调用 `#getParameterType(Method method)` 方法，获得参数的类型。详细解析，见 [「2.6.1 getParameterType」](http://svip.iocoder.cn/MyBatis/builder-package-4/#) 。

- `<2>` 处，调用 `#getLanguageDriver(Method method)` 方法，获得 LanguageDriver 对象。详细解析，见 [「2.6.2 getLanguageDriver」](http://svip.iocoder.cn/MyBatis/builder-package-4/#) 。

- `<3>` 处，调用 `#getSqlSourceFromAnnotations(...)` 方法，从注解中，获得 SqlSource 对象。详细解析，见 [「2.6.3 getSqlSourceFromAnnotations」](http://svip.iocoder.cn/MyBatis/builder-package-4/#) 。

- ```
  <4>
  ```

   

  处，获得各种属性。

  - 

- ```
  <5>
  ```

   

  处，获得 KeyGenerator 对象。

  - `<5.1>` 处，如果有 `@SelectKey` 注解，则调用 `#handleSelectKeyAnnotation(...)` 方法，处理 `@@SelectKey` 注解，生成对应的 SelectKey 对象。详细解析，见 [「2.6.4 handleSelectKeyAnnotation」](http://svip.iocoder.cn/MyBatis/builder-package-4/#) 。
  - `<5.2>` 处，如果无 `@Options` 注解，则根据全局配置处理，使用 Jdbc3KeyGenerator 或 NoKeyGenerator 单例。
  - `<5.3>` 处，如果有 `@Options` 注解，则使用该注解的配置处理。
  - `<5.4>` 处，非插入和更新语句，无需 KeyGenerator 对象，所以使用 NoKeyGenerator 单例。

- `<6>` 处，初始化各种属性。

- ```
  <7>
  ```

   

  处，获得

   

  ```
  resultMapId
  ```

   

  编号字符串。

  - `<7.1>` 处，如果有 `@ResultMap` 注解，使用该注解为 `resultMapId` 属性。因为 `@ResultMap` 注解的作用，就是注解使用的结果集。
  - `<7.2>` 处，如果无 `@ResultMap` 注解，调用 `#parseResultMap(Method method)` 方法，解析其它注解，作为 `resultMapId` 属性。详细解析，见 [「2.6.6 parseResultMap」](http://svip.iocoder.cn/MyBatis/builder-package-4/#) 。

###### 2.6.1 getParameterType

调用 `#getParameterType(Method method)` 方法，获得参数的类型。代码如下：

```
// MapperAnnotationBuilder.java

private Class<?> getParameterType(Method method) {
    Class<?> parameterType = null;
    // 遍历参数类型数组
    // 排除 RowBounds 和 ResultHandler 两种参数
    // 1. 如果是多参数，则是 ParamMap 类型
    // 2. 如果是单参数，则是该参数的类型
    Class<?>[] parameterTypes = method.getParameterTypes();
    for (Class<?> currentParameterType : parameterTypes) {
        if (!RowBounds.class.isAssignableFrom(currentParameterType) && !ResultHandler.class.isAssignableFrom(currentParameterType)) {
            if (parameterType == null) {
                parameterType = currentParameterType;
            } else {
                // issue #135
                parameterType = ParamMap.class;
            }
        }
    }
    return parameterType;
}
```

- 比较简单，根据是否为多参数，返回是 ParamMap 类型，还是单参数对应的类型。

###### 2.6.2 getLanguageDriver

`#getLanguageDriver(Method method)` 方法，获得 LanguageDriver 对象。代码如下：

```
// MapperAnnotationBuilder.java

private LanguageDriver getLanguageDriver(Method method) {
    // 解析 @Lang 注解，获得对应的类型
    Lang lang = method.getAnnotation(Lang.class);
    Class<? extends LanguageDriver> langClass = null;
    if (lang != null) {
        langClass = lang.value();
    }
    // 获得 LanguageDriver 对象
    // 如果 langClass 为空，即无 @Lang 注解，则会使用默认 LanguageDriver 类型
    return assistant.getLanguageDriver(langClass);
}
```

- 调用 `MapperBuilderAssistant#getLanguageDriver(Class<? extends LanguageDriver> langClass)` 方法，获得 LanguageDriver 对象。在 [《精尽 MyBatis 源码分析 —— MyBatis 初始化（三）之加载 Statement 配置》](http://svip.iocoder.cn/MyBatis/builder-package-3) 的 [「2.4 getLanguageDriver」](http://svip.iocoder.cn/MyBatis/builder-package-4/#) 中，已经详细解析。

###### 2.6.3 getSqlSourceFromAnnotations

`#getSqlSourceFromAnnotations(Method method, Class<?> parameterType, LanguageDriver languageDriver)` 方法，从注解中，获得 SqlSource 对象。代码如下：

```
// MapperAnnotationBuilder.java

private SqlSource getSqlSourceFromAnnotations(Method method, Class<?> parameterType, LanguageDriver languageDriver) {
    try {
        // <1.1> <1.2> 获得方法上的 SQL_ANNOTATION_TYPES 和 SQL_PROVIDER_ANNOTATION_TYPES 对应的类型
        Class<? extends Annotation> sqlAnnotationType = getSqlAnnotationType(method);
        Class<? extends Annotation> sqlProviderAnnotationType = getSqlProviderAnnotationType(method);
        // <2> 如果 SQL_ANNOTATION_TYPES 对应的类型非空
        if (sqlAnnotationType != null) {
            // 如果 SQL_PROVIDER_ANNOTATION_TYPES 对应的类型非空，则抛出 BindingException 异常，因为冲突了。
            if (sqlProviderAnnotationType != null) {
                throw new BindingException("You cannot supply both a static SQL and SqlProvider to method named " + method.getName());
            }
            // <2.1> 获得 SQL_ANNOTATION_TYPES 对应的注解
            Annotation sqlAnnotation = method.getAnnotation(sqlAnnotationType);
            // <2.2> 获得 value 属性
            final String[] strings = (String[]) sqlAnnotation.getClass().getMethod("value").invoke(sqlAnnotation);
            // <2.3> 创建 SqlSource 对象
            return buildSqlSourceFromStrings(strings, parameterType, languageDriver);
        // <3> 如果 SQL_PROVIDER_ANNOTATION_TYPES 对应的类型非空
        } else if (sqlProviderAnnotationType != null) {
            // <3.1> 获得 SQL_PROVIDER_ANNOTATION_TYPES 对应的注解
            Annotation sqlProviderAnnotation = method.getAnnotation(sqlProviderAnnotationType);
            // <3.2> 创建 ProviderSqlSource 对象
            return new ProviderSqlSource(assistant.getConfiguration(), sqlProviderAnnotation, type, method);
        }
        // <4> 返回空
        return null;
    } catch (Exception e) {
        throw new BuilderException("Could not find value method on SQL annotation.  Cause: " + e, e);
    }
}
```

- `<1.1>` 处，调用 `#getSqlAnnotationType(Method method)` 方法，获得方法上的 `SQL_ANNOTATION_TYPES` 类型的注解类型。详细解析，见 [「2.6.3.1 getSqlAnnotationType」](http://svip.iocoder.cn/MyBatis/builder-package-4/#) 。

- `<1.2>` 处，调用 `#getSqlProviderAnnotationType(Method method)` 方法，获得方法上的 `SQL_PROVIDER_ANNOTATION_TYPES` 类型的注解类型。详细解析，见 [「2.6.3.2 getSqlAnnotationType」](http://svip.iocoder.cn/MyBatis/builder-package-4/#) 。

- ```
  <2>
  ```

   

  处，如果

   

  ```
  SQL_ANNOTATION_TYPES
  ```

   

  对应的类型非空的情况，例如

   

  ```
  @Insert
  ```

   

  注解：

  - `<2.1>` 处，获得 `SQL_ANNOTATION_TYPES` 对应的注解。
  - `<2.2>` 处，通过反射，获得对应的 `value` 属性。因为这里的注解类有多种，所以只好通过反射方法来获取该属性。
  - `<2.3>` 处，调用 `#buildSqlSourceFromStrings(...)` 方法，创建 SqlSource 对象。详细解析，见 [「2.6.3.3 buildSqlSourceFromStrings」](http://svip.iocoder.cn/MyBatis/builder-package-4/#) 。

- ```
  <3>
  ```

   

  处，如果

   

  ```
  SQL_PROVIDER_ANNOTATION_TYPES
  ```

   

  对应的类型非空的情况，例如

   

  ```
  @InsertProvider
  ```

   

  注解：

  - `<3.1>` 处，获得 `SQL_PROVIDER_ANNOTATION_TYPES` 对应的注解。
  - `<3.2>` 处，创建 ProviderSqlSource 对象。详细解析，见后续的文章。

- `<4>` 处，没有空，没有符合的注解。

####### 2.6.3.1 getSqlAnnotationType

`#getSqlAnnotationType(Method method)` 方法，获得方法上的 `SQL_ANNOTATION_TYPES` 类型的注解类型。代码如下：

```
// MapperAnnotationBuilder.java

private Class<? extends Annotation> getSqlAnnotationType(Method method) {
    return chooseAnnotationType(method, SQL_ANNOTATION_TYPES);
}

/**
 * 获得符合指定类型的注解类型
 *
 * @param method 方法
 * @param types 指定类型
 * @return 查到的注解类型
 */
private Class<? extends Annotation> chooseAnnotationType(Method method, Set<Class<? extends Annotation>> types) {
    for (Class<? extends Annotation> type : types) {
        Annotation annotation = method.getAnnotation(type);
        if (annotation != null) {
            return type;
        }
    }
    return null;
}
```

####### 2.6.3.2 getSqlProviderAnnotationType

`#getSqlProviderAnnotationType(Method method)` 方法，获得方法上的 `SQL_ANNOTATION_TYPES` 类型的注解类型。代码如下：

```
// MapperAnnotationBuilder.java

private Class<? extends Annotation> getSqlProviderAnnotationType(Method method) {
    return chooseAnnotationType(method, SQL_PROVIDER_ANNOTATION_TYPES);
}
```

####### 2.6.3.3 buildSqlSourceFromStrings

`#buildSqlSourceFromStrings(String[] strings, Class<?> parameterTypeClass, LanguageDriver languageDriver)` 方法，创建 SqlSource 对象。代码如下：

```
// MapperAnnotationBuilder.java

private SqlSource buildSqlSourceFromStrings(String[] strings, Class<?> parameterTypeClass, LanguageDriver languageDriver) {
    // <1> 拼接 SQL
    final StringBuilder sql = new StringBuilder();
    for (String fragment : strings) {
        sql.append(fragment);
        sql.append(" ");
    }
    // <2> 创建 SqlSource 对象
    return languageDriver.createSqlSource(configuration, sql.toString().trim(), parameterTypeClass);
}
```

- `<1>` 处，拼接 SQL ，使用 `" "` 空格分隔。因为，`@Select` 等注解，`value` 属性是个**数组**。
- `<2>` 处，调用 `LanguageDriver#createSqlSource(Configuration configuration, String script, Class<?> parameterType)` 方法，创建 SqlSource 对象。详细解析，见后续文章。

###### 2.6.4 handleSelectKeyAnnotation

`#handleSelectKeyAnnotation(SelectKey selectKeyAnnotation, String baseStatementId, Class<?> parameterTypeClass, LanguageDriver languageDriver)` 方法，处理 `@@SelectKey` 注解，生成对应的 SelectKey 对象。代码如下：

```
// MapperAnnotationBuilder.java

private KeyGenerator handleSelectKeyAnnotation(SelectKey selectKeyAnnotation, String baseStatementId, Class<?> parameterTypeClass, LanguageDriver languageDriver) {
    // 获得各种属性和对应的类
    String id = baseStatementId + SelectKeyGenerator.SELECT_KEY_SUFFIX;
    Class<?> resultTypeClass = selectKeyAnnotation.resultType();
    StatementType statementType = selectKeyAnnotation.statementType();
    String keyProperty = selectKeyAnnotation.keyProperty();
    String keyColumn = selectKeyAnnotation.keyColumn();
    boolean executeBefore = selectKeyAnnotation.before();

    // defaults
    // 创建 MappedStatement 需要用到的默认值
    boolean useCache = false;
    KeyGenerator keyGenerator = NoKeyGenerator.INSTANCE;
    Integer fetchSize = null;
    Integer timeout = null;
    boolean flushCache = false;
    String parameterMap = null;
    String resultMap = null;
    ResultSetType resultSetTypeEnum = null;

    // 创建 SqlSource 对象
    SqlSource sqlSource = buildSqlSourceFromStrings(selectKeyAnnotation.statement(), parameterTypeClass, languageDriver);
    SqlCommandType sqlCommandType = SqlCommandType.SELECT;

    // 创建 MappedStatement 对象
    assistant.addMappedStatement(id, sqlSource, statementType, sqlCommandType, fetchSize, timeout, parameterMap, parameterTypeClass, resultMap, resultTypeClass, resultSetTypeEnum,
            flushCache, useCache, false,
            keyGenerator, keyProperty, keyColumn, null, languageDriver, null);

    // 获得 SelectKeyGenerator 的编号，格式为 `${namespace}.${id}`
    id = assistant.applyCurrentNamespace(id, false);
    // 获得 MappedStatement 对象
    MappedStatement keyStatement = configuration.getMappedStatement(id, false);
    // 创建 SelectKeyGenerator 对象，并添加到 configuration 中
    SelectKeyGenerator answer = new SelectKeyGenerator(keyStatement, executeBefore);
    configuration.addKeyGenerator(id, answer);
    return answer;
}
```

- 从实现逻辑上，我们会发现，和 `XMLStatementBuilder#parseSelectKeyNode(String id, XNode nodeToHandle, Class<?> parameterTypeClass, LanguageDriver langDriver, String databaseId)` 方法是**一致**的。所以就不重复解析了，胖友可以看看 [《精尽 MyBatis 源码分析 —— MyBatis 初始化（三）之加载 Statement 配置》](http://svip.iocoder.cn/MyBatis/builder-package-3) 的 [「2.5.2 parseSelectKeyNode」](http://svip.iocoder.cn/MyBatis/builder-package-4/#) 。

###### 2.6.5 getSqlCommandType

`#getSqlCommandType(Method method)` 方法，获得方法对应的 SQL 命令类型。代码如下：

```
// MapperAnnotationBuilder.java

private SqlCommandType getSqlCommandType(Method method) {
    // 获得符合 SQL_ANNOTATION_TYPES 类型的注解类型
    Class<? extends Annotation> type = getSqlAnnotationType(method);

    if (type == null) {
        // 获得符合 SQL_PROVIDER_ANNOTATION_TYPES 类型的注解类型
        type = getSqlProviderAnnotationType(method);

        // 找不到，返回 SqlCommandType.UNKNOWN
        if (type == null) {
            return SqlCommandType.UNKNOWN;
        }

        // 转换成对应的枚举
        if (type == SelectProvider.class) {
            type = Select.class;
        } else if (type == InsertProvider.class) {
            type = Insert.class;
        } else if (type == UpdateProvider.class) {
            type = Update.class;
        } else if (type == DeleteProvider.class) {
            type = Delete.class;
        }
    }

    // 转换成对应的枚举
    return SqlCommandType.valueOf(type.getSimpleName().toUpperCase(Locale.ENGLISH));
}
```

###### 2.6.6 parseResultMap

`#parseResultMap(Method method)` 方法，解析其它注解，返回 `resultMapId` 属性。代码如下：

```
// MapperAnnotationBuilder.java

private String parseResultMap(Method method) {
    // <1> 获得返回类型
    Class<?> returnType = getReturnType(method);
    // <2> 获得 @ConstructorArgs、@Results、@TypeDiscriminator 注解
    ConstructorArgs args = method.getAnnotation(ConstructorArgs.class);
    Results results = method.getAnnotation(Results.class);
    TypeDiscriminator typeDiscriminator = method.getAnnotation(TypeDiscriminator.class);
    // <3> 生成 resultMapId
    String resultMapId = generateResultMapName(method);
    // <4> 生成 ResultMap 对象
    applyResultMap(resultMapId, returnType, argsIf(args), resultsIf(results), typeDiscriminator);
    return resultMapId;
}
```

- `<1>` 处，调用 `#getReturnType(Method method)` 方法，获得返回类型。详细解析，见 [「2.6.6.1 getReturnType」](http://svip.iocoder.cn/MyBatis/builder-package-4/#) 。

- `<2>` 处，获得 `@ConstructorArgs`、`@Results`、`@TypeDiscriminator` 注解。

- `<3>` 处，调用 `#generateResultMapName(Method method)` 方法，生成 `resultMapId` 。详细解析，见 [「2.6.6.2 generateResultMapName」](http://svip.iocoder.cn/MyBatis/builder-package-4/#) 。

- `<4>` 处，调用 `#argsIf(ConstructorArgs args)` 方法，获得 `@ConstructorArgs` 注解的 `@Arg[]` 数组。代码如下：

  ```
  // MapperAnnotationBuilder.java
  
  private Arg[] argsIf(ConstructorArgs args) {
      return args == null ? new Arg[0] : args.value();
  }
  ```

- `<4>` 处，调用 `#resultsIf((Results results)` 方法，获得 `@(Results` 注解的 `@Result[]` 数组。

  ```
  // MapperAnnotationBuilder.java
  
  private Result[] resultsIf(Results results) {
      return results == null ? new Result[0] : results.value();
  }
  ```

- `<4>` 处，调用 `#applyResultMap(...)` 方法，生成 ResultMap 对象。详细解析，见 [「2.6.6.3 applyResultMap」](http://svip.iocoder.cn/MyBatis/builder-package-4/#) 。

####### 2.6.6.1 getReturnType

`#getReturnType(Method method)` 方法，获得返回类型。代码如下：

```
// MapperAnnotationBuilder.java

private Class<?> getReturnType(Method method) {
    // 获得方法的返回类型
    Class<?> returnType = method.getReturnType();
    // 解析成对应的 Type
    Type resolvedReturnType = TypeParameterResolver.resolveReturnType(method, type);
    // 如果 Type 是 Class ，普通类
    if (resolvedReturnType instanceof Class) {
        returnType = (Class<?>) resolvedReturnType;
        // 如果是数组类型，则使用 componentType
        if (returnType.isArray()) {
            returnType = returnType.getComponentType();
        }
        // gcode issue #508
        // 如果返回类型是 void ，则尝试使用 @ResultType 注解
        if (void.class.equals(returnType)) {
            ResultType rt = method.getAnnotation(ResultType.class);
            if (rt != null) {
                returnType = rt.value();
            }
        }
    // 如果 Type 是 ParameterizedType ，泛型
    } else if (resolvedReturnType instanceof ParameterizedType) {
        // 获得泛型 rawType
        ParameterizedType parameterizedType = (ParameterizedType) resolvedReturnType;
        Class<?> rawType = (Class<?>) parameterizedType.getRawType();
        // 如果是 Collection 或者 Cursor 类型时
        if (Collection.class.isAssignableFrom(rawType) || Cursor.class.isAssignableFrom(rawType)) {
            // 获得 <> 中实际类型
            Type[] actualTypeArguments = parameterizedType.getActualTypeArguments();
            // 如果 actualTypeArguments 的大小为 1 ，进一步处理
            if (actualTypeArguments != null && actualTypeArguments.length == 1) {
                Type returnTypeParameter = actualTypeArguments[0];
                // 如果是 Class ，则直接使用 Class
                if (returnTypeParameter instanceof Class<?>) {
                    returnType = (Class<?>) returnTypeParameter;
                // 如果是 ParameterizedType ，则获取 <> 中实际类型
                } else if (returnTypeParameter instanceof ParameterizedType) {
                    // (gcode issue #443) actual type can be a also a parameterized type
                    returnType = (Class<?>) ((ParameterizedType) returnTypeParameter).getRawType();
                // 如果是泛型数组类型，则获得 genericComponentType 对应的类
                } else if (returnTypeParameter instanceof GenericArrayType) {
                    Class<?> componentType = (Class<?>) ((GenericArrayType) returnTypeParameter).getGenericComponentType();
                    // (gcode issue #525) support List<byte[]>
                    returnType = Array.newInstance(componentType, 0).getClass();
                }
            }
        // 如果有 @MapKey 注解，并且是 Map 类型
        } else if (method.isAnnotationPresent(MapKey.class) && Map.class.isAssignableFrom(rawType)) {
            // (gcode issue 504) Do not look into Maps if there is not MapKey annotation
            // 获得 <> 中实际类型
            Type[] actualTypeArguments = parameterizedType.getActualTypeArguments();
            // 如果 actualTypeArguments 的大小为 2 ，进一步处理。
            // 为什么是 2 ，因为 Map<K, V> 呀，有 K、V 两个泛型
            if (actualTypeArguments != null && actualTypeArguments.length == 2) {
                // 处理 V 泛型
                Type returnTypeParameter = actualTypeArguments[1];
                // 如果 V 泛型为 Class ，则直接使用 Class
                if (returnTypeParameter instanceof Class<?>) {
                    returnType = (Class<?>) returnTypeParameter;
                // 如果 V 泛型为 ParameterizedType ，则获取 <> 中实际类型
                } else if (returnTypeParameter instanceof ParameterizedType) {
                    // (gcode issue 443) actual type can be a also a parameterized type
                    returnType = (Class<?>) ((ParameterizedType) returnTypeParameter).getRawType();
                }
            }
        // 如果是 Optional 类型时
        } else if (Optional.class.equals(rawType)) {
            // 获得 <> 中实际类型
            Type[] actualTypeArguments = parameterizedType.getActualTypeArguments();
            // 因为是 Optional<T> 类型，所以 actualTypeArguments 数组大小是一
            Type returnTypeParameter = actualTypeArguments[0];
            // 如果 <T> 泛型为 Class ，则直接使用 Class
            if (returnTypeParameter instanceof Class<?>) {
                returnType = (Class<?>) returnTypeParameter;
            }
        }
    }

    return returnType;
}
```

- 方法非常的长，主要是进一步处理方法的返回类型 `Method.returnType` ，例如是数组类型、泛型类型、有 `@MapKey` 注解，或是 Optional 类型，等等情况。
- 胖友大体看看就好，也不需要看的特别细。等回过头有需要，在详细看。

####### 2.6.6.2 generateResultMapName

`#generateResultMapName(Method method)` 方法，生成 `resultMapId` 。代码如下：

```
// MapperAnnotationBuilder.java

private String generateResultMapName(Method method) {
    // 第一种情况，已经声明
    // 如果有 @Results 注解，并且有设置 id 属性，则直接返回。格式为：`${type.name}.${Results.id}` 。
    Results results = method.getAnnotation(Results.class);
    if (results != null && !results.id().isEmpty()) {
        return type.getName() + "." + results.id();
    }
    // 第二种情况，自动生成
    // 获得 suffix 前缀，相当于方法参数构成的签名
    StringBuilder suffix = new StringBuilder();
    for (Class<?> c : method.getParameterTypes()) {
        suffix.append("-");
        suffix.append(c.getSimpleName());
    }
    if (suffix.length() < 1) {
        suffix.append("-void");
    }
    // 拼接返回。格式为 `${type.name}.${method.name}${suffix}` 。
    return type.getName() + "." + method.getName() + suffix;
}
```

- 分成两种情况：已经声明和自动生成。
- 代码比较简单，胖友自己瞅瞅。

####### 2.6.6.3 applyResultMap

`#applyResultMap(String resultMapId, Class<?> returnType, Arg[] args, Result[] results, TypeDiscriminator discriminator)` 方法，创建 ResultMap 对象。代码如下：

```
// MapperAnnotationBuilder.java

private void applyResultMap(String resultMapId, Class<?> returnType, Arg[] args, Result[] results, TypeDiscriminator discriminator) {
    // <1> 创建 ResultMapping 数组
    List<ResultMapping> resultMappings = new ArrayList<>();
    // <2> 将 @Arg[] 注解数组，解析成对应的 ResultMapping 对象们，并添加到 resultMappings 中。
    applyConstructorArgs(args, returnType, resultMappings);
    // <3> 将 @Result[] 注解数组，解析成对应的 ResultMapping 对象们，并添加到 resultMappings 中。
    applyResults(results, returnType, resultMappings);
    // <4> 创建 Discriminator 对象
    Discriminator disc = applyDiscriminator(resultMapId, returnType, discriminator);
    // TODO add AutoMappingBehaviour
    // <5> ResultMap 对象
    assistant.addResultMap(resultMapId, returnType, null, disc, resultMappings, null);
    // <6> 创建 Discriminator 的 ResultMap 对象们
    createDiscriminatorResultMaps(resultMapId, returnType, discriminator);
}
```

- `<1>` 处，创建 ResultMapping 数组。
- `<2>` 处，调用 `#applyConstructorArgs(...)` 方法，将 `@Arg[]` 注解数组，解析成对应的 ResultMapping 对象们，并添加到 `resultMappings` 中。详细解析，见 [「2.6.6.3.1 applyConstructorArgs」](http://svip.iocoder.cn/MyBatis/builder-package-4/#) 。
- `<3>` 处，调用 `#applyResults(...)` 方法，将 `@Result[]` 注解数组，解析成对应的 ResultMapping 对象们，并添加到 `resultMappings` 中。 详细解析，见 [「2.6.6.3.2 applyResults」](http://svip.iocoder.cn/MyBatis/builder-package-4/#) 。
- `<4>` 处，调用 `#applyDiscriminator(...)` 方法，创建 Discriminator 对象。详细解析，见 [「2.6.6.3.3 applyDiscriminator」](http://svip.iocoder.cn/MyBatis/builder-package-4/#) 。
- `<5>` 处，调用 `MapperAnnotationBuilder#addResultMap(...)` 方法，创建 ResultMap 对象。
- `<6>` 处，调用 `#createDiscriminatorResultMaps(...)` 方法，创建 Discriminator 的 ResultMap 对象们。详细解析，见 [「2.6.6.3.4 createDiscriminatorResultMaps」](http://svip.iocoder.cn/MyBatis/builder-package-4/#) 。
- 看完上述逻辑后，你会发现该方法，和 `XMLMapperBuilder#resultMapElement(XNode resultMapNode, List<ResultMapping> additionalResultMappings)` 方法是类似的。所以，胖友后续可以自己去回味下。

######## 2.6.6.3.1 applyConstructorArgs

`#applyConstructorArgs(...)` 方法，将 `@Arg[]` 注解数组，解析成对应的 ResultMapping 对象们，并添加到 `resultMappings` 中。代码如下：

```
// MapperAnnotationBuilder.java

private void applyConstructorArgs(Arg[] args, Class<?> resultType, List<ResultMapping> resultMappings) {
    // 遍历 @Arg[] 数组
    for (Arg arg : args) {
        // 创建 ResultFlag 数组
        List<ResultFlag> flags = new ArrayList<>();
        flags.add(ResultFlag.CONSTRUCTOR);
        if (arg.id()) {
            flags.add(ResultFlag.ID);
        }
        @SuppressWarnings("unchecked")
        // 获得 TypeHandler 乐
        Class<? extends TypeHandler<?>> typeHandler = (Class<? extends TypeHandler<?>>)
                (arg.typeHandler() == UnknownTypeHandler.class ? null : arg.typeHandler());
        // 将当前 @Arg 注解构建成 ResultMapping 对象
        ResultMapping resultMapping = assistant.buildResultMapping(
                resultType,
                nullOrEmpty(arg.name()),
                nullOrEmpty(arg.column()),
                arg.javaType() == void.class ? null : arg.javaType(),
                arg.jdbcType() == JdbcType.UNDEFINED ? null : arg.jdbcType(),
                nullOrEmpty(arg.select()),
                nullOrEmpty(arg.resultMap()),
                null,
                nullOrEmpty(arg.columnPrefix()),
                typeHandler,
                flags,
                null,
                null,
                false);
        // 添加到 resultMappings 中
        resultMappings.add(resultMapping);
    }
}
```

- 从实现逻辑上，我们会发现，和 `XMLMapperBuilder#processConstructorElement(XNode resultChild, Class<?> resultType, List<ResultMapping> resultMappings)` 方法是**一致**的。所以就不重复解析了，胖友可以看看 [《精尽 MyBatis 源码分析 —— MyBatis 初始化（二）之加载 Mapper 映射配置》](http://svip.iocoder.cn/MyBatis/builder-package-2) 的 [「2.3.3.1 processConstructorElement」](http://svip.iocoder.cn/MyBatis/builder-package-4/#) 。

######## 2.6.6.3.2 applyResults

`#applyResults(Result[] results, Class<?> resultType, List<ResultMapping> resultMappings)` 方法，将 `@Result[]` 注解数组，解析成对应的 ResultMapping 对象们，并添加到 `resultMappings` 中。代码如下：

```
// MapperAnnotationBuilder.java

private void applyResults(Result[] results, Class<?> resultType, List<ResultMapping> resultMappings) {
    // 遍历 @Result[] 数组
    for (Result result : results) {
        // 创建 ResultFlag 数组
        List<ResultFlag> flags = new ArrayList<>();
        if (result.id()) {
            flags.add(ResultFlag.ID);
        }
        @SuppressWarnings("unchecked")
        // 获得 TypeHandler 类
        Class<? extends TypeHandler<?>> typeHandler = (Class<? extends TypeHandler<?>>)
                ((result.typeHandler() == UnknownTypeHandler.class) ? null : result.typeHandler());
        // 构建 ResultMapping 对象
        ResultMapping resultMapping = assistant.buildResultMapping(
                resultType,
                nullOrEmpty(result.property()),
                nullOrEmpty(result.column()),
                result.javaType() == void.class ? null : result.javaType(),
                result.jdbcType() == JdbcType.UNDEFINED ? null : result.jdbcType(),
                hasNestedSelect(result) ? nestedSelectId(result) : null, // <1.1> <1.2> 
                null,
                null,
                null,
                typeHandler,
                flags,
                null,
                null,
                isLazy(result)); // <2>
        // 添加到 resultMappings 中
        resultMappings.add(resultMapping);
    }
}
```

- 从实现逻辑上，我们会发现，和 `XMLMapperBuilder#buildResultMappingFromContext(XNode context, Class<?> resultType, List<ResultFlag> flags)` 方法是**一致**的。所以就不重复解析了，胖友可以看看 [《精尽 MyBatis 源码分析 —— MyBatis 初始化（二）之加载 Mapper 映射配置》](http://svip.iocoder.cn/MyBatis/builder-package-2) 的 [「2.3.3.3 buildResultMappingFromContext」](http://svip.iocoder.cn/MyBatis/builder-package-4/#) 。

- `<1.1>` 处，调用 `#hasNestedSelect(Result result)` 方法，判断是否有内嵌的查询。代码如下：

  ```
  // MapperAnnotationBuilder.java
  
  private boolean hasNestedSelect(Result result) {
      if (result.one().select().length() > 0 && result.many().select().length() > 0) {
          throw new BuilderException("Cannot use both @One and @Many annotations in the same @Result");
      }
      // 判断有 @One 或 @Many 注解
      return result.one().select().length() > 0 || result.many().select().length() > 0;
  }
  ```

- `<1.2>` 处，调用 `#nestedSelectId((Result result)` 方法，调用 `#nestedSelectId(Result result)` 方法，获得内嵌的查询编号。代码如下：

  ```
  // MapperAnnotationBuilder.java
  
  private String nestedSelectId(Result result) {
      // 先获得 @One 注解
      String nestedSelect = result.one().select();
      // 获得不到，则再获得 @Many
      if (nestedSelect.length() < 1) {
          nestedSelect = result.many().select();
      }
      // 获得内嵌查询编号，格式为 `{type.name}.${select}`
      if (!nestedSelect.contains(".")) {
          nestedSelect = type.getName() + "." + nestedSelect;
      }
      return nestedSelect;
  }
  ```

- `<2>` 处，调用 `#isLazy(Result result)` 方法，判断是否懒加载。代码如下：

  ```
  // MapperAnnotationBuilder.java
  
  private boolean isLazy(Result result) {
      // 判断是否开启懒加载
      boolean isLazy = configuration.isLazyLoadingEnabled();
      // 如果有 @One 注解，则判断是否懒加载
      if (result.one().select().length() > 0 && FetchType.DEFAULT != result.one().fetchType()) {
          isLazy = result.one().fetchType() == FetchType.LAZY;
      // 如果有 @Many 注解，则判断是否懒加载
      } else if (result.many().select().length() > 0 && FetchType.DEFAULT != result.many().fetchType()) {
          isLazy = result.many().fetchType() == FetchType.LAZY;
      }
      return isLazy;
  }
  ```

  - 根据全局是否懒加载 + `@One` 或 `@Many` 注解。

######## 2.6.6.3.3 applyDiscriminator

`#applyDiscriminator(...)` 方法，创建 Discriminator 对象。代码如下：

```
// MapperAnnotationBuilder.java

private Discriminator applyDiscriminator(String resultMapId, Class<?> resultType, TypeDiscriminator discriminator) {
    if (discriminator != null) {
        // 解析各种属性
        String column = discriminator.column();
        Class<?> javaType = discriminator.javaType() == void.class ? String.class : discriminator.javaType();
        JdbcType jdbcType = discriminator.jdbcType() == JdbcType.UNDEFINED ? null : discriminator.jdbcType();
        @SuppressWarnings("unchecked")
        // 获得 TypeHandler 类
        Class<? extends TypeHandler<?>> typeHandler = (Class<? extends TypeHandler<?>>)
                (discriminator.typeHandler() == UnknownTypeHandler.class ? null : discriminator.typeHandler());
        // 遍历 @Case[] 注解数组，解析成 discriminatorMap 集合
        Case[] cases = discriminator.cases();
        Map<String, String> discriminatorMap = new HashMap<>();
        for (Case c : cases) {
            String value = c.value();
            String caseResultMapId = resultMapId + "-" + value;
            discriminatorMap.put(value, caseResultMapId);
        }
        // 创建 Discriminator 对象
        return assistant.buildDiscriminator(resultType, column, javaType, jdbcType, typeHandler, discriminatorMap);
    }
    return null;
}
```

- 从实现逻辑上，我们会发现，和 `XMLMapperBuilder#processDiscriminatorElement(XNode context, Class<?> resultType, List<ResultMapping> resultMappings)` 方法是**一致**的。所以就不重复解析了，胖友可以看看 [《精尽 MyBatis 源码分析 —— MyBatis 初始化（二）之加载 Mapper 映射配置》](http://svip.iocoder.cn/MyBatis/builder-package-2) 的 [「2.3.3.2 processDiscriminatorElement」](http://svip.iocoder.cn/MyBatis/builder-package-4/#) 。

######## 2.6.6.3.4 createDiscriminatorResultMaps

`#createDiscriminatorResultMaps(...)` 方法，创建 Discriminator 的 ResultMap 对象们。代码如下：

```
// MapperAnnotationBuilder.java

private void createDiscriminatorResultMaps(String resultMapId, Class<?> resultType, TypeDiscriminator discriminator) {
    if (discriminator != null) {
        // 遍历 @Case 注解
        for (Case c : discriminator.cases()) {
            // 创建 @Case 注解的 ResultMap 的编号
            String caseResultMapId = resultMapId + "-" + c.value();
            // 创建 ResultMapping 数组
            List<ResultMapping> resultMappings = new ArrayList<>();
            // issue #136
            // 将 @Arg[] 注解数组，解析成对应的 ResultMapping 对象们，并添加到 resultMappings 中。
            applyConstructorArgs(c.constructArgs(), resultType, resultMappings);
            // 将 @Result[] 注解数组，解析成对应的 ResultMapping 对象们，并添加到 resultMappings 中。
            applyResults(c.results(), resultType, resultMappings);
            // TODO add AutoMappingBehaviour
            // 创建 ResultMap 对象
            assistant.addResultMap(caseResultMapId, c.type(), resultMapId, null, resultMappings, null);
        }
    }
}
```

- 逻辑比较简单，遍历 `@Case[]` 注解数组，创建每个 `@Case` 对应的 ResultMap 对象。

#### 2.7 MethodResolver

`org.apache.ibatis.builder.annotation.MethodResolver` ，注解方法的处理器。代码如下：

```
// MethodResolver.java

public class MethodResolver {

    /**
     * MapperAnnotationBuilder 对象
     */
    private final MapperAnnotationBuilder annotationBuilder;
    /**
     * Method 方法
     */
    private final Method method;

    public MethodResolver(MapperAnnotationBuilder annotationBuilder, Method method) {
        this.annotationBuilder = annotationBuilder;
        this.method = method;
    }

    public void resolve() {
        // 执行注解方法的解析
        annotationBuilder.parseStatement(method);
    }

}
```

- 在 `#resolve()` 方法里，可以调用 `MapperAnnotationBuilder#parseStatement(Method method)` 方法，执行注解方法的解析。

#### 2.8 parsePendingMethods

`#parsePendingMethods()` 方法，代码如下：

```
private void parsePendingMethods() {
    // 获得 MethodResolver 集合，并遍历进行处理
    Collection<MethodResolver> incompleteMethods = configuration.getIncompleteMethods();
    synchronized (incompleteMethods) {
        Iterator<MethodResolver> iter = incompleteMethods.iterator();
        while (iter.hasNext()) {
            try {
                // 执行解析
                iter.next().resolve();
                iter.remove();
                iter.remove();
            } catch (IncompleteElementException e) {
                // This method is still missing a resource
            }
        }
    }
}
```

- 《精尽 MyBatis 源码分析 —— MyBatis 初始化（二）之加载 Mapper 映射配置文件》

   

  的

   

  「2.5 parsePendingXXX」

   

  是一个思路。

  - 1）获得对应的集合；2）遍历集合，执行解析；3）执行成功，则移除出集合；4）执行失败，忽略异常。
  - 当然，实际上，此处还是可能有执行解析失败的情况，但是随着每一个 Mapper 接口对应的 MapperAnnotationBuilder 执行一次这些方法，逐步逐步就会被全部解析完。😈

# 总结

参考和推荐如下文章：

- 无忌 [《MyBatis 源码解读之配置》](https://my.oschina.net/wenjinglian/blog/1833051)

- 祖大俊 [《Mybatis3.3.x技术内幕（八）：Mybatis初始化流程（上）》](https://my.oschina.net/zudajun/blog/668738)

- 祖大俊 [《Mybatis3.3.x技术内幕（九）：Mybatis初始化流程（中）》](https://my.oschina.net/zudajun/blog/668787)

- 祖大俊 [《Mybatis3.3.x技术内幕（十）：Mybatis初始化流程（下）》](https://my.oschina.net/zudajun/blog/669868)

- 田小波 [《MyBatis 源码分析 - 映射文件解析过程》](https://www.tianxiaobo.com/2018/07/30/MyBatis-源码分析-映射文件解析过程/)

- 徐郡明 [《MyBatis 技术内幕》](https://item.jd.com/12125531.html) 的 [「3.1 MyBatis 初始化」](http://svip.iocoder.cn/MyBatis/builder-package-2/#) 小节

- 祖大俊 [《Mybatis3.4.x技术内幕（十六）：Mybatis之sqlFragment（可复用的sql片段）》](https://my.oschina.net/zudajun/blog/687326)

  