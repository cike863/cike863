# 1、调试环境搭建

## 1. 依赖工具

- Maven
- Git
- JDK
- IntelliJ IDEA

## 2. 源码拉取

从官方仓库 https://github.com/mybatis/spring `Fork` 出属于自己的仓库。为什么要 `Fork` ？既然开始阅读、调试源码，我们可能会写一些注释，有了自己的仓库，可以进行自由的提交。😈

使用 `IntelliJ IDEA` 从 `Fork` 出来的仓库拉取代码。

本文使用的 MyBatis 版本为 `2.0.0-SNAPSHOT` 。统计代码量如下：[![代码统计](http://static.iocoder.cn/images/MyBatis/2020_06_01/01.png)](http://static.iocoder.cn/images/MyBatis/2020_06_01/01.png)代码统计

## 3. 调试

MyBatis 想要调试，非常方便，只需要打开 `org.mybatis.spring.sample.AbstractSampleTest` 单元测试类，选择 `#testFooService()` 方法，右键，开始调试即可。

```
@DirtiesContext
public abstract class AbstractSampleTest {

    @Autowired
    protected FooService fooService;

    @Test
    final void testFooService() {
        User user = this.fooService.doSomeBusinessStuff("u1");
        assertThat(user).isNotNull();
        assertThat(user.getName()).isEqualTo("Pocoyo");
    }

}
```

有两点要注意：

1、因为单元测试基于 Junit5 ，所以需要依赖 JDK10 的环境，因此需要修改项目的 Project SDK 为 JDK10 。如下图所示：[![修改 Project SDK](http://static.iocoder.cn/images/MyBatis/2020_06_01/02.png)](http://static.iocoder.cn/images/MyBatis/2020_06_01/02.png)修改 Project SDK

2、AbstractSampleTest 是示例单元测试**基类**，它有五种实现类，分别如下：

```
/**
 * 示例单元测试基类
 *
 * 1. {@link SampleEnableTest}
 *      基于 @MapperScan 注解，扫描指定包
 *
 * 2. {@link SampleMapperTest}
 *      基于 {@link org.mybatis.spring.mapper.MapperFactoryBean} 类，直接声明指定的 Mapper 接口
 *
 * 3. {@link SampleNamespaceTest}
 *      基于 <mybatis:scan /> 标签，扫描指定包
 *
 * 4. {@link SampleScannerTest}
 *      基于 {@link org.mybatis.spring.mapper.MapperScannerConfigurer} 类，扫描指定包
 *
 * 5. {@link SampleBatchTest}
 *      在 SampleMapperTest 的基础上，使用 BatchExecutor 执行器
 */
```

- 每种实现类，都对应其配置类或配置文件。胖友可以根据自己需要，运行 AbstractSampleTest 时，选择对应的实现类。

# 2、初始化

## 1. 概述

在前面的，我们已经看了四篇 MyBatis 初始化相关的文章：

- [《MyBatis 初始化（一）之加载 mybatis-config》](http://svip.iocoder.cn/MyBatis/builder-package-1)
- [《MyBatis 初始化（二）之加载 Mapper 映射配置文件》](http://svip.iocoder.cn/MyBatis/builder-package-2)
- [《MyBatis 初始化（三）之加载 Statement 配置》](http://svip.iocoder.cn/MyBatis/builder-package-3)
- [《MyBatis 初始化（四）之加载注解配置》](http://svip.iocoder.cn/MyBatis/builder-package-4)

那么，本文我们就来看看，Spring 和 MyBatis 如何集成。主要涉及如下三个包：

- `annotation`
- `config`
- `mapper`

## 2. SqlSessionFactoryBean

`org.mybatis.spring.SqlSessionFactoryBean` ，实现 FactoryBean、InitializingBean、ApplicationListener 接口，负责创建 SqlSessionFactory 对象。

使用示例如下：

```
<!-- simplest possible SqlSessionFactory configuration -->
<bean id="sqlSessionFactory" class="org.mybatis.spring.SqlSessionFactoryBean">
    <property name="dataSource" ref="dataSource" />
    <!-- Directly specify the location of the MyBatis mapper xml file. This
         is NOT required when using MapperScannerConfigurer or
         MapperFactoryBean; they will load the xml automatically if it is
         in the same classpath location as the DAO interface. Rather than
         directly referencing the xml files, the 'configLocation' property
         could also be used to specify the location of a MyBatis config
         file. This config file could, in turn, contain &ltmapper&gt
         elements that point to the correct mapper xml files.
     -->
    <property name="mapperLocations" value="classpath:org/mybatis/spring/sample/mapper/*.xml" />
</bean>
```

另外，如果胖友不熟悉 Spring FactoryBean 的机制。可以看看 [《Spring bean 之 FactoryBean》](https://www.jianshu.com/p/d6c42d723464) 文章。

#### 2.1 构造方法

```
// SqlSessionFactoryBean.java

private static final Logger LOGGER = LoggerFactory.getLogger(SqlSessionFactoryBean.class);

/**
 * 指定 mybatis-config.xml 路径的 Resource 对象
 */
private Resource configLocation;

private Configuration configuration;

/**
 * 指定 Mapper 路径的 Resource 数组
 */
private Resource[] mapperLocations;

private DataSource dataSource;

private TransactionFactory transactionFactory;

private Properties configurationProperties;

private SqlSessionFactoryBuilder sqlSessionFactoryBuilder = new SqlSessionFactoryBuilder();

private SqlSessionFactory sqlSessionFactory;

//EnvironmentAware requires spring 3.1
private String environment = SqlSessionFactoryBean.class.getSimpleName();

private boolean failFast;

private Interceptor[] plugins;

private TypeHandler<?>[] typeHandlers;

private String typeHandlersPackage;

private Class<?>[] typeAliases;

private String typeAliasesPackage;

private Class<?> typeAliasesSuperType;

//issue #19. No default provider.
private DatabaseIdProvider databaseIdProvider;

private Class<? extends VFS> vfs;

private Cache cache;

private ObjectFactory objectFactory;

private ObjectWrapperFactory objectWrapperFactory;

// 省略 setting 方法
```

- 是不是各种熟悉的属性，这里就不多解释每个对象了。你比我懂，嘿嘿。

#### 2.2 afterPropertiesSet

`#afterPropertiesSet()` 方法，构建 SqlSessionFactory 对象。代码如下：

```
// SqlSessionFactoryBean.java

@Override
public void afterPropertiesSet() throws Exception {
    notNull(dataSource, "Property 'dataSource' is required");
    notNull(sqlSessionFactoryBuilder, "Property 'sqlSessionFactoryBuilder' is required");
    state((configuration == null && configLocation == null) || !(configuration != null && configLocation != null),
            "Property 'configuration' and 'configLocation' can not specified with together");

    // 创建 SqlSessionFactory 对象
    this.sqlSessionFactory = buildSqlSessionFactory();
}

/**
 * Build a {@code SqlSessionFactory} instance.
 *
 * The default implementation uses the standard MyBatis {@code XMLConfigBuilder} API to build a
 * {@code SqlSessionFactory} instance based on an Reader.
 * Since 1.3.0, it can be specified a {@link Configuration} instance directly(without config file).
 *
 * @return SqlSessionFactory
 * @throws IOException if loading the config file failed
 */
protected SqlSessionFactory buildSqlSessionFactory() throws IOException {
    Configuration configuration;

    // 初始化 configuration 对象，和设置其 `configuration.variables` 属性
    XMLConfigBuilder xmlConfigBuilder = null;
    if (this.configuration != null) {
        configuration = this.configuration;
        if (configuration.getVariables() == null) {
            configuration.setVariables(this.configurationProperties);
        } else if (this.configurationProperties != null) {
            configuration.getVariables().putAll(this.configurationProperties);
        }
    } else if (this.configLocation != null) {
        xmlConfigBuilder = new XMLConfigBuilder(this.configLocation.getInputStream(), null, this.configurationProperties);
        configuration = xmlConfigBuilder.getConfiguration();
    } else {
        LOGGER.debug(() -> "Property 'configuration' or 'configLocation' not specified, using default MyBatis Configuration");
        configuration = new Configuration();
        if (this.configurationProperties != null) {
            configuration.setVariables(this.configurationProperties);
        }
    }

    // 设置 `configuration.objectFactory` 属性
    if (this.objectFactory != null) {
        configuration.setObjectFactory(this.objectFactory);
    }

    // 设置 `configuration.objectWrapperFactory` 属性
    if (this.objectWrapperFactory != null) {
        configuration.setObjectWrapperFactory(this.objectWrapperFactory);
    }

    // 设置 `configuration.vfs` 属性
    if (this.vfs != null) {
        configuration.setVfsImpl(this.vfs);
    }

    // 设置 `configuration.typeAliasesPackage` 属性
    if (hasLength(this.typeAliasesPackage)) {
        String[] typeAliasPackageArray = tokenizeToStringArray(this.typeAliasesPackage,
                ConfigurableApplicationContext.CONFIG_LOCATION_DELIMITERS);
        for (String packageToScan : typeAliasPackageArray) {
            configuration.getTypeAliasRegistry().registerAliases(packageToScan,
                    typeAliasesSuperType == null ? Object.class : typeAliasesSuperType);
            LOGGER.debug(() -> "Scanned package: '" + packageToScan + "' for aliases");
        }
    }

    // 设置 `configuration.typeAliases` 属性
    if (!isEmpty(this.typeAliases)) {
        for (Class<?> typeAlias : this.typeAliases) {
            configuration.getTypeAliasRegistry().registerAlias(typeAlias);
            LOGGER.debug(() -> "Registered type alias: '" + typeAlias + "'");
        }
    }

    // 初始化 `configuration.interceptorChain` 属性，即拦截器
    if (!isEmpty(this.plugins)) {
        for (Interceptor plugin : this.plugins) {
            configuration.addInterceptor(plugin);
            LOGGER.debug(() -> "Registered plugin: '" + plugin + "'");
        }
    }

    // 扫描 typeHandlersPackage 包，注册 TypeHandler
    if (hasLength(this.typeHandlersPackage)) {
        String[] typeHandlersPackageArray = tokenizeToStringArray(this.typeHandlersPackage,
                ConfigurableApplicationContext.CONFIG_LOCATION_DELIMITERS);
        for (String packageToScan : typeHandlersPackageArray) {
            configuration.getTypeHandlerRegistry().register(packageToScan);
            LOGGER.debug(() -> "Scanned package: '" + packageToScan + "' for type handlers");
        }
    }
    // 如果 typeHandlers 非空，注册对应的 TypeHandler
    if (!isEmpty(this.typeHandlers)) {
        for (TypeHandler<?> typeHandler : this.typeHandlers) {
            configuration.getTypeHandlerRegistry().register(typeHandler);
            LOGGER.debug(() -> "Registered type handler: '" + typeHandler + "'");
        }
    }

    // 设置 `configuration.databaseId` 属性
    if (this.databaseIdProvider != null) {//fix #64 set databaseId before parse mapper xmls
        try {
            configuration.setDatabaseId(this.databaseIdProvider.getDatabaseId(this.dataSource));
        } catch (SQLException e) {
            throw new NestedIOException("Failed getting a databaseId", e);
        }
    }

    // 设置 `configuration.cache` 属性
    if (this.cache != null) {
        configuration.addCache(this.cache);
    }

    // <1> 解析 mybatis-config.xml 配置
    if (xmlConfigBuilder != null) {
        try {
            xmlConfigBuilder.parse();
            LOGGER.debug(() -> "Parsed configuration file: '" + this.configLocation + "'");
        } catch (Exception ex) {
            throw new NestedIOException("Failed to parse config resource: " + this.configLocation, ex);
        } finally {
            ErrorContext.instance().reset();
        }
    }

    // 初始化 TransactionFactory 对象
    if (this.transactionFactory == null) {
        this.transactionFactory = new SpringManagedTransactionFactory();
    }
    // 设置 `configuration.environment` 属性
    configuration.setEnvironment(new Environment(this.environment, this.transactionFactory, this.dataSource));

    // 扫描 Mapper XML 文件们，并进行解析
    if (!isEmpty(this.mapperLocations)) {
        for (Resource mapperLocation : this.mapperLocations) {
            if (mapperLocation == null) {
                continue;
            }

            try {
                XMLMapperBuilder xmlMapperBuilder = new XMLMapperBuilder(mapperLocation.getInputStream(),
                        configuration, mapperLocation.toString(), configuration.getSqlFragments());
                xmlMapperBuilder.parse();
            } catch (Exception e) {
                throw new NestedIOException("Failed to parse mapping resource: '" + mapperLocation + "'", e);
            } finally {
                ErrorContext.instance().reset();
            }
            LOGGER.debug(() -> "Parsed mapper file: '" + mapperLocation + "'");
        }
    } else {
        LOGGER.debug(() -> "Property 'mapperLocations' was not specified or no matching resources found");
    }

    // 创建 SqlSessionFactory 对象
    return this.sqlSessionFactoryBuilder.build(configuration);
}
```

- 代码比较长，胖友自己耐心看下注释，我们只挑重点的瞅瞅。
- `<1>` 处，调用 `XMLConfigBuilder#parse()` 方法，解析 `mybatis-config.xml` 配置。即对应 [《MyBatis 初始化（一）之加载 mybatis-config》](http://svip.iocoder.cn/MyBatis/builder-package-1) 文章。
- `<2>` 处，调用 `XMLMapperBuilder#parse()` 方法，解析 Mapper XML 配置。即对应 [《MyBatis 初始化（二）之加载 Mapper 映射配置文件》](http://svip.iocoder.cn/MyBatis/builder-package-2) 。
- `<3>` 处，调用 `SqlSessionFactoryBuilder#build(Configuration config)` 方法，创建 SqlSessionFactory 对象。
- 另外，我们看到 `LOGGER` 的使用。那么，可能胖友会跟艿艿有一样的疑问，`org.mybatis.logging` 包下，又定义了 Logger 呢？因为想使用方法参数为 `Supplier<String>` 的方法，即使用起来更加方便。这也是 `mybatis-spring` 项目的 `logging` 包的用途。感兴趣的胖友，可以自己去看看。

#### 2.3 getObject

`#getObject()` 方法，获得 SqlSessionFactory 对象。代码如下：

```
// SqlSessionFactoryBean.java

@Override
public SqlSessionFactory getObject() throws Exception {
    // 保证 SqlSessionFactory 对象的初始化
    if (this.sqlSessionFactory == null) {
        afterPropertiesSet();
    }

    return this.sqlSessionFactory;
}
```

#### 2.4 getObjectType

```
// SqlSessionFactoryBean.java

@Override
public Class<? extends SqlSessionFactory> getObjectType() {
    return this.sqlSessionFactory == null ? SqlSessionFactory.class : this.sqlSessionFactory.getClass();
}
```

#### 2.5 isSingleton

```
// SqlSessionFactoryBean.java

@Override
public boolean isSingleton() {
    return true;
}
```

#### 2.6 onApplicationEvent

`#onApplicationEvent()` 方法，监听 ContextRefreshedEvent 事件，如果 MapperStatement 们，没有都初始化**都**完成，会抛出 IncompleteElementException 异常。代码如下：

```
// SqlSessionFactoryBean.java

@Override
public void onApplicationEvent(ApplicationEvent event) {
    if (failFast && event instanceof ContextRefreshedEvent) {
        // fail-fast -> check all statements are completed
        // 如果 MapperStatement 们，没有都初始化完成，会抛出 IncompleteElementException 异常
        this.sqlSessionFactory.getConfiguration().getMappedStatementNames();
    }
}
```

- 使用该功能时，需要设置 `fastFast` 属性，为 `true` 。

------

在 `mybatis-spring` 项目中，提供了多种配置 Mapper 的方式。下面，我们一种一种来看。😈 实际上，配置 `SqlSessionFactoryBean.mapperLocations` 属性，也是方式之一。嘿嘿嘿。

## 3. MapperFactoryBean

`org.mybatis.spring.mapper.MapperFactoryBean` ，实现 FactoryBean 接口，继承 SqlSessionDaoSupport 抽象类，创建 Mapper 对象。

使用示例如下：

```
<!-- Directly injecting mappers. The required SqlSessionFactory will be autowired. -->
<bean id="userMapper" class="org.mybatis.spring.mapper.MapperFactoryBean" autowire="byType">
	<property name="mapperInterface" value="org.mybatis.spring.sample.mapper.UserMapper" />
</bean>
```

- 该示例来自 `org.mybatis.spring.sample.SampleMapperTest` 单元测试。胖友可以基于它调试。

#### 3.1 MapperFactoryBean

```
// MapperFactoryBean.java

/**
 * Mapper 接口
 */
private Class<T> mapperInterface;
/**
 * 是否添加到 {@link Configuration} 中
 */
private boolean addToConfig = true;

public MapperFactoryBean() {
    //intentionally empty
}

public MapperFactoryBean(Class<T> mapperInterface) {
    this.mapperInterface = mapperInterface;
}

// 省略 setting 方法
```

#### 3.2 checkDaoConfig

```
// MapperFactoryBean.java

@Override
protected void checkDaoConfig() {
    // <1> 校验 sqlSessionTemplate 非空
    super.checkDaoConfig();
    // <2> 校验 mapperInterface 非空
    notNull(this.mapperInterface, "Property 'mapperInterface' is required");

    // <3> 添加 Mapper 接口到 configuration 中
    Configuration configuration = getSqlSession().getConfiguration();
    if (this.addToConfig && !configuration.hasMapper(this.mapperInterface)) {
        try {
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

- 该方法，是在 `org.springframework.dao.support.DaoSupport` 定义，被 `#afterPropertiesSet()` 方法所调用，代码如下：

  ```
  // DaoSupport.java
  
  public final void afterPropertiesSet() throws IllegalArgumentException, BeanInitializationException {
      this.checkDaoConfig();
  
      try {
          this.initDao();
      } catch (Exception var2) {
          throw new BeanInitializationException("Initialization of DAO failed", var2);
      }
  }
  ```

  - 所以此处，就和 [「2.2 afterPropertiesSet」](http://svip.iocoder.cn/MyBatis/Spring-Integration-2/#) 的等价。

- `<1>` 处，调用父 `SqlSessionDaoSupport#checkDaoConfig()` 方法，校验 `sqlSessionTemplate` 非空。😈 关于 SqlSessionDaoSupport 抽象类，后续我们详细解析。

- `<2>` 处，校验 `mapperInterface` 非空。

- `<3>` 处，调用 `Configuration#addMapper(mapperInterface)` 方法，添加 Mapper 接口到 `configuration` 中。

#### 3.3 getObject

`#getObject()` 方法，获得 Mapper 对象。**注意**，返回的是基于 Mapper 接口自动生成的代理对象。代码如下：

```
// MapperFactoryBean.java

@Override
public T getObject() throws Exception {
    return getSqlSession().getMapper(this.mapperInterface);
}
```

#### 3.4 getObjectType

```
// MapperFactoryBean.java

@Override
public Class<T> getObjectType() {
    return this.mapperInterface;
}
```

#### 3.5 isSingleton

```
// MapperFactoryBean.java

@Override
public boolean isSingleton() {
    return true;
}
```

## 4. @MapperScan

> 艿艿：本小节，需要胖友对 Spring IOC 有一定的了解。如果不熟悉的胖友，建议不需要特别深入的理解。或者说，先去看 [《精尽 Spring 源码解析》](http://svip.iocoder.cn/categories/Spring/) 。

`org.mybatis.spring.annotation.@MapperScan` 注解，指定需要扫描的包，将包中符合的 Mapper 接口，注册成 `beanClass` 为 MapperFactoryBean 的 BeanDefinition 对象，从而实现创建 Mapper 对象。代码如下：

```
// MapperScan.java

@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.TYPE)
@Documented
@Import(MapperScannerRegistrar.class)
@Repeatable(MapperScans.class)
public @interface MapperScan {

    /**
     * Alias for the {@link #basePackages()} attribute. Allows for more concise
     * annotation declarations e.g.:
     * {@code @MapperScan("org.my.pkg")} instead of {@code @MapperScan(basePackages = "org.my.pkg"})}.
     *
     * 和 {@link #basePackages()} 相同意思
     *
     * @return base package names
     */
    String[] value() default {};

    /**
     * Base packages to scan for MyBatis interfaces. Note that only interfaces
     * with at least one method will be registered; concrete classes will be
     * ignored.
     *
     * 扫描的包地址
     *
     * @return base package names for scanning mapper interface
     */
    String[] basePackages() default {};

    /**
     * Type-safe alternative to {@link #basePackages()} for specifying the packages
     * to scan for annotated components. The package of each class specified will be scanned.
     * <p>Consider creating a special no-op marker class or interface in each package
     * that serves no purpose other than being referenced by this attribute.
     *
     * @return classes that indicate base package for scanning mapper interface
     */
    Class<?>[] basePackageClasses() default {};

    /**
     * The {@link BeanNameGenerator} class to be used for naming detected components
     * within the Spring container.
     *
     * @return the class of {@link BeanNameGenerator}
     */
    Class<? extends BeanNameGenerator> nameGenerator() default BeanNameGenerator.class;

    /**
     * This property specifies the annotation that the scanner will search for.
     * <p>
     * The scanner will register all interfaces in the base package that also have
     * the specified annotation.
     * <p>
     * Note this can be combined with markerInterface.
     *
     * 指定注解
     *
     * @return the annotation that the scanner will search for
     */
    Class<? extends Annotation> annotationClass() default Annotation.class;

    /**
     * This property specifies the parent that the scanner will search for.
     * <p>
     * The scanner will register all interfaces in the base package that also have
     * the specified interface class as a parent.
     * <p>
     * Note this can be combined with annotationClass.
     *
     * 指定接口
     *
     * @return the parent that the scanner will search for
     */
    Class<?> markerInterface() default Class.class;

    /**
     * Specifies which {@code SqlSessionTemplate} to use in the case that there is
     * more than one in the spring context. Usually this is only needed when you
     * have more than one datasource.
     *
     * 指向的 SqlSessionTemplate 的名字
     *
     * @return the bean name of {@code SqlSessionTemplate}
     */
    String sqlSessionTemplateRef() default "";

    /**
     * Specifies which {@code SqlSessionFactory} to use in the case that there is
     * more than one in the spring context. Usually this is only needed when you
     * have more than one datasource.
     *
     * 指向的 SqlSessionFactory 的名字
     *
     * @return the bean name of {@code SqlSessionFactory}
     */
    String sqlSessionFactoryRef() default "";

    /**
     * Specifies a custom MapperFactoryBean to return a mybatis proxy as spring bean.
     *
     * 可自定义 MapperFactoryBean 的实现类
     *
     * @return the class of {@code MapperFactoryBean}
     */
    Class<? extends MapperFactoryBean> factoryBean() default MapperFactoryBean.class;

}
```

- 属性比较多，实际上，胖友常用的就是 `value()` 属性。
- 重点是 `@Import(MapperScannerRegistrar.class)` 。为什么呢？`@Import` 注解，负责资源的导入。如果导入的是一个 Java 类，例如此处为 MapperScannerRegistrar 类，Spring 会将其注册成一个 Bean 对象。而 MapperScannerRegistrar 类呢？详细解析，见 [「4.1 MapperScannerRegistrar」](http://svip.iocoder.cn/MyBatis/Spring-Integration-2/#) 。

使用示例如下：

```
@Configuration
@ImportResource("classpath:org/mybatis/spring/sample/config/applicationContext-infrastructure.xml")
@MapperScan("org.mybatis.spring.sample.mapper") // here
static class AppConfig {
}
```

- 该示例来自 `org.mybatis.spring.sample.SampleEnableTest` 单元测试。胖友可以基于它调试。

#### 4.1 MapperScannerRegistrar

`org.mybatis.spring.annotation.MapperScannerRegistrar` ，实现 ImportBeanDefinitionRegistrar、ResourceLoaderAware 接口，`@MapperScann` 的注册器，负责将扫描到的 Mapper 接口，注册成 beanClass 为 MapperFactoryBean 的 BeanDefinition 对象，从而实现创建 Mapper 对象。

###### 4.1.1 构造方法

```
// MapperScannerRegistrar.java

/**
 * ResourceLoader 对象
 */
private ResourceLoader resourceLoader;

@Override
public void setResourceLoader(ResourceLoader resourceLoader) {
    this.resourceLoader = resourceLoader;
}
```

- 因为实现了 ResourceLoaderAware 接口，所以 `resourceLoader` 属性，能够被注入。

###### 4.1.2 registerBeanDefinitions

```
// MapperScannerRegistrar.java

@Override
public void registerBeanDefinitions(AnnotationMetadata importingClassMetadata, BeanDefinitionRegistry registry) {
    // <1> 获得 @MapperScan 注解信息
    AnnotationAttributes mapperScanAttrs = AnnotationAttributes
            .fromMap(importingClassMetadata.getAnnotationAttributes(MapperScan.class.getName()));
    // <2> 扫描包，将扫描到的 Mapper 接口，注册成 beanClass 为 MapperFactoryBean 的 BeanDefinition 对象
    registerBeanDefinitions(mapperScanAttrs, registry);
}

void registerBeanDefinitions(AnnotationAttributes annoAttrs, BeanDefinitionRegistry registry) {
    // <3.1> 创建 ClassPathMapperScanner 对象
    ClassPathMapperScanner scanner = new ClassPathMapperScanner(registry);

    // <3.2> 设置 resourceLoader 属性到 scanner 中
    // this check is needed in Spring 3.1
    if (resourceLoader != null) {
        scanner.setResourceLoader(resourceLoader);
    }

    // <3.3> 获得 @(MapperScan 注解上的属性，设置到 scanner 中
    Class<? extends Annotation> annotationClass = annoAttrs.getClass("annotationClass");
    if (!Annotation.class.equals(annotationClass)) {
        scanner.setAnnotationClass(annotationClass);
    }
    Class<?> markerInterface = annoAttrs.getClass("markerInterface");
    if (!Class.class.equals(markerInterface)) {
        scanner.setMarkerInterface(markerInterface);
    }
    Class<? extends BeanNameGenerator> generatorClass = annoAttrs.getClass("nameGenerator");
    if (!BeanNameGenerator.class.equals(generatorClass)) {
        scanner.setBeanNameGenerator(BeanUtils.instantiateClass(generatorClass));
    }
    Class<? extends MapperFactoryBean> mapperFactoryBeanClass = annoAttrs.getClass("factoryBean");
    if (!MapperFactoryBean.class.equals(mapperFactoryBeanClass)) {
        scanner.setMapperFactoryBean(BeanUtils.instantiateClass(mapperFactoryBeanClass));
    }
    scanner.setSqlSessionTemplateBeanName(annoAttrs.getString("sqlSessionTemplateRef"));
    scanner.setSqlSessionFactoryBeanName(annoAttrs.getString("sqlSessionFactoryRef"));

    // <4> 获得要扫描的包
    List<String> basePackages = new ArrayList<>();
    basePackages.addAll(
            Arrays.stream(annoAttrs.getStringArray("value")) // 包
                    .filter(StringUtils::hasText)
                    .collect(Collectors.toList()));
    basePackages.addAll(
            Arrays.stream(annoAttrs.getStringArray("basePackages")) // 包
                    .filter(StringUtils::hasText)
                    .collect(Collectors.toList()));
    basePackages.addAll(
            Arrays.stream(annoAttrs.getClassArray("basePackageClasses")) // 类
                    .map(ClassUtils::getPackageName)
                    .collect(Collectors.toList()));

    // <5> 注册 scanner 的过滤器
    scanner.registerFilters();
    // <6> 执行扫描
    scanner.doScan(StringUtils.toStringArray(basePackages));
}
```

- `<1>` 处，获得 `@MapperScan` 注解信息。

- `<2>` 处，调用 `#registerBeanDefinitions(AnnotationAttributes annoAttrs, BeanDefinitionRegistry registry)` 方法，扫描包，将扫描到的 Mapper 接口，注册成 `beanClass` 为 MapperFactoryBean 的 BeanDefinition 对象。

- ```
  <3.1>
  ```

   

  处，创建 ClassPathMapperScanner 对象。下面的代码，都是和 ClassPathMapperScanner 相关。所以，详细的解析，最终看

   

  「4.2 ClassPathMapperScanner」

   

  。

  - `<3.2>` 处， 设置 `resourceLoader` 属性到 `scanner` 中。
  - `<3.3>` 处，获得 `@MapperScan` 注解上的属性，设置到 `scanner` 中。

- `<4>` 处，获得要扫描的包。

- `<5>` 处，调用 `ClassPathMapperScanner#registerFilters()` 方法，注册 scanner 的过滤器。

- `<6>` 处，调用 `ClassPathMapperScanner#doScan(String... basePackages)` 方法，执行扫描，将扫描到的 Mapper 接口，注册成 beanClass 为 MapperFactoryBean 的 BeanDefinition 对象，从而实现创建 Mapper 对象。

- 😈 上面看了一堆的 ClassPathMapperScanner 类的调用，下面开始我们的旅程。

#### 4.2 ClassPathMapperScanner

`org.mybatis.spring.mapper.ClassPathMapperScanner` ，继承 `org.springframework.context.annotation.ClassPathMapperScanner` 类，负责执行扫描，将扫描到的 Mapper 接口，注册成 beanClass 为 MapperFactoryBean 的 BeanDefinition 对象，从而实现创建 Mapper 对象。

可能很多胖友不熟悉 ClassPathMapperScanner 类，可以看看 [《Spring自定义类扫描器》](https://fangjian0423.github.io/2017/06/11/spring-custom-component-provider/) 文章。哈哈哈，艿艿也是现学的。

###### 4.2.1 构造方法

```
// ClassPathMapperScanner.java

/**
 * 是否添加到 {@link org.apache.ibatis.session.Configuration} 中
 */
private boolean addToConfig = true;

private SqlSessionFactory sqlSessionFactory;

private SqlSessionTemplate sqlSessionTemplate;

/**
 * {@link #sqlSessionTemplate} 的 bean 名字
 */
private String sqlSessionTemplateBeanName;

/**
 * {@link #sqlSessionFactory} 的 bean 名字
 */
private String sqlSessionFactoryBeanName;

/**
 * 指定注解
 */
private Class<? extends Annotation> annotationClass;

/**
 * 指定接口
 */
private Class<?> markerInterface;

/**
 * MapperFactoryBean 对象
 */
private MapperFactoryBean<?> mapperFactoryBean = new MapperFactoryBean<>();

// ... 省略 setting 方法
```

###### 4.2.2 registerFilters

`#registerFilters()` 方法，注册过滤器。代码如下：

```
// ClassPathMapperScanner.java

/**
 * Configures parent scanner to search for the right interfaces. It can search
 * for all interfaces or just for those that extends a markerInterface or/and
 * those annotated with the annotationClass
 *
 * 注册过滤器
 */
public void registerFilters() {
    boolean acceptAllInterfaces = true; // 是否接受所有接口

    // if specified, use the given annotation and / or marker interface
    // 如果指定了注解，则添加 INCLUDE 过滤器 AnnotationTypeFilter 对象
    if (this.annotationClass != null) {
        addIncludeFilter(new AnnotationTypeFilter(this.annotationClass));
        acceptAllInterfaces = false; // 标记不是接受所有接口
    }

    // override AssignableTypeFilter to ignore matches on the actual marker interface
    // 如果指定了接口，则添加 INCLUDE 过滤器 AssignableTypeFilter 对象
    if (this.markerInterface != null) {
        addIncludeFilter(new AssignableTypeFilter(this.markerInterface) {
            @Override
            protected boolean matchClassName(String className) {
                return false;
            }
        });
        acceptAllInterfaces = false; // 标记不是接受所有接口
    }

    // 如果接受所有接口，则添加自定义 INCLUDE 过滤器 TypeFilter ，全部返回 true
    if (acceptAllInterfaces) {
        // default include filter that accepts all classes
        addIncludeFilter((metadataReader, metadataReaderFactory) -> true);
    }

    // exclude package-info.java
    // 添加 INCLUDE 过滤器，排除 package-info.java
    addExcludeFilter((metadataReader, metadataReaderFactory) -> {
        String className = metadataReader.getClassMetadata().getClassName();
        return className.endsWith("package-info");
    });
}
```

- 根据配置，添加 INCLUDE 和 EXCLUDE 过滤器。

###### 4.2.3 doScan

`#doScan(String... basePackages)` 方法，执行扫描，将扫描到的 Mapper 接口，注册成 beanClass 为 MapperFactoryBean 的 BeanDefinition 对象。代码如下：

```
// ClassPathMapperScanner.java

/**
 * Calls the parent search that will search and register all the candidates.
 * Then the registered objects are post processed to set them as
 * MapperFactoryBeans
 */
@Override
public Set<BeanDefinitionHolder> doScan(String... basePackages) {
    // <1> 执行扫描，获得包下符合的类们，并分装成 BeanDefinitionHolder 对象的集合
    Set<BeanDefinitionHolder> beanDefinitions = super.doScan(basePackages);

    if (beanDefinitions.isEmpty()) {
        LOGGER.warn(() -> "No MyBatis mapper was found in '" + Arrays.toString(basePackages) + "' package. Please check your configuration.");
    } else {
        // 处理 BeanDefinitionHolder 对象的集合
        processBeanDefinitions(beanDefinitions);
    }

    return beanDefinitions;
}
```

- `<1>` 处，调用父 `ClassPathBeanDefinitionScanner#doScan(basePackages)` 方法，执行扫描，获得包下符合的类们，并分装成 BeanDefinitionHolder 对象的集合。

- `<2>` 处，调用 `#processBeanDefinitions((Set<BeanDefinitionHolder> beanDefinitions)` 方法，处理 BeanDefinitionHolder 对象的集合。代码如下：

  ```
  // ClassPathMapperScanner.java
  
  private void processBeanDefinitions(Set<BeanDefinitionHolder> beanDefinitions) {
      GenericBeanDefinition definition;
      // <1> 遍历 BeanDefinitionHolder 数组，逐一设置属性
      for (BeanDefinitionHolder holder : beanDefinitions) {
          definition = (GenericBeanDefinition) holder.getBeanDefinition();
          String beanClassName = definition.getBeanClassName();
          LOGGER.debug(() -> "Creating MapperFactoryBean with name '" + holder.getBeanName()
                  + "' and '" + beanClassName + "' mapperInterface");
  
          // the mapper interface is the original class of the bean
          // but, the actual class of the bean is MapperFactoryBean
          // <2> 此处 definition 的 beanClass 为 Mapper 接口，需要修改成 MapperFactoryBean 类，从而创建 Mapper 代理对象
          definition.getConstructorArgumentValues().addGenericArgumentValue(beanClassName); // issue #59
          definition.setBeanClass(this.mapperFactoryBean.getClass());
  
          // <3> 设置 `MapperFactoryBean.addToConfig` 属性
          definition.getPropertyValues().add("addToConfig", this.addToConfig);
  
          boolean explicitFactoryUsed = false; // <4.1>是否已经显式设置了 sqlSessionFactory 或 sqlSessionFactory 属性
          // <4.2> 如果 sqlSessionFactoryBeanName 或 sqlSessionFactory 非空，设置到 `MapperFactoryBean.sqlSessionFactory` 属性
          if (StringUtils.hasText(this.sqlSessionFactoryBeanName)) {
              definition.getPropertyValues().add("sqlSessionFactory", new RuntimeBeanReference(this.sqlSessionFactoryBeanName));
              explicitFactoryUsed = true;
          } else if (this.sqlSessionFactory != null) {
              definition.getPropertyValues().add("sqlSessionFactory", this.sqlSessionFactory);
              explicitFactoryUsed = true;
          }
          // <4.3> 如果 sqlSessionTemplateBeanName 或 sqlSessionTemplate 非空，设置到 `MapperFactoryBean.sqlSessionTemplate` 属性
          if (StringUtils.hasText(this.sqlSessionTemplateBeanName)) {
              if (explicitFactoryUsed) {
                  LOGGER.warn(() -> "Cannot use both: sqlSessionTemplate and sqlSessionFactory together. sqlSessionFactory is ignored.");
              }
              definition.getPropertyValues().add("sqlSessionTemplate", new RuntimeBeanReference(this.sqlSessionTemplateBeanName));
              explicitFactoryUsed = true;
          } else if (this.sqlSessionTemplate != null) {
              if (explicitFactoryUsed) {
                  LOGGER.warn(() -> "Cannot use both: sqlSessionTemplate and sqlSessionFactory together. sqlSessionFactory is ignored.");
              }
              definition.getPropertyValues().add("sqlSessionTemplate", this.sqlSessionTemplate);
              explicitFactoryUsed = true;
          }
          // <4.4> 如果未显式设置，则设置根据类型自动注入
          if (!explicitFactoryUsed) {
              LOGGER.debug(() -> "Enabling autowire by type for MapperFactoryBean with name '" + holder.getBeanName() + "'.");
              definition.setAutowireMode(AbstractBeanDefinition.AUTOWIRE_BY_TYPE);
          }
      }
  }
  ```

  - 虽然方法很长，重点就是修改 BeanDefinitionHolder 的相关属性。

  - `<1>` 处，遍历 BeanDefinitionHolder 数组，逐一设置属性。

  - `<2>` 处，此处 `definition` 的 `beanClass` 为 Mapper 接口，需要修改成 MapperFactoryBean 类，从而创建 Mapper 代理对象。

  - `<3>` 处，设置 `MapperFactoryBean.addToConfig` 属性。

  - ```
    <4.1>
    ```

     

    处，是否已经显式设置了

     

    ```
    sqlSessionFactory 或 sqlSessionFactory
    ```

     

    属性。

    - `<4.2>` 处，如果 `sqlSessionFactoryBeanName` 或 `sqlSessionFactory` 非空，设置到 `MapperFactoryBean.sqlSessionFactory` 属性。
    - `<4.3>` 处，如果 `sqlSessionTemplateBeanName` 或 `sqlSessionTemplate` 非空，设置到 `MapperFactoryBean.sqlSessionTemplate` 属性。
    - `<4.4>` 处，如果未显式设置，**则设置根据类型自动注入**。

###### 4.3 @MapperScans

`org.mybatis.spring.annotation.@MapperScans` ，多 `@MapperScan` 的注解，功能是相同的。代码如下：

```
// MapperScans.java

@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.TYPE)
@Documented
@Import(MapperScannerRegistrar.RepeatingRegistrar.class)
public @interface MapperScans {

    /**
     * @return @MapperScan 数组
     */
    MapperScan[] value();

}
```

- 此处，`@Import(MapperScannerRegistrar.RepeatingRegistrar.class)` 是 RepeatingRegistrar 类。

###### 4.4 RepeatingRegistrar

RepeatingRegistrar ，是 MapperScannerRegistrar 的内部静态类，继承 MapperScannerRegistrar 类，`@MapperScans` 的注册器。代码如下：

```
// MapperScannerRegistrar.java

static class RepeatingRegistrar extends MapperScannerRegistrar {
    
    @Override
    public void registerBeanDefinitions(AnnotationMetadata importingClassMetadata, BeanDefinitionRegistry registry) {
        // 获得 @MapperScans 注解信息
        AnnotationAttributes mapperScansAttrs = AnnotationAttributes
                .fromMap(importingClassMetadata.getAnnotationAttributes(MapperScans.class.getName()));
        // 遍历 @MapperScans 的值，调用 `#registerBeanDefinitions(mapperScanAttrs, registry)` 方法，循环扫描处理
        Arrays.stream(mapperScansAttrs.getAnnotationArray("value"))
                .forEach(mapperScanAttrs -> registerBeanDefinitions(mapperScanAttrs, registry));
    }

}
```

- [「4.1.2 registerBeanDefinitions」](http://svip.iocoder.cn/MyBatis/Spring-Integration-2/#) 的**循环**版。

## 5. 自定义 `<mybatis:scan />` 标签

使用示例如下：

```
 <!-- Scan for mappers and let them be autowired; notice there is no
     UserDaoImplementation needed. The required SqlSessionFactory will be
     autowired. -->
<mybatis:scan base-package="org.mybatis.spring.sample.mapper" />
```

- 该示例来自 `org.mybatis.spring.sample.SampleNamespaceTest` 单元测试。胖友可以基于它调试。

简单来理解，`<mybatis:scan />` 标签，和 `@MapperScan` 注解，用途是等价的。

#### 5.1 spring.schemas

MyBatis 在 `META-INF/spring.schemas` 定义如下：

```
http\://mybatis.org/schema/mybatis-spring-1.2.xsd=org/mybatis/spring/config/mybatis-spring.xsd
http\://mybatis.org/schema/mybatis-spring.xsd=org/mybatis/spring/config/mybatis-spring.xsd
```

- xmlns 为 `http://mybatis.org/schema/mybatis-spring-1.2.xsd` 或 `http://mybatis.org/schema/mybatis-spring.xsd` 。
- xsd 为 `org/mybatis/spring/config/mybatis-spring.xsd` 。

#### 5.2 mybatis-spring.xsd

[![mybatis-spring.xsd](https://github.com/YunaiV/mybatis-spring/blob/master/src/main/java/org/mybatis/spring/config/mybatis-spring.xsd)](https://github.com/YunaiV/mybatis-spring/blob/master/src/main/java/org/mybatis/spring/config/mybatis-spring.xsd)mybatis-spring.xsd定义如下：[mybatis-spring.xsd](http://static.iocoder.cn/images/MyBatis/2020_06_04/01.png)

#### 5.3 spring.handler

`spring.handlers` 定义如下：

```
http\://mybatis.org/schema/mybatis-spring=org.mybatis.spring.config.NamespaceHandler
```

定义了 MyBatis 的 XML Namespace 的处理器 NamespaceHandler 。

#### 5.4 NamespaceHandler

`org.mybatis.spring.config.NamespaceHandler` ，继承 NamespaceHandlerSupport 抽象类，MyBatis 的 XML Namespace 的处理器。代码如下：

```
// NamespaceHandler.java

public class NamespaceHandler extends NamespaceHandlerSupport {

    @Override
    public void init() {
        registerBeanDefinitionParser("scan", new MapperScannerBeanDefinitionParser());
    }

}
```

- `<mybatis:scan />` 标签，使用 MapperScannerBeanDefinitionParser 解析。

#### 5.5 MapperScannerBeanDefinitionParser

`org.mybatis.spring.config.MapperScannerBeanDefinitionParser` ，实现 BeanDefinitionParser 接口，`<mybatis:scan />` 的解析器。代码如下：

```
// BeanDefinitionParser.java

public class MapperScannerBeanDefinitionParser implements BeanDefinitionParser {

    private static final String ATTRIBUTE_BASE_PACKAGE = "base-package";
    private static final String ATTRIBUTE_ANNOTATION = "annotation";
    private static final String ATTRIBUTE_MARKER_INTERFACE = "marker-interface";
    private static final String ATTRIBUTE_NAME_GENERATOR = "name-generator";
    private static final String ATTRIBUTE_TEMPLATE_REF = "template-ref";
    private static final String ATTRIBUTE_FACTORY_REF = "factory-ref";

    /**
     * {@inheritDoc}
     */
    @Override
    public synchronized BeanDefinition parse(Element element, ParserContext parserContext) {
        // 创建 ClassPathMapperScanner 对象
        ClassPathMapperScanner scanner = new ClassPathMapperScanner(parserContext.getRegistry());
        ClassLoader classLoader = scanner.getResourceLoader().getClassLoader();
        XmlReaderContext readerContext = parserContext.getReaderContext();
        scanner.setResourceLoader(readerContext.getResourceLoader()); // 设置 resourceLoader 属性
        try {
            // 解析 annotation 属性
            String annotationClassName = element.getAttribute(ATTRIBUTE_ANNOTATION);
            if (StringUtils.hasText(annotationClassName)) {
                @SuppressWarnings("unchecked")
                Class<? extends Annotation> markerInterface = (Class<? extends Annotation>) classLoader.loadClass(annotationClassName);
                scanner.setAnnotationClass(markerInterface);
            }
            // 解析 marker-interface 属性
            String markerInterfaceClassName = element.getAttribute(ATTRIBUTE_MARKER_INTERFACE);
            if (StringUtils.hasText(markerInterfaceClassName)) {
                Class<?> markerInterface = classLoader.loadClass(markerInterfaceClassName);
                scanner.setMarkerInterface(markerInterface);
            }
            // 解析 name-generator 属性
            String nameGeneratorClassName = element.getAttribute(ATTRIBUTE_NAME_GENERATOR);
            if (StringUtils.hasText(nameGeneratorClassName)) {
                Class<?> nameGeneratorClass = classLoader.loadClass(nameGeneratorClassName);
                BeanNameGenerator nameGenerator = BeanUtils.instantiateClass(nameGeneratorClass, BeanNameGenerator.class);
                scanner.setBeanNameGenerator(nameGenerator);
            }
        } catch (Exception ex) {
            readerContext.error(ex.getMessage(), readerContext.extractSource(element), ex.getCause());
        }
        // 解析 template-ref 属性
        String sqlSessionTemplateBeanName = element.getAttribute(ATTRIBUTE_TEMPLATE_REF);
        scanner.setSqlSessionTemplateBeanName(sqlSessionTemplateBeanName);
        // 解析 factory-ref 属性
        String sqlSessionFactoryBeanName = element.getAttribute(ATTRIBUTE_FACTORY_REF);
        scanner.setSqlSessionFactoryBeanName(sqlSessionFactoryBeanName);

        // 注册 scanner 的过滤器
        scanner.registerFilters();

        // 获得要扫描的包
        String basePackage = element.getAttribute(ATTRIBUTE_BASE_PACKAGE);
        // 执行扫描
        scanner.scan(StringUtils.tokenizeToStringArray(basePackage, ConfigurableApplicationContext.CONFIG_LOCATION_DELIMITERS));
        return null;
    }

}
```

- 代码实现上，和 [「4.2 ClassPathMapperScanner」](http://svip.iocoder.cn/MyBatis/Spring-Integration-2/#) 是基本一致的。所以就不详细解析啦。

## 6. MapperScannerConfigurer

`org.mybatis.spring.mapper.MapperScannerConfigurer` ，实现 BeanDefinitionRegistryPostProcessor、InitializingBean、ApplicationContextAware、BeanNameAware 接口，定义需要扫描的包，将包中符合的 Mapper 接口，注册成 beanClass 为 MapperFactoryBean 的 BeanDefinition 对象，从而实现创建 Mapper 对象。

使用示例如下：

```
// XML

<!-- Scan for mappers and let them be autowired; notice there is no
     UserDaoImplementation needed. The required SqlSessionFactory will be
     autowired. -->
<bean class="org.mybatis.spring.mapper.MapperScannerConfigurer">
    <property name="basePackage" value="org.mybatis.spring.sample.mapper" />
</bean>
```

- 该示例来自 `org.mybatis.spring.sample.MapperScannerConfigurer` 单元测试。胖友可以基于它调试。

#### 6.1 构造方法

```
// MapperScannerConfigurer.java

private String basePackage;

private boolean addToConfig = true;

private SqlSessionFactory sqlSessionFactory;

private SqlSessionTemplate sqlSessionTemplate;

private String sqlSessionFactoryBeanName;

private String sqlSessionTemplateBeanName;

private Class<? extends Annotation> annotationClass;

private Class<?> markerInterface;

private ApplicationContext applicationContext;

private String beanName;

private boolean processPropertyPlaceHolders;

private BeanNameGenerator nameGenerator;

// 省略 setting 方法
```

#### 6.2 afterPropertiesSet

```
// MapperScannerConfigurer.java

@Override
public void afterPropertiesSet() throws Exception {
    notNull(this.basePackage, "Property 'basePackage' is required");
}
```

- 啥都不做。

#### 6.3 postProcessBeanFactory

```
// MapperScannerConfigurer.java

@Override
public void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) {
    // left intentionally blank
}
```

#### 6.4 postProcessBeanDefinitionRegistry

```
// MapperScannerConfigurer.java

@Override
public void postProcessBeanDefinitionRegistry(BeanDefinitionRegistry registry) {
    // <1> 如果有属性占位符，则进行获得，例如 ${basePackage} 等等
    if (this.processPropertyPlaceHolders) {
        processPropertyPlaceHolders();
    }

    // <2> 创建 ClassPathMapperScanner 对象，并设置其相关属性
    ClassPathMapperScanner scanner = new ClassPathMapperScanner(registry);
    scanner.setAddToConfig(this.addToConfig);
    scanner.setAnnotationClass(this.annotationClass);
    scanner.setMarkerInterface(this.markerInterface);
    scanner.setSqlSessionFactory(this.sqlSessionFactory);
    scanner.setSqlSessionTemplate(this.sqlSessionTemplate);
    scanner.setSqlSessionFactoryBeanName(this.sqlSessionFactoryBeanName);
    scanner.setSqlSessionTemplateBeanName(this.sqlSessionTemplateBeanName);
    scanner.setResourceLoader(this.applicationContext);
    scanner.setBeanNameGenerator(this.nameGenerator);
    // 注册 scanner 过滤器
    scanner.registerFilters();
    // 执行扫描
    scanner.scan(StringUtils.tokenizeToStringArray(this.basePackage, ConfigurableApplicationContext.CONFIG_LOCATION_DELIMITERS));
}
```

- `<1>` 处，调用 `#processPropertyPlaceHolders()` 方法，如果有属性占位符，则进行获得，例如 `${basePackage}` 等等。代码如下：

  ```
  // MapperScannerConfigurer.java
  
  private void processPropertyPlaceHolders() {
      Map<String, PropertyResourceConfigurer> prcs = applicationContext.getBeansOfType(PropertyResourceConfigurer.class);
  
      if (!prcs.isEmpty() && applicationContext instanceof ConfigurableApplicationContext) {
          BeanDefinition mapperScannerBean = ((ConfigurableApplicationContext) applicationContext)
                  .getBeanFactory().getBeanDefinition(beanName);
  
          // PropertyResourceConfigurer does not expose any methods to explicitly perform
          // property placeholder substitution. Instead, create a BeanFactory that just
          // contains this mapper scanner and post process the factory.
          DefaultListableBeanFactory factory = new DefaultListableBeanFactory();
          factory.registerBeanDefinition(beanName, mapperScannerBean);
  
          for (PropertyResourceConfigurer prc : prcs.values()) {
              prc.postProcessBeanFactory(factory);
          }
  
          PropertyValues values = mapperScannerBean.getPropertyValues();
  
          this.basePackage = updatePropertyValue("basePackage", values);
          this.sqlSessionFactoryBeanName = updatePropertyValue("sqlSessionFactoryBeanName", values);
          this.sqlSessionTemplateBeanName = updatePropertyValue("sqlSessionTemplateBeanName", values);
      }
  }
  
  // 获得属性值，并转换成 String 类型
  private String updatePropertyValue(String propertyName, PropertyValues values) {
      PropertyValue property = values.getPropertyValue(propertyName);
      Object value = property.getValue();
      if (value instanceof String) {
          return value.toString();
      } else if (value instanceof TypedStringValue) {
          return ((TypedStringValue) value).getValue();
      } else {
          return null;
      }
  }
  ```

- `<2>` 处，代码实现上，和 [「4.2 ClassPathMapperScanner」](http://svip.iocoder.cn/MyBatis/Spring-Integration-2/#) 是基本一致的。所以就不详细解析啦。

# 3、SqlSession

## 1. 概述

本文我们就来看看，Spring 和 MyBatis 的 SqlSession 是如何集成。

## 2. SqlSessionTemplate

`org.mybatis.spring.SqlSessionTemplate` ，实现 SqlSession、DisposableBean 接口，SqlSession 操作模板实现类。实际上，代码实现和 `org.apache.ibatis.session.SqlSessionManager` 超级相似。或者说，这是 `mybatis-spring` 项目实现的 “SqlSessionManager” 类。

#### 2.1 构造方法

```
// SqlSessionTemplate.java

private final SqlSessionFactory sqlSessionFactory;

/**
 * 执行器类型
 */
private final ExecutorType executorType;
/**
 * SqlSession 代理对象
 */
private final SqlSession sqlSessionProxy;
/**
 * 异常转换器
 */
private final PersistenceExceptionTranslator exceptionTranslator;

public SqlSessionTemplate(SqlSessionFactory sqlSessionFactory) {
    this(sqlSessionFactory, sqlSessionFactory.getConfiguration().getDefaultExecutorType());
}

public SqlSessionTemplate(SqlSessionFactory sqlSessionFactory, ExecutorType executorType) {
    this(sqlSessionFactory, executorType,
            new MyBatisExceptionTranslator(
                    sqlSessionFactory.getConfiguration().getEnvironment().getDataSource(), true));
}

public SqlSessionTemplate(SqlSessionFactory sqlSessionFactory, ExecutorType executorType,
                          PersistenceExceptionTranslator exceptionTranslator) {

    notNull(sqlSessionFactory, "Property 'sqlSessionFactory' is required");
    notNull(executorType, "Property 'executorType' is required");

    this.sqlSessionFactory = sqlSessionFactory;
    this.executorType = executorType;
    this.exceptionTranslator = exceptionTranslator;
    // <1> 创建 sqlSessionProxy 对象
    this.sqlSessionProxy = (SqlSession) newProxyInstance(
            SqlSessionFactory.class.getClassLoader(),
            new Class[]{SqlSession.class},
            new SqlSessionInterceptor());
}

// ... 省略 setting 方法
```

- `executorType` 属性，执行器类型。后续，根据它来创建对应类型的执行器。
- `sqlSessionProxy` 属性，SqlSession 代理对象。在 `<1>` 处，进行创建 `sqlSessionProxy` 对象，使用的方法拦截器是 SqlSessionInterceptor 类。😈 是不是发现和 SqlSessionManager 灰常像。
- `exceptionTranslator` 属性，异常转换器。

#### 2.2 对 SqlSession 的实现方法

① 如下是直接调用 `sqlSessionProxy` 对应的方法。代码如下：

```
// SqlSessionTemplate.java

@Override
public <T> T selectOne(String statement) {
    return this.sqlSessionProxy.selectOne(statement);
}

@Override
public <T> T selectOne(String statement, Object parameter) {
    return this.sqlSessionProxy.selectOne(statement, parameter);
}

@Override
public <K, V> Map<K, V> selectMap(String statement, String mapKey) {
    return this.sqlSessionProxy.selectMap(statement, mapKey);
}

@Override
public <K, V> Map<K, V> selectMap(String statement, Object parameter, String mapKey) {
    return this.sqlSessionProxy.selectMap(statement, parameter, mapKey);
}

@Override
public <K, V> Map<K, V> selectMap(String statement, Object parameter, String mapKey, RowBounds rowBounds) {
    return this.sqlSessionProxy.selectMap(statement, parameter, mapKey, rowBounds);
}

@Override
public <T> Cursor<T> selectCursor(String statement) {
    return this.sqlSessionProxy.selectCursor(statement);
}

@Override
public <T> Cursor<T> selectCursor(String statement, Object parameter) {
    return this.sqlSessionProxy.selectCursor(statement, parameter);
}

@Override
public <T> Cursor<T> selectCursor(String statement, Object parameter, RowBounds rowBounds) {
    return this.sqlSessionProxy.selectCursor(statement, parameter, rowBounds);
}

@Override
public <E> List<E> selectList(String statement) {
    return this.sqlSessionProxy.selectList(statement);
}

@Override
public <E> List<E> selectList(String statement, Object parameter) {
    return this.sqlSessionProxy.selectList(statement, parameter);
}

@Override
public <E> List<E> selectList(String statement, Object parameter, RowBounds rowBounds) {
    return this.sqlSessionProxy.selectList(statement, parameter, rowBounds);
}

@Override
public void select(String statement, ResultHandler handler) {
    this.sqlSessionProxy.select(statement, handler);
}

@Override
public void select(String statement, Object parameter, ResultHandler handler) {
    this.sqlSessionProxy.select(statement, parameter, handler);
}

@Override
public void select(String statement, Object parameter, RowBounds rowBounds, ResultHandler handler) {
    this.sqlSessionProxy.select(statement, parameter, rowBounds, handler);
}

@Override
public int insert(String statement) {
    return this.sqlSessionProxy.insert(statement);
}

@Override
public int insert(String statement, Object parameter) {
    return this.sqlSessionProxy.insert(statement, parameter);
}

@Override
public int update(String statement) {
    return this.sqlSessionProxy.update(statement);
}

@Override
public int update(String statement, Object parameter) {
    return this.sqlSessionProxy.update(statement, parameter);
}

@Override
public int delete(String statement) {
    return this.sqlSessionProxy.delete(statement);
}

@Override
public int delete(String statement, Object parameter) {
    return this.sqlSessionProxy.delete(statement, parameter);
}

@Override
public <T> T getMapper(Class<T> type) {
    return getConfiguration().getMapper(type, this);
}

@Override
public void clearCache() {
    this.sqlSessionProxy.clearCache();
}

@Override
public Configuration getConfiguration() {
    return this.sqlSessionFactory.getConfiguration();
}

@Override
public Connection getConnection() {
    return this.sqlSessionProxy.getConnection();
}

@Override
public List<BatchResult> flushStatements() {
    return this.sqlSessionProxy.flushStatements();
}
```

② 如下是不支持的方法，直接抛出 UnsupportedOperationException 异常。代码如下：

```
// SqlSessionTemplate.java

@Override
public void commit() {
    throw new UnsupportedOperationException("Manual commit is not allowed over a Spring managed SqlSession");
}

@Override
public void commit(boolean force) {
    throw new UnsupportedOperationException("Manual commit is not allowed over a Spring managed SqlSession");
}

@Override
public void rollback() {
    throw new UnsupportedOperationException("Manual rollback is not allowed over a Spring managed SqlSession");
}

@Override
public void rollback(boolean force) {
    throw new UnsupportedOperationException("Manual rollback is not allowed over a Spring managed SqlSession");
}

@Override
public void close() {
    throw new UnsupportedOperationException("Manual close is not allowed over a Spring managed SqlSession");
}
```

- 和事务相关的方法，不允许**手动**调用。

#### 2.3 destroy

```
// SqlSessionTemplate.java

@Override
public void destroy() throws Exception {
    //This method forces spring disposer to avoid call of SqlSessionTemplate.close() which gives UnsupportedOperationException
}
```

#### 2.4 SqlSessionInterceptor

SqlSessionInterceptor ，是 SqlSessionTemplate 的内部类，实现 InvocationHandler 接口，将 SqlSession 的操作，路由到 Spring 托管的事务管理器中。代码如下：

```
// SqlSessionTemplate.java

/**
 * Proxy needed to route MyBatis method calls to the proper SqlSession got
 * from Spring's Transaction Manager
 * It also unwraps exceptions thrown by {@code Method#invoke(Object, Object...)} to
 * pass a {@code PersistenceException} to the {@code PersistenceExceptionTranslator}.
 */
private class SqlSessionInterceptor implements InvocationHandler {

    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        // <1> 获得 SqlSession 对象
        SqlSession sqlSession = getSqlSession(
                SqlSessionTemplate.this.sqlSessionFactory,
                SqlSessionTemplate.this.executorType,
                SqlSessionTemplate.this.exceptionTranslator);
        try {
            // 执行 SQL 操作
            Object result = method.invoke(sqlSession, args);
            // 如果非 Spring 托管的 SqlSession 对象，则提交事务
            if (!isSqlSessionTransactional(sqlSession, SqlSessionTemplate.this.sqlSessionFactory)) {
                // force commit even on non-dirty sessions because some databases require
                // a commit/rollback before calling close()
                sqlSession.commit(true);
            }
            // 返回结果
            return result;
        } catch (Throwable t) {
            // <4.1> 如果是 PersistenceException 异常，则进行转换
            Throwable unwrapped = unwrapThrowable(t);
            if (SqlSessionTemplate.this.exceptionTranslator != null && unwrapped instanceof PersistenceException) {
                // release the connection to avoid a deadlock if the translator is no loaded. See issue #22
                // <4.2> 根据情况，关闭 SqlSession 对象
                // 如果非 Spring 托管的 SqlSession 对象，则关闭 SqlSession 对象
                // 如果是 Spring 托管的 SqlSession 对象，则减少其 SqlSessionHolder 的计数
                closeSqlSession(sqlSession, SqlSessionTemplate.this.sqlSessionFactory);
                // <4.3> 置空，避免下面 final 又做处理
                sqlSession = null;
                // <4.4> 进行转换
                Throwable translated = SqlSessionTemplate.this.exceptionTranslator.translateExceptionIfPossible((PersistenceException) unwrapped);
                if (translated != null) {
                    unwrapped = translated;
                }
            }
            // <4.5> 抛出异常
            throw unwrapped;
        } finally {
            // <5> 根据情况，关闭 SqlSession 对象
            // 如果非 Spring 托管的 SqlSession 对象，则关闭 SqlSession 对象
            // 如果是 Spring 托管的 SqlSession 对象，则减少其 SqlSessionHolder 的计数
            if (sqlSession != null) {
                closeSqlSession(sqlSession, SqlSessionTemplate.this.sqlSessionFactory);
            }
        }
    }
}
```

- 类上的英文注释，胖友可以耐心看下。

- `<1>` 处，调用 `SqlSessionUtils#getSqlSession(SqlSessionFactory sessionFactory, ExecutorType executorType, PersistenceExceptionTranslator exceptionTranslator)` 方法，获得 SqlSession 对象。此处，和 Spring 事务托管的事务已经相关。详细的解析，见 [《精尽 MyBatis 源码解析 —— Spring 集成（四）之事务》](http://svip.iocoder.cn/MyBatis/Spring-Integration-4) 中。下面，和事务相关的，我们也统一放在该文中。

- `<2>` 处，调用 `Method#invoke(sqlSession, args)` 方法，**反射执行 SQL 操作**。

- `<3>` 处，调用 `SqlSessionUtils#isSqlSessionTransactional(SqlSession session, SqlSessionFactory sessionFactory)` 方法，判断是否为**非** Spring 托管的 SqlSession 对象，则调用 `SqlSession#commit(true)` 方法，提交事务。

- ```
  <4.1>
  ```

   

  处，如果是 PersistenceException 异常，则：

  - `<4.2>` 处，调用 `SqlSessionUtils#closeSqlSession(SqlSession session, SqlSessionFactory sessionFactory)` 方法，根据情况，关闭 SqlSession 对象。1）如果非 Spring 托管的 SqlSession 对象，则关闭 SqlSession 对象。2）如果是 Spring 托管的 SqlSession 对象，则减少其 SqlSessionHolder 的计数。😈 也就是说，Spring 托管事务的情况下，最终是在“外部”执行最终的事务处理。
  - `<4.3>` 处，置空 `sqlSession` 属性，避免下面 `final` 又做关闭处理。
  - `<4.4>` 处，调用 `MyBatisExceptionTranslator#translateExceptionIfPossible(RuntimeException e)` 方法，转换异常。
  - `<4.5>` 处，抛出异常。

- `<5>` 处，如果 `sqlSession` 非空，则调用 `SqlSessionUtils#closeSqlSession(SqlSession session, SqlSessionFactory sessionFactory)` 方法，根据情况，关闭 SqlSession 对象。

## 3. SqlSessionDaoSupport

`org.mybatis.spring.support.SqlSessionDaoSupport` ，继承 DaoSupport 抽象类，SqlSession 的 DaoSupport 抽象类。代码如下：

```
// SqlSessionDaoSupport.java

public abstract class SqlSessionDaoSupport extends DaoSupport {

    /**
     * SqlSessionTemplate 对象
     */
    private SqlSessionTemplate sqlSessionTemplate;

    /**
     * Set MyBatis SqlSessionFactory to be used by this DAO.
     * Will automatically create SqlSessionTemplate for the given SqlSessionFactory.
     *
     * @param sqlSessionFactory a factory of SqlSession
     */
    public void setSqlSessionFactory(SqlSessionFactory sqlSessionFactory) {
        if (this.sqlSessionTemplate == null || sqlSessionFactory != this.sqlSessionTemplate.getSqlSessionFactory()) {
            this.sqlSessionTemplate = createSqlSessionTemplate(sqlSessionFactory); // 使用 sqlSessionFactory 属性，创建 sqlSessionTemplate 对象
        }
    }

    /**
     * Create a SqlSessionTemplate for the given SqlSessionFactory.
     * Only invoked if populating the DAO with a SqlSessionFactory reference!
     * <p>Can be overridden in subclasses to provide a SqlSessionTemplate instance
     * with different configuration, or a custom SqlSessionTemplate subclass.
     * @param sqlSessionFactory the MyBatis SqlSessionFactory to create a SqlSessionTemplate for
     * @return the new SqlSessionTemplate instance
     * @see #setSqlSessionFactory
     */
    @SuppressWarnings("WeakerAccess")
    protected SqlSessionTemplate createSqlSessionTemplate(SqlSessionFactory sqlSessionFactory) {
        return new SqlSessionTemplate(sqlSessionFactory);
    }

    /**
     * Return the MyBatis SqlSessionFactory used by this DAO.
     *
     * @return a factory of SqlSession
     */
    public final SqlSessionFactory getSqlSessionFactory() {
        return (this.sqlSessionTemplate != null ? this.sqlSessionTemplate.getSqlSessionFactory() : null);
    }


    /**
     * Set the SqlSessionTemplate for this DAO explicitly,
     * as an alternative to specifying a SqlSessionFactory.
     *
     * @param sqlSessionTemplate a template of SqlSession
     * @see #setSqlSessionFactory
     */
    public void setSqlSessionTemplate(SqlSessionTemplate sqlSessionTemplate) {
        this.sqlSessionTemplate = sqlSessionTemplate;
    }

    /**
     * Users should use this method to get a SqlSession to call its statement methods
     * This is SqlSession is managed by spring. Users should not commit/rollback/close it
     * because it will be automatically done.
     *
     * @return Spring managed thread safe SqlSession
     */
    public SqlSession getSqlSession() {
        return this.sqlSessionTemplate;
    }

    /**
     * Return the SqlSessionTemplate for this DAO,
     * pre-initialized with the SessionFactory or set explicitly.
     * <p><b>Note: The returned SqlSessionTemplate is a shared instance.</b>
     * You may introspect its configuration, but not modify the configuration
     * (other than from within an {@link #initDao} implementation).
     * Consider creating a custom SqlSessionTemplate instance via
     * {@code new SqlSessionTemplate(getSqlSessionFactory())}, in which case
     * you're allowed to customize the settings on the resulting instance.
     *
     * @return a template of SqlSession
     */
    public SqlSessionTemplate getSqlSessionTemplate() {
        return this.sqlSessionTemplate;
    }

    /**
     * {@inheritDoc}
     */
    @Override
    protected void checkDaoConfig() {
        notNull(this.sqlSessionTemplate, "Property 'sqlSessionFactory' or 'sqlSessionTemplate' are required");
    }

}
```

- 主要用途是，`sqlSessionTemplate` 属性的初始化，或者注入。
- 具体示例，可以看看 `org.mybatis.spring.sample.dao.UserDaoImpl` 类。

## 4. 异常相关

`mybatis-spring` 项目中，有两个异常相关的类：

- `org.mybatis.spring.MyBatisExceptionTranslator` ，MyBatis 自定义的异常转换器。
- `org.mybatis.spring.MyBatisSystemException` ，MyBatis 自定义的异常类。

代码比较简单，胖友自己去看。

# 4、事务

## 1. 概述

本文我们就来看看，Spring 和 MyBatis 的**事务**是如何集成。需要胖友阅读的前置文章是：

- [《精尽 MyBatis 源码分析 —— 事务模块》](http://svip.iocoder.cn/MyBatis/transaction-package/)
- [《精尽 Spring 源码分析 —— Transaction 源码简单导读》](http://svip.iocoder.cn/Spring/transaction-simple-intro/) 需要胖友对 Spring Transaction 有源码级的了解，否则对该文章，会有点懵逼。

## 2. SpringManagedTransaction

`org.mybatis.spring.transaction.SpringManagedTransaction` ，实现 `org.apache.ibatis.transaction.Transaction` 接口，Spring 托管事务的 Transaction 实现类。

#### 2.1 构造方法

```
// SpringManagedTransaction.java

/**
 * DataSource 对象
 */
private final DataSource dataSource;
/**
 * Connection 对象
 */
private Connection connection;
/**
 * 当前连接是否处于事务中
 *
 * @see DataSourceUtils#isConnectionTransactional(Connection, DataSource)
 */
private boolean isConnectionTransactional;
/**
 * 是否自动提交
 */
private boolean autoCommit;

public SpringManagedTransaction(DataSource dataSource) {
    notNull(dataSource, "No DataSource specified");
    this.dataSource = dataSource;
}
```

#### 2.2 getConnection

`#getConnection()` 方法，获得连接。代码如下：

```
// SpringManagedTransaction.java

@Override
public Connection getConnection() throws SQLException {
    if (this.connection == null) {
        // 如果连接不存在，获得连接
        openConnection();
    }
    return this.connection;
}
```

- 如果 `connection` 为空，则调用 `#openConnection()` 方法，获得连接。代码如下：

  ```
  // SpringManagedTransaction.java
  
  /**
   * Gets a connection from Spring transaction manager and discovers if this
   * {@code Transaction} should manage connection or let it to Spring.
   * <p>
   * It also reads autocommit setting because when using Spring Transaction MyBatis
   * thinks that autocommit is always false and will always call commit/rollback
   * so we need to no-op that calls.
   */
  private void openConnection() throws SQLException {
      // 获得连接
      this.connection = DataSourceUtils.getConnection(this.dataSource);
      this.autoCommit = this.connection.getAutoCommit();
      this.isConnectionTransactional = DataSourceUtils.isConnectionTransactional(this.connection, this.dataSource);
  
      LOGGER.debug(() ->
              "JDBC Connection ["
                      + this.connection
                      + "] will"
                      + (this.isConnectionTransactional ? " " : " not ")
                      + "be managed by Spring");
  }
  ```

  - 比较有趣的是，此处获取连接，不是通过 `DataSource#getConnection()` 方法，而是通过 `org.springframework.jdbc.datasource.DataSourceUtils#getConnection(DataSource dataSource)` 方法，获得 Connection 对象。而实际上，基于 Spring Transaction 体系，如果此处正在**事务中**时，已经有和当前线程绑定的 Connection 对象，就是存储在 ThreadLocal 中。

#### 2.3 commit

`#commit()` 方法，提交事务。代码如下：

```
// SpringManagedTransaction.java

@Override
public void commit() throws SQLException {
    if (this.connection != null && !this.isConnectionTransactional && !this.autoCommit) {
        LOGGER.debug(() -> "Committing JDBC Connection [" + this.connection + "]");
        this.connection.commit();
    }
}
```

#### 2.4 rollback

`#rollback()` 方法，回滚事务。代码如下：

```
// SpringManagedTransaction.java

@Override
public void rollback() throws SQLException {
    if (this.connection != null && !this.isConnectionTransactional && !this.autoCommit) {
        LOGGER.debug(() -> "Rolling back JDBC Connection [" + this.connection + "]");
        this.connection.rollback();
    }
}
```

#### 2.5 close

`#close()` 方法，释放连接。代码如下：

```
// SpringManagedTransaction.java

@Override
public void close() throws SQLException {
    DataSourceUtils.releaseConnection(this.connection, this.dataSource);
}
```

- 比较有趣的是，此处获取连接，不是通过 `Connection#close()` 方法，而是通过 `org.springframework.jdbc.datasource.DataSourceUtils#releaseConnection(Connection connection,DataSource dataSource)` 方法，“释放”连接。但是，具体会不会关闭连接，根据当前线程绑定的 Connection 对象，是不是传入的 `connection` 参数。

#### 2.6 getTimeout

```
// SpringManagedTransaction.java

@Override
public Integer getTimeout() throws SQLException {
    ConnectionHolder holder = (ConnectionHolder) TransactionSynchronizationManager.getResource(dataSource);
    if (holder != null && holder.hasTimeout()) {
        return holder.getTimeToLiveInSeconds();
    }
    return null;
}
```

## 3. SpringManagedTransactionFactory

`org.mybatis.spring.transaction.SpringManagedTransactionFactory` ，实现 TransactionFactory 接口，SpringManagedTransaction 的工厂实现类。代码如下：

```
// SpringManagedTransactionFactory.java

public class SpringManagedTransactionFactory implements TransactionFactory {

    @Override
    public Transaction newTransaction(DataSource dataSource, TransactionIsolationLevel level, boolean autoCommit) {
        // 创建 SpringManagedTransaction 对象
        return new SpringManagedTransaction(dataSource);
    }

    @Override
    public Transaction newTransaction(Connection conn) {
        // 抛出异常，因为 Spring 事务，需要一个 DataSource 对象
        throw new UnsupportedOperationException("New Spring transactions require a DataSource");
    }

    @Override
    public void setProperties(Properties props) {
        // not needed in this version
    }

}
```

## 4. SqlSessionHolder

`org.mybatis.spring.SqlSessionHolder` ，继承 `org.springframework.transaction.support.ResourceHolderSupport` 抽象类，SqlSession 持有器，用于保存当前 SqlSession 对象，保存到 `org.springframework.transaction.support.TransactionSynchronizationManager` 中。代码如下：

```
// SqlSessionHolder.java

/**
 * Used to keep current {@code SqlSession} in {@code TransactionSynchronizationManager}.
 *
 * The {@code SqlSessionFactory} that created that {@code SqlSession} is used as a key.
 * {@code ExecutorType} is also kept to be able to check if the user is trying to change it
 * during a TX (that is not allowed) and throw a Exception in that case.
 *
 * SqlSession 持有器，用于保存当前 SqlSession 对象，保存到 TransactionSynchronizationManager 中
 *
 * @author Hunter Presnall
 * @author Eduardo Macarron
 */
public final class SqlSessionHolder extends ResourceHolderSupport {

    /**
     * SqlSession 对象
     */
    private final SqlSession sqlSession;
    /**
     * 执行器类型
     */
    private final ExecutorType executorType;
    /**
     * PersistenceExceptionTranslator 对象
     */
    private final PersistenceExceptionTranslator exceptionTranslator;

    /**
     * Creates a new holder instance.
     *
     * @param sqlSession the {@code SqlSession} has to be hold.
     * @param executorType the {@code ExecutorType} has to be hold.
     * @param exceptionTranslator the {@code PersistenceExceptionTranslator} has to be hold.
     */
    public SqlSessionHolder(SqlSession sqlSession,
                            ExecutorType executorType,
                            PersistenceExceptionTranslator exceptionTranslator) {
        notNull(sqlSession, "SqlSession must not be null");
        notNull(executorType, "ExecutorType must not be null");

        this.sqlSession = sqlSession;
        this.executorType = executorType;
        this.exceptionTranslator = exceptionTranslator;
    }

    // ... 省略 getting 方法

}
```

- 当存储到 TransactionSynchronizationManager 中时，使用的 KEY 为创建该 SqlSession 对象的 SqlSessionFactory 对象。详细解析，见 `SqlSessionUtils#registerSessionHolder(...)` 方法。

## 5. SqlSessionUtils

`org.mybatis.spring.SqlSessionUtils` ，SqlSession 工具类。它负责处理 MyBatis SqlSession 的生命周期。它可以从 Spring TransactionSynchronizationManager 中，注册和获得对应的 SqlSession 对象。同时，它也支持当前不处于事务的情况下。

😈 当然，这个描述看起来，有点绕。所以，我们直接干起代码。

#### 5.1 构造方法

```
// SqlSessionUtils.java

private static final String NO_EXECUTOR_TYPE_SPECIFIED = "No ExecutorType specified";
private static final String NO_SQL_SESSION_FACTORY_SPECIFIED = "No SqlSessionFactory specified";
private static final String NO_SQL_SESSION_SPECIFIED = "No SqlSession specified";

/**
 * This class can't be instantiated, exposes static utility methods only.
 */
private SqlSessionUtils() {
    // do nothing
}
```

- 空的，直接跳过。

#### 5.2 getSqlSession

`#getSqlSession(SqlSessionFactory sessionFactory, ...)` 方法，获得 SqlSession 对象。代码如下：

```
// SqlSessionUtils.java

/**
 * Creates a new MyBatis {@code SqlSession} from the {@code SqlSessionFactory}
 * provided as a parameter and using its {@code DataSource} and {@code ExecutorType}
 *
 * @param sessionFactory a MyBatis {@code SqlSessionFactory} to create new sessions
 * @return a MyBatis {@code SqlSession}
 * @throws TransientDataAccessResourceException if a transaction is active and the
 *             {@code SqlSessionFactory} is not using a {@code SpringManagedTransactionFactory}
 */
public static SqlSession getSqlSession(SqlSessionFactory sessionFactory) {
    // 获得执行器类型
    ExecutorType executorType = sessionFactory.getConfiguration().getDefaultExecutorType();
    // 获得 SqlSession 对象
    return getSqlSession(sessionFactory, executorType, null);
}

/**
 * Gets an SqlSession from Spring Transaction Manager or creates a new one if needed.
 * Tries to get a SqlSession out of current transaction. If there is not any, it creates a new one.
 * Then, it synchronizes the SqlSession with the transaction if Spring TX is active and
 * <code>SpringManagedTransactionFactory</code> is configured as a transaction manager.
 *
 * @param sessionFactory a MyBatis {@code SqlSessionFactory} to create new sessions
 * @param executorType The executor type of the SqlSession to create
 * @param exceptionTranslator Optional. Translates SqlSession.commit() exceptions to Spring exceptions.
 * @return an SqlSession managed by Spring Transaction Manager
 * @throws TransientDataAccessResourceException if a transaction is active and the
 *             {@code SqlSessionFactory} is not using a {@code SpringManagedTransactionFactory}
 * @see SpringManagedTransactionFactory
 */
public static SqlSession getSqlSession(SqlSessionFactory sessionFactory, ExecutorType executorType, PersistenceExceptionTranslator exceptionTranslator) {
    notNull(sessionFactory, NO_SQL_SESSION_FACTORY_SPECIFIED);
    notNull(executorType, NO_EXECUTOR_TYPE_SPECIFIED);

    // <1> 获得 SqlSessionHolder 对象
    SqlSessionHolder holder = (SqlSessionHolder) TransactionSynchronizationManager.getResource(sessionFactory);

    // <2.1> 获得 SqlSession 对象
    SqlSession session = sessionHolder(executorType, holder);
    if (session != null) { // <2.2> 如果非空，直接返回
        return session;
    }

    LOGGER.debug(() -> "Creating a new SqlSession");
    // <3.1> 创建 SqlSession 对象
    session = sessionFactory.openSession(executorType);
    // <3.2> 注册到 TransactionSynchronizationManager 中
    registerSessionHolder(sessionFactory, executorType, exceptionTranslator, session);

    return session;
}
```

- 我们先看看每一步的作用，然后胖友在自己看下上面的英文注释，嘿嘿。

- `<1>` 处，调用 `TransactionSynchronizationManager#getResource(sessionFactory)` 方法，获得 SqlSessionHolder 对象。为什么可以获取到呢？答案在 `<3.2>` 中。关于 TransactionSynchronizationManager 类，如果不熟悉的胖友，真的真的真的，先去看懂 Spring Transaction 体系。

- `<2.1>` 处，调用 `#sessionHolder(ExecutorType executorType, SqlSessionHolder holder)` 方法，从 SqlSessionHolder 中，获得 SqlSession 对象。代码如下：

  ```
  // SqlSessionUtils.java
  
  private static SqlSession sessionHolder(ExecutorType executorType, SqlSessionHolder holder) {
      SqlSession session = null;
      if (holder != null && holder.isSynchronizedWithTransaction()) {
          // 如果执行器类型发生了变更，抛出 TransientDataAccessResourceException 异常
          if (holder.getExecutorType() != executorType) {
              throw new TransientDataAccessResourceException("Cannot change the ExecutorType when there is an existing transaction");
          }
  
          // <1> 增加计数
          holder.requested();
  
          LOGGER.debug(() -> "Fetched SqlSession [" + holder.getSqlSession() + "] from current transaction");
          // <2> 获得 SqlSession 对象
          session = holder.getSqlSession();
      }
      return session;
  }
  ```

  - `<1>` 处，调用 `SqlSessionHolder#requested()` 方法，增加计数。注意，这个的计数，是用于关闭 SqlSession 时使用。详细解析，见 TODO 方法。
  - `<2>` 处，调用 `SqlSessionHolder#getSqlSession()` 方法，获得 SqlSession 对象。

- `<2.2>` 处，如果非空，直接返回。

- `<3.1>` 处，调用 `SqlSessionFactory#openSession(executorType)` 方法，创建 SqlSession 对象。其中，使用的执行器类型，由传入的 `executorType` 方法参数所决定。

- `<3.2>` 处，调用 `#registerSessionHolder(SqlSessionFactory sessionFactory, ExecutorType executorType, PersistenceExceptionTranslator exceptionTranslator, SqlSession session)` 方法，注册 SqlSession 对象，到 TransactionSynchronizationManager 中。代码如下：

  ```
  // SqlSessionUtils.java
  
  /**
   * Register session holder if synchronization is active (i.e. a Spring TX is active).
   *
   * Note: The DataSource used by the Environment should be synchronized with the
   * transaction either through DataSourceTxMgr or another tx synchronization.
   * Further assume that if an exception is thrown, whatever started the transaction will
   * handle closing / rolling back the Connection associated with the SqlSession.
   *
   * @param sessionFactory sqlSessionFactory used for registration.
   * @param executorType executorType used for registration.
   * @param exceptionTranslator persistenceExceptionTranslator used for registration.
   * @param session sqlSession used for registration.
   */
  private static void registerSessionHolder(SqlSessionFactory sessionFactory, ExecutorType executorType,
                                            PersistenceExceptionTranslator exceptionTranslator, SqlSession session) {
      SqlSessionHolder holder;
      if (TransactionSynchronizationManager.isSynchronizationActive()) {
          Environment environment = sessionFactory.getConfiguration().getEnvironment();
  
          // <1> 如果使用 Spring 事务管理器
          if (environment.getTransactionFactory() instanceof SpringManagedTransactionFactory) {
              LOGGER.debug(() -> "Registering transaction synchronization for SqlSession [" + session + "]");
  
              // <1.1> 创建 SqlSessionHolder 对象
              holder = new SqlSessionHolder(session, executorType, exceptionTranslator);
              // <1.2> 绑定到 TransactionSynchronizationManager 中
              TransactionSynchronizationManager.bindResource(sessionFactory, holder);
              // <1.3> 创建 SqlSessionSynchronization 到 TransactionSynchronizationManager 中
              TransactionSynchronizationManager.registerSynchronization(new SqlSessionSynchronization(holder, sessionFactory));
              // <1.4> 设置同步
              holder.setSynchronizedWithTransaction(true);
              // <1.5> 增加计数
              holder.requested();
          // <2> 如果非 Spring 事务管理器，抛出 TransientDataAccessResourceException 异常
          } else {
              TransactionSynchronizationManager.getResource(environment.getDataSource());
              throw new TransientDataAccessResourceException(
                      "SqlSessionFactory must be using a SpringManagedTransactionFactory in order to use Spring transaction synchronization");
          }
      } else {
          LOGGER.debug(() -> "SqlSession [" + session + "] was not registered for synchronization because synchronization is not active");
      }
  }
  ```

  - `<2>` 处，如果非 Spring 事务管理器，抛出 TransientDataAccessResourceException 异常。

  - ```
    <1>
    ```

     

    处，如果使用 Spring 事务管理器(

     

    SpringManagedTransactionFactory

     

    )，则进行注册。

    - `<1.1>` 处，将 `sqlSession` 封装成 SqlSessionHolder 对象。
    - `<1.2>` 处，调用 `TransactionSynchronizationManager#bindResource(sessionFactory, holder)` 方法，绑定 `holder` 到 TransactionSynchronizationManager 中。**注意**，此时的 KEY 是 `sessionFactory` ，就创建 `sqlSession` 的 SqlSessionFactory 对象。
    - `<1.3>` 处，创建 SqlSessionSynchronization 到 TransactionSynchronizationManager 中。详细解析，见 [「6. SqlSessionSynchronization」](http://svip.iocoder.cn/MyBatis/Spring-Integration-4/#) 。
    - `<1.4>` 处，设置同步。
    - `<1.5>` 处，调用 `SqlSessionHolder#requested()` 方法，增加计数。

#### 5.3 isSqlSessionTransactional

`#isSqlSessionTransactional(SqlSession session, SqlSessionFactory sessionFactory)` 方法，判断传入的 SqlSession 参数，是否在 Spring 事务中。代码如下：

```
// SqlSessionUtils.java

/**
 * Returns if the {@code SqlSession} passed as an argument is being managed by Spring
 *
 * @param session a MyBatis SqlSession to check
 * @param sessionFactory the SqlSessionFactory which the SqlSession was built with
 * @return true if session is transactional, otherwise false
 */
public static boolean isSqlSessionTransactional(SqlSession session, SqlSessionFactory sessionFactory) {
    notNull(session, NO_SQL_SESSION_SPECIFIED);
    notNull(sessionFactory, NO_SQL_SESSION_FACTORY_SPECIFIED);

    // 从 TransactionSynchronizationManager 中，获得 SqlSessionHolder 对象
    SqlSessionHolder holder = (SqlSessionHolder) TransactionSynchronizationManager.getResource(sessionFactory);
    // 如果相等，说明在 Spring 托管的事务中
    return (holder != null) && (holder.getSqlSession() == session);
}
```

- 代码比较简单，直接看注释。

#### 5.4 closeSqlSession

`#closeSqlSession(SqlSession session, SqlSessionFactory sessionFactory)` 方法，关闭 SqlSession 对象。代码如下：

```
// SqlSessionUtils.java

/**
 * Checks if {@code SqlSession} passed as an argument is managed by Spring {@code TransactionSynchronizationManager}
 * If it is not, it closes it, otherwise it just updates the reference counter and
 * lets Spring call the close callback when the managed transaction ends
 *
 * @param session a target SqlSession
 * @param sessionFactory a factory of SqlSession
 */
public static void closeSqlSession(SqlSession session, SqlSessionFactory sessionFactory) {
    notNull(session, NO_SQL_SESSION_SPECIFIED);
    notNull(sessionFactory, NO_SQL_SESSION_FACTORY_SPECIFIED);

    // <1> 从 TransactionSynchronizationManager 中，获得 SqlSessionHolder 对象
    SqlSessionHolder holder = (SqlSessionHolder) TransactionSynchronizationManager.getResource(sessionFactory);

    // <2.1> 如果相等，说明在 Spring 托管的事务中，则释放 holder 计数
    if ((holder != null) && (holder.getSqlSession() == session)) {
        LOGGER.debug(() -> "Releasing transactional SqlSession [" + session + "]");
        holder.released();
    // <2.2> 如果不相等，说明不在 Spring 托管的事务中，直接关闭 SqlSession 对象
    } else {
        LOGGER.debug(() -> "Closing non transactional SqlSession [" + session + "]");
        session.close();
    }
}
```

- `<1>` 处，从 TransactionSynchronizationManager 中，获得 SqlSessionHolder 对象。
- `<2.1>` 处，如果相等，说明在 Spring 托管的事务中，则释放 `holder` 计数。那有什么用呢？具体原因，见 [「6. SqlSessionSynchronization」](http://svip.iocoder.cn/MyBatis/Spring-Integration-4/#) 。
- `<2.2>` 处，如果不相等，说明不在 Spring 托管的事务中，直接关闭 SqlSession 对象。

## 6. SqlSessionSynchronization

SqlSessionSynchronization ，是 SqlSessionUtils 的内部类，继承 TransactionSynchronizationAdapter 抽象类，SqlSession 的 同步器，基于 Spring Transaction 体系。

```
/**
 * Callback for cleaning up resources. It cleans TransactionSynchronizationManager and
 * also commits and closes the {@code SqlSession}.
 * It assumes that {@code Connection} life cycle will be managed by
 * {@code DataSourceTransactionManager} or {@code JtaTransactionManager}
 */
```

#### 6.1 构造方法

```
// SqlSessionSynchronization.java

/**
 * SqlSessionHolder 对象
 */
private final SqlSessionHolder holder;
/**
 * SqlSessionFactory 对象
 */
private final SqlSessionFactory sessionFactory;
/**
 * 是否开启
 */
private boolean holderActive = true;

public SqlSessionSynchronization(SqlSessionHolder holder, SqlSessionFactory sessionFactory) {
    notNull(holder, "Parameter 'holder' must be not null");
    notNull(sessionFactory, "Parameter 'sessionFactory' must be not null");

    this.holder = holder;
    this.sessionFactory = sessionFactory;
}
```

#### 6.2 getOrder

```
// SqlSessionSynchronization.java

@Override
public int getOrder() {
    // order right before any Connection synchronization
    return DataSourceUtils.CONNECTION_SYNCHRONIZATION_ORDER - 1;
}
```

#### 6.3 suspend

`#suspend()` 方法，当事务挂起时，取消当前线程的绑定的 SqlSessionHolder 对象。代码如下：

```
// SqlSessionSynchronization.java

@Override
public void suspend() {
    if (this.holderActive) {
        LOGGER.debug(() -> "Transaction synchronization suspending SqlSession [" + this.holder.getSqlSession() + "]");
        TransactionSynchronizationManager.unbindResource(this.sessionFactory);
    }
}
```

#### 6.4 resume

`#resume()` 方法，当事务恢复时，重新绑定当前线程的 SqlSessionHolder 对象。代码如下：

```
// SqlSessionSynchronization.java

public void resume() {
    if (this.holderActive) {
        LOGGER.debug(() -> "Transaction synchronization resuming SqlSession [" + this.holder.getSqlSession() + "]");
        TransactionSynchronizationManager.bindResource(this.sessionFactory, this.holder);
    }
}
```

- 因为，当前 SqlSessionSynchronization 对象中，有 `holder` 对象，所以可以直接恢复。

#### 6.5 beforeCommit

`#beforeCommit(boolean readOnly)` 方法，在事务提交之前，调用 `SqlSession#commit()` 方法，提交事务。虽然说，Spring 自身也会调用 `Connection#commit()` 方法，进行事务的提交。但是，`SqlSession#commit()` 方法中，不仅仅有事务的提交，还有提交批量操作，刷新本地缓存等等。代码如下：

```
// SqlSessionSynchronization.java

@Override
public void beforeCommit(boolean readOnly) {
    // Connection commit or rollback will be handled by ConnectionSynchronization or
    // DataSourceTransactionManager.
    // But, do cleanup the SqlSession / Executor, including flushing BATCH statements so
    // they are actually executed.
    // SpringManagedTransaction will no-op the commit over the jdbc connection
    // TODO This updates 2nd level caches but the tx may be rolledback later on!
    if (TransactionSynchronizationManager.isActualTransactionActive()) {
        try {
            LOGGER.debug(() -> "Transaction synchronization committing SqlSession [" + this.holder.getSqlSession() + "]");
            // 提交事务
            this.holder.getSqlSession().commit();
        } catch (PersistenceException p) {
            // 如果发生异常，则进行转换，并抛出异常
            if (this.holder.getPersistenceExceptionTranslator() != null) {
                DataAccessException translated = this.holder
                        .getPersistenceExceptionTranslator()
                        .translateExceptionIfPossible(p);
                throw translated;
            }
            throw p;
        }
    }
}
```

- 耐心的看看英文注释，更有助于理解该方法。

#### 6.6 beforeCompletion

> 老艿艿：TransactionSynchronization 的事务提交的执行顺序是：beforeCommit => beforeCompletion => 提交操作 => afterCompletion => afterCommit 。

`#beforeCompletion()` 方法，提交事务完成之前，关闭 SqlSession 对象。代码如下：

```
// SqlSessionSynchronization.java

@Override
public void beforeCompletion() {
    // Issue #18 Close SqlSession and deregister it now
    // because afterCompletion may be called from a different thread
    if (!this.holder.isOpen()) {
        LOGGER.debug(() -> "Transaction synchronization deregistering SqlSession [" + this.holder.getSqlSession() + "]");
        // 取消当前线程的绑定的 SqlSessionHolder 对象
        TransactionSynchronizationManager.unbindResource(sessionFactory);
        // 标记无效
        this.holderActive = false;
        LOGGER.debug(() -> "Transaction synchronization closing SqlSession [" + this.holder.getSqlSession() + "]");
        // 关闭 SqlSession 对象
        this.holder.getSqlSession().close();
    }
}
```

- 因为，beforeCompletion 方法是在 beforeCommit 之后执行，并且在 beforeCommit 已经提交了事务，所以此处可以放心关闭 SqlSession 对象了。

- 要执行关闭操作之前，需要先调用 `SqlSessionHolder#isOpen()` 方法来判断，是否处于开启状态。代码如下：

  ```
  // ResourceHolderSupport.java
  
  public boolean isOpen() {
  	return (this.referenceCount > 0);
  }
  ```

  - 这就是，我们前面看到的各种计数增减的作用。

#### 6.7 afterCompletion

`#afterCompletion()` 方法，解决可能出现的**跨线程**的情况，简单理解下就好。代码如下：

```
// ResourceHolderSupport.java

@Override
public void afterCompletion(int status) {
    if (this.holderActive) { // 处于有效状态
        // afterCompletion may have been called from a different thread
        // so avoid failing if there is nothing in this one
        LOGGER.debug(() -> "Transaction synchronization deregistering SqlSession [" + this.holder.getSqlSession() + "]");
        // 取消当前线程的绑定的 SqlSessionHolder 对象
        TransactionSynchronizationManager.unbindResourceIfPossible(sessionFactory);
        // 标记无效
        this.holderActive = false;
        LOGGER.debug(() -> "Transaction synchronization closing SqlSession [" + this.holder.getSqlSession() + "]");
        // 关闭 SqlSession 对象
        this.holder.getSqlSession().close();
    }
    this.holder.reset();
}
```

- 😈 貌似，官方没对这块做单元测试。

# 5、批处理

## 1. 概述

本文我们就来看看，Spring 和 MyBatis 的**批处理**是如何集成。它在 `batch` 包下实现，基于 [Spring Batch](https://spring.io/projects/spring-batch) 框架实现，整体类图如下：[![类图](http://static.iocoder.cn/images/MyBatis/2020_06_13/01.png)](http://static.iocoder.cn/images/MyBatis/2020_06_13/01.png)类图

- 读操作：MyBatisCursorItemReader、MyBatisPagingItemReader 。
- 写操作：MyBatisBatchItemWriter 。

## 2. 调试环境

在 `org.mybatis.spring.batch.SpringBatchTest` 单元测试类中，可以直接进行调试。

😈 调试起来把，胖友。

## 2. MyBatisPagingItemReader

`org.mybatis.spring.batch.MyBatisPagingItemReader` ，继承 `org.springframework.batch.item.database.AbstractPagingItemReader` 抽象类，基于分页的 MyBatis 的读取器。

#### 2.1 示例

```
// SpringBatchTest.java

@Test
@Transactional
void shouldDuplicateSalaryOfAllEmployees() throws Exception {
    // <x> 批量读取 Employee 数组
    List<Employee> employees = new ArrayList<>();
    Employee employee = pagingNoNestedItemReader.read();
    while (employee != null) {
        employee.setSalary(employee.getSalary() * 2);
        employees.add(employee);
        employee = pagingNoNestedItemReader.read();
    }

    // 批量写入
    //noinspection Duplicates
    writer.write(employees);

    assertThat((Integer) session.selectOne("checkSalarySum")).isEqualTo(20000);
    assertThat((Integer) session.selectOne("checkEmployeeCount")).isEqualTo(employees.size());
}
```

- `<x>` 处，不断读取，直到为空。

#### 2.2 构造方法

```
// MyBatisPagingItemReader.java

/**
 * SqlSessionFactory 对象
 */
private SqlSessionFactory sqlSessionFactory;
/**
 * SqlSessionTemplate 对象
 */
private SqlSessionTemplate sqlSessionTemplate;
/**
 * 查询编号
 */
private String queryId;
/**
 * 参数值的映射
 */
private Map<String, Object> parameterValues;

public MyBatisPagingItemReader() {
    setName(getShortName(MyBatisPagingItemReader.class));
}

// ... 省略 setting 方法
```

#### 2.3 afterPropertiesSet

```
// MyBatisPagingItemReader.java

@Override
public void afterPropertiesSet() throws Exception {
    // 父类的处理
    super.afterPropertiesSet();
    notNull(sqlSessionFactory, "A SqlSessionFactory is required.");
    // 创建 SqlSessionTemplate 对象
    sqlSessionTemplate = new SqlSessionTemplate(sqlSessionFactory, ExecutorType.BATCH);
    notNull(queryId, "A queryId is required.");
}
```

- 主要目的，是创建 SqlSessionTemplate 对象。

#### 2.4 doReadPage

`#doReadPage()` 方法，执行每一次分页的读取。代码如下：

```
// MyBatisPagingItemReader.java

@Override
protected void doReadPage() {
    // <1> 创建 parameters 参数
    Map<String, Object> parameters = new HashMap<>();
    // <1.1> 设置原有参数
    if (parameterValues != null) {
        parameters.putAll(parameterValues);
    }
    // <1.2> 设置分页参数
    parameters.put("_page", getPage());
    parameters.put("_pagesize", getPageSize());
    parameters.put("_skiprows", getPage() * getPageSize());
    // <2> 清空目前的 results 结果
    if (results == null) {
        results = new CopyOnWriteArrayList<>();
    } else {
        results.clear();
    }
    // <3> 查询结果
    results.addAll(sqlSessionTemplate.selectList(queryId, parameters));
}
```

- ```
  <1>
  ```

   

  处，创建

   

  ```
  parameters
  ```

   

  参数。

  - `<1.1>` 处，设置原有参数。
  - `<1.2>` 处，设置分页参数。

- `<2>` 处，创建或清空当前的 `results` ，保证是空的数组。使用 CopyOnWriteArrayList 的原因是，可能存在并发读取的问题。

- 【重要】 `<3>` 处，调用 `SqlSessionTemplate#selectList(queryId, parameters)` 方法，执行查询列表。查询后，将结果添加到 `results` 中。

#### 2.5 doJumpToPage

```
// MyBatisPagingItemReader.java

@Override
protected void doJumpToPage(int itemIndex) {
    // Not Implemented
}
```

## 3. MyBatisCursorItemReader

`org.mybatis.spring.batch.MyBatisCursorItemReader` ，继承 `org.springframework.batch.item.support.AbstractItemCountingItemStreamItemReader` 抽象类，实现 InitializingBean 接口，基于 **Cursor** 的 MyBatis 的读取器。

#### 3.1 示例

```
// SpringBatchTest.java

@Test
@Transactional
void checkCursorReadingWithoutNestedInResultMap() throws Exception {
    // 打开 Cursor
    cursorNoNestedItemReader.doOpen();
    try {
        // Employee 数组
        List<Employee> employees = new ArrayList<>();
        // <x> 循环读取，写入到 Employee 数组中
        Employee employee = cursorNoNestedItemReader.read();
        while (employee != null) {
            employee.setSalary(employee.getSalary() * 2);
            employees.add(employee);
            employee = cursorNoNestedItemReader.read();
        }

        // 批量写入
        writer.write(employees);

        assertThat((Integer) session.selectOne("checkSalarySum")).isEqualTo(20000);
        assertThat((Integer) session.selectOne("checkEmployeeCount")).isEqualTo(employees.size());
    } finally {
        // 关闭 Cursor
        cursorNoNestedItemReader.doClose();
    }
}
```

- `<x>` 处，不断读取，直到为空。

#### 3.2 构造方法

```
// MyBatisCursorItemReader.java

/**
 * SqlSessionFactory 对象
 */
private SqlSessionFactory sqlSessionFactory;
/**
 * SqlSession 对象
 */
private SqlSession sqlSession;

/**
 * 查询编号
 */
private String queryId;
/**
 * 参数值的映射
 */
private Map<String, Object> parameterValues;

/**
 * Cursor 对象
 */
private Cursor<T> cursor;
/**
 * {@link #cursor} 的迭代器
 */
private Iterator<T> cursorIterator;

public MyBatisCursorItemReader() {
    setName(getShortName(MyBatisCursorItemReader.class));
}

// ... 省略 setting 方法
```

#### 3.3 afterPropertiesSet

```
// MyBatisCursorItemReader.java

@Override
public void afterPropertiesSet() throws Exception {
    notNull(sqlSessionFactory, "A SqlSessionFactory is required.");
    notNull(queryId, "A queryId is required.");
}
```

#### 3.4 doOpen

`#doOpen()` 方法，打开 Cursor 。代码如下：

```
// MyBatisCursorItemReader.java

@Override
protected void doOpen() throws Exception {
    // <1> 创建 parameters 参数
    Map<String, Object> parameters = new HashMap<>();
    if (parameterValues != null) {
        parameters.putAll(parameterValues);
    }

    // <2> 创建 SqlSession 对象
    sqlSession = sqlSessionFactory.openSession(ExecutorType.SIMPLE);

    // <3.1> 查询，返回 Cursor 对象
    cursor = sqlSession.selectCursor(queryId, parameters);
    // <3.2> 获得 cursor 的迭代器
    cursorIterator = cursor.iterator();
}
```

- `<1>` 处，创建 parameters 参数。

- `<2>` 处，调用 `SqlSessionFactory#openSession(ExecutorType.SIMPLE)` 方法，创建简单执行器的 SqlSession 对象。

- ```
  <3.1>
  ```

   

  处，调用

   

  ```
  SqlSession#selectCursor(queryId, parameters)
  ```

   

  方法，返回 Cursor 对象。

  - `<3.2>` 处，调用 `Cursor#iterator()` 方法，获得迭代器。

#### 3.5 doRead

`#doRead()` 方法，读取下一条数据。代码如下：

```
// MyBatisCursorItemReader.java

@Override
protected T doRead() throws Exception {
    // 置空 next
    T next = null;
    // 读取下一条
    if (cursorIterator.hasNext()) {
        next = cursorIterator.next();
    }
    // 返回
    return next;
}
```

#### 3.6 doClose

`#doClose()` 方法，关闭 Cursor 和 SqlSession 对象。代码如下：

```
// MyBatisCursorItemReader.java

@Override
protected void doClose() throws Exception {
    // 关闭 cursor 对象
    if (cursor != null) {
        cursor.close();
    }
    // 关闭 sqlSession 对象
    if (sqlSession != null) {
        sqlSession.close();
    }
    // 置空 cursorIterator
    cursorIterator = null;
}
```

## 4. MyBatisBatchItemWriter

`org.mybatis.spring.batch.MyBatisBatchItemWriter` ，实现 `org.springframework.batch.item.ItemWriter`、InitializingBean 接口，MyBatis 批量写入器。

#### 4.1 示例

```
// SpringBatchTest.java

@Test
@Transactional
void shouldDuplicateSalaryOfAllEmployees() throws Exception {
    // 批量读取 Employee 数组
    List<Employee> employees = new ArrayList<>();
    Employee employee = pagingNoNestedItemReader.read();
    while (employee != null) {
        employee.setSalary(employee.getSalary() * 2);
        employees.add(employee);
        employee = pagingNoNestedItemReader.read();
    }

    // <x> 批量写入
    writer.write(employees);

    assertThat((Integer) session.selectOne("checkSalarySum")).isEqualTo(20000);
    assertThat((Integer) session.selectOne("checkEmployeeCount")).isEqualTo(employees.size());
}
```

- `<x>` 处，执行**一次**批量写入到数据库中。

#### 4.2 构造方法

```
// SpringBatchTest.java

/**
 * SqlSessionTemplate 对象
 */
private SqlSessionTemplate sqlSessionTemplate;

/**
 * 语句编号
 */
private String statementId;

/**
 * 是否校验
 */
private boolean assertUpdates = true;

/**
 * 参数转换器
 */
private Converter<T, ?> itemToParameterConverter = new PassThroughConverter<>();

// ... 省略 setting 方法
```

#### 4.3 PassThroughConverter

PassThroughConverter ，是 MyBatisBatchItemWriter 的内部静态类，实现 Converter 接口，直接返回自身。代码如下：

```
// MyBatisBatchItemWriter.java

private static class PassThroughConverter<T> implements Converter<T, T> {

    @Override
    public T convert(T source) {
        return source;
    }

}
```

- 这是一个默认的 Converter 实现类。我们可以自定义 Converter 实现类，设置到 MyBatisBatchItemWriter 中。那么具体什么用呢？答案见 [「4.5 write」](http://svip.iocoder.cn/MyBatis/Spring-Integration-5/#) 。

#### 4.4 afterPropertiesSet

```
// MyBatisBatchItemWriter

@Override
public void afterPropertiesSet() {
    notNull(sqlSessionTemplate, "A SqlSessionFactory or a SqlSessionTemplate is required.");
    isTrue(ExecutorType.BATCH == sqlSessionTemplate.getExecutorType(), "SqlSessionTemplate's executor type must be BATCH");
    notNull(statementId, "A statementId is required.");
    notNull(itemToParameterConverter, "A itemToParameterConverter is required.");
}
```

#### 4.5 write

`#write(List<? extends T> items)` 方法，将传入的 `items` 数组，执行一次批量操作，仅执行一次批量操作。代码如下：

```
// MyBatisBatchItemWriter.java

@Override
public void write(final List<? extends T> items) {
    if (!items.isEmpty()) {
        LOGGER.debug(() -> "Executing batch with " + items.size() + " items.");

        // <1> 遍历 items 数组，提交到 sqlSessionTemplate 中
        for (T item : items) {
            sqlSessionTemplate.update(statementId, itemToParameterConverter.convert(item));
        }
        // <2> 执行一次批量操作
        List<BatchResult> results = sqlSessionTemplate.flushStatements();

        if (assertUpdates) {
            // 如果有多个返回结果，抛出 InvalidDataAccessResourceUsageException 异常
            if (results.size() != 1) {
                throw new InvalidDataAccessResourceUsageException("Batch execution returned invalid results. " +
                        "Expected 1 but number of BatchResult objects returned was " + results.size());
            }

            // <3> 遍历执行结果，若存在未更新的情况，则抛出 EmptyResultDataAccessException 异常
            int[] updateCounts = results.get(0).getUpdateCounts();
            for (int i = 0; i < updateCounts.length; i++) {
                int value = updateCounts[i];
                if (value == 0) {
                    throw new EmptyResultDataAccessException("Item " + i + " of " + updateCounts.length
                            + " did not update any rows: [" + items.get(i) + "]", 1);
                }
            }
        }
    }
}
```

- ```
  <1>
  ```

   

  处，遍历

   

  ```
  items
  ```

   

  数组，提交到

   

  ```
  sqlSessionTemplate
  ```

   

  中。

  - `<2>` 处，执行一次批量操作。

- `<3>` 处，遍历执行结果，若存在未更新的情况，则抛出 EmptyResultDataAccessException 异常。

## 5. Builder 类

在 `batch/builder` 包下，读取器和写入器都有 Builder 类，胖友自己去瞅瞅，灰常简单。