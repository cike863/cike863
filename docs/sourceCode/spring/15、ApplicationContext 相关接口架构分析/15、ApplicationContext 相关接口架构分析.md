**本文主要基于 Spring 5.0.6.RELEASE**

摘要: 原创出处 http://cmsblogs.com/?p=todo 「小明哥」，谢谢！

作为「小明哥」的忠实读者，「老艿艿」略作修改，记录在理解过程中，参考的资料。

------

﻿在前面 40 篇博客中都是基于 BeanFactory 这个容器来进行分析的，BeanFactory 容器有点儿简单，它并不适用于我们生产环境，在生产环境我们通常会选择 ApplicationContext ，相对于大多数人而言，它才是正规军，相比于 BeanFactory 这个杂牌军而言，它由如下几个区别：

1. 继承 MessageSource，提供国际化的标准访问策略。
2. 继承 ApplicationEventPublisher ，提供强大的事件机制。
3. 扩展 ResourceLoader，可以用来加载多个 Resource，可以灵活访问不同的资源。
4. 对 Web 应用的支持。

# 1. ApplicationContext

下图是 ApplicationContext 结构类图：

[![ApplicationContext 结构类图](http://static.iocoder.cn/3a0321713096156d42661f2df11a93c2)](http://static.iocoder.cn/3a0321713096156d42661f2df11a93c2)ApplicationContext 结构类图

- **BeanFactory**：Spring 管理 Bean 的顶层接口，我们可以认为他是一个简易版的 Spring 容器。ApplicationContext 继承 BeanFactory 的两个子类：HierarchicalBeanFactory 和 ListableBeanFactory。HierarchicalBeanFactory 是一个具有层级关系的 BeanFactory，拥有属性 `parentBeanFactory` 。ListableBeanFactory 实现了枚举方法可以列举出当前 BeanFactory 中所有的 bean 对象而不必根据 name 一个一个的获取。
- **ApplicationEventPublisher**：用于封装事件发布功能的接口，向事件监听器（Listener）发送事件消息。
- **ResourceLoader**：Spring 加载资源的顶层接口，用于从一个源加载资源文件。ApplicationContext 继承 ResourceLoader 的子类 ResourcePatternResolver，该接口是将 location 解析为 Resource 对象的策略接口。
- **MessageSource**：解析 message 的策略接口，用不支撑国际化等功能。
- **EnvironmentCapable**：用于获取 Environment 的接口。

# 2. ApplicationContext 的子接口

ApplicationContext 有两个直接子类：WebApplicationContext 和 ConfigurableApplicationContext 。

## 2.1 WebApplicationContext

```
// WebApplicationContext.java

public interface WebApplicationContext extends ApplicationContext {

    ServletContext getServletContext();

}
```

该接口只有一个 `#getServletContext()` 方法，用于给 Servlet 提供上下文信息。

## 2.2 ConfigurableApplicationContext

```
// ConfigurableApplicationContext.java

public interface ConfigurableApplicationContext extends ApplicationContext, Lifecycle, Closeable {

    // 为 ApplicationContext 设置唯一 ID
    void setId(String id);

    // 为 ApplicationContext 设置 parent
    // 父类不应该被修改：如果创建的对象不可用时，则应该在构造函数外部设置它
    void setParent(@Nullable ApplicationContext parent);

    // 设置 Environment
    void setEnvironment(ConfigurableEnvironment environment);

    // 获取 Environment
    @Override
    ConfigurableEnvironment getEnvironment();

    // 添加 BeanFactoryPostProcessor
    void addBeanFactoryPostProcessor(BeanFactoryPostProcessor postProcessor);

    // 添加 ApplicationListener
    void addApplicationListener(ApplicationListener<?> listener);

    // 添加 ProtocolResolver
    void addProtocolResolver(ProtocolResolver resolver);

    // 加载或者刷新配置
    // 这是一个非常重要的方法
    void refresh() throws BeansException, IllegalStateException;

    // 注册 shutdown hook
    void registerShutdownHook();

    // 关闭 ApplicationContext
    @Override
    void close();

    // ApplicationContext 是否处于激活状态
    boolean isActive();

    // 获取当前上下文的 BeanFactory
    ConfigurableListableBeanFactory getBeanFactory() throws IllegalStateException;

}
```

从上面代码可以看到 ConfigurableApplicationContext 接口提供的方法都是对 ApplicationContext 进行配置的，例如 `#setEnvironment(ConfigurableEnvironment environment)`、`#addBeanFactoryPostProcessor(BeanFactoryPostProcessor postProcessor)`，同时它还继承了如下两个接口：

- Lifecycle：对 context 生命周期的管理，它提供 `#start()` 和 `#stop()` 方法启动和暂停组件。
- Closeable：标准 JDK 所提供的一个接口，用于最后关闭组件释放资源等。

## 2.3 ConfigurableWebApplicationContext

WebApplicationContext 接口和 ConfigurableApplicationContext 接口有一个共同的子类接口 ConfigurableWebApplicationContext，该接口将这两个接口进行合并，提供了一个可配置、可管理、可关闭的 WebApplicationContext ，同时该接口还增加了 `#setServletContext(ServletContext servletContext)`，`setServletConfig(ServletConfig servletConfig)` 等方法，用于装配 WebApplicationContext 。代码如下：

```
// ConfigurableWebApplicationContext.java

public interface ConfigurableWebApplicationContext extends WebApplicationContext, ConfigurableApplicationContext {

    void setServletContext(@Nullable ServletContext servletContext);

    void setServletConfig(@Nullable ServletConfig servletConfig);
    ServletConfig getServletConfig();

    void setNamespace(@Nullable String namespace);
    String getNamespace();

    void setConfigLocation(String configLocation);
    void setConfigLocations(String... configLocations);
    String[] getConfigLocations();

}
```

上面三个接口就可以构成一个比较完整的 Spring 容器，整个 Spring 容器体系涉及的接口较多，所以下面小编就一个具体的实现类来看看 ApplicationContext 的实现（其实在前面一系列的文章中，小编对涉及的大部分接口都已经分析了其原理），当然不可能每个方法都涉及到，但小编会把其中最为重要的实现方法贴出来分析。ApplicationContext 的实现类较多，**就以 ClassPathXmlApplicationContext 来分析 ApplicationContext**。

# 3. ClassPathXmlApplicationContext

ClassPathXmlApplicationContext 是我们在学习 Spring 过程中用的非常多的一个类，很多人第一个接触的 Spring 容器就是它，包括小编自己，下面代码我想很多人依然还记得吧。

```
// 示例
ApplicationContext ac = new ClassPathXmlApplicationContext("applicationContext.xml");
StudentService studentService = (StudentService)ac.getBean("studentService");
```

下图是 ClassPathXmlApplicationContext 的结构类图：

[![ClassPathXmlApplicationContext 的类图](http://static.iocoder.cn/dde0bf4ae9014ec73c80f4c45045850a)](http://static.iocoder.cn/dde0bf4ae9014ec73c80f4c45045850a)ClassPathXmlApplicationContext 的类图

主要的的类层级关系如下：

```
org.springframework.context.support.AbstractApplicationContext
      org.springframework.context.support.AbstractRefreshableApplicationContext
            org.springframework.context.support.AbstractRefreshableConfigApplicationContext
                  org.springframework.context.support.AbstractXmlApplicationContext
                        org.springframework.context.support.ClassPathXmlApplicationContext
```

这种设计是模板方法模式典型的应用，AbstractApplicationContext 实现了 ConfigurableApplicationContext 这个全家桶接口，其子类 AbstractRefreshableConfigApplicationContext 又实现了 BeanNameAware 和 InitializingBean 接口。所以 ClassPathXmlApplicationContext 设计的顶级接口有：

```
BeanFactory：Spring 容器 Bean 的管理
MessageSource：管理 message ，实现国际化等功能
ApplicationEventPublisher：事件发布
ResourcePatternResolver：资源加载
EnvironmentCapable：系统 Environment（profile + Properties） 相关
Lifecycle：管理生命周期
Closable：关闭，释放资源
InitializingBean：自定义初始化
BeanNameAware：设置 beanName 的 Aware 接口
```

下面就这些接口来一一分析。

## 3.1 MessageSource

MessageSource 定义了获取 message 的策略方法 `#getMessage(...)` 。
在 ApplicationContext 体系中，该方法由 AbstractApplicationContext 实现。
在 AbstractApplicationContext 中，它持有一个 MessageSource 实例，将 `#getMessage(...)` 方法委托给该实例来实现，代码如下：

```
// AbstractApplicationContext.java

private MessageSource messageSource;

// 实现 getMessage()
public String getMessage(String code, @Nullable Object[] args, @Nullable String defaultMessage, Locale locale) {
    // 委托给 messageSource 实现
    return getMessageSource().getMessage(code, args, defaultMessage, locale);
}

private MessageSource getMessageSource() throws IllegalStateException {
    if (this.messageSource == null) {
        throw new IllegalStateException("MessageSource not initialized - " + "call 'refresh' before accessing messages via the context: " + this);
    }
    return this.messageSource;
}
```

- 真正实现逻辑，是在 AbstractMessageSource 中，代码如下：

  ```
  // AbstractMessageSource.java
  
  public final String getMessage(String code, @Nullable Object[] args, @Nullable String defaultMessage, Locale locale) {
      String msg = getMessageInternal(code, args, locale);
      if (msg != null) {
          return msg;
      }
      if (defaultMessage == null) {
          return getDefaultMessage(code);
      }
      return renderDefaultMessage(defaultMessage, args, locale);
  }
  ```

  - 具体的实现这里就不分析了，有兴趣的小伙伴可以自己去深入研究。

## 3.2 ApplicationEventPublisher

ApplicationEventPublisher ，用于封装事件发布功能的接口，向事件监听器（Listener）发送事件消息。

该接口提供了一个 `#publishEvent(Object event, ...)` 方法，用于通知在此应用程序中注册的所有的监听器。该方法在 AbstractApplicationContext 中实现。

```
// AbstractApplicationContext.java

@Override
public void publishEvent(ApplicationEvent event) {
    publishEvent(event, null);
}

@Override
public void publishEvent(Object event) {
    publishEvent(event, null);
}

protected void publishEvent(Object event, @Nullable ResolvableType eventType) {
    Assert.notNull(event, "Event must not be null");

    // Decorate event as an ApplicationEvent if necessary
    ApplicationEvent applicationEvent;
    if (event instanceof ApplicationEvent) {
        applicationEvent = (ApplicationEvent) event;
    } else {
        applicationEvent = new PayloadApplicationEvent<>(this, event);
        if (eventType == null) {
            eventType = ((PayloadApplicationEvent) applicationEvent).getResolvableType();
        }
    }

    // Multicast right now if possible - or lazily once the multicaster is initialized
    if (this.earlyApplicationEvents != null) {
        this.earlyApplicationEvents.add(applicationEvent);
    } else {
        getApplicationEventMulticaster().multicastEvent(applicationEvent, eventType);
    }

    // Publish event via parent context as well...
    if (this.parent != null) {
        if (this.parent instanceof AbstractApplicationContext) {
            ((AbstractApplicationContext) this.parent).publishEvent(event, eventType);
        } else {
            this.parent.publishEvent(event);
        }
    }
}
```

- 如果指定的事件不是 ApplicationEvent，则它将包装在PayloadApplicationEvent 中。
- 如果存在父级 ApplicationContext ，则同样要将 event 发布给父级 ApplicationContext 。

## 3.3 ResourcePatternResolver

ResourcePatternResolver 接口继承 ResourceLoader 接口，为将 location 解析为 Resource 对象的策略接口。他提供的 `#getResources(String locationPattern)` 方法，在 AbstractApplicationContext 中实现，在 AbstractApplicationContext 中他持有一个 ResourcePatternResolver 的实例对象。代码如下：

```
// AbstractApplicationContext.java

/** ResourcePatternResolver used by this context. */
private ResourcePatternResolver resourcePatternResolver;

public Resource[] getResources(String locationPattern) throws IOException {
    return this.resourcePatternResolver.getResources(locationPattern);
}
```

- 如果小伙伴对 Spring 的 ResourceLoader 比较熟悉的话，你会发现最终是在 PathMatchingResourcePatternResolver 中实现，该类是 ResourcePatternResolver 接口的实现者。

## 3.4 EnvironmentCapable

提供当前系统环境 Environment 组件。提供了一个 `#getEnvironment()` 方法，用于返回 Environment 实例对象。该方法在 AbstractApplicationContext 实现。代码如下：

```
// AbstractApplicationContext.java
public ConfigurableEnvironment getEnvironment() {
    if (this.environment == null) {
        this.environment = createEnvironment();
    }
    return this.environment;
}
```

- 如果持有的 `environment` 实例对象为空，则调用 `#createEnvironment()` 方法，创建一个。代码如下：

  ```
  // AbstractApplicationContext.java
  
  protected ConfigurableEnvironment createEnvironment() {
      return new StandardEnvironment();
  }
  ```

  - StandardEnvironment 是一个适用于非 WEB 应用的 Environment。

## 3.5 Lifecycle

Lifecycle ，一个用于管理声明周期的接口。

在 AbstractApplicationContext 中存在一个 LifecycleProcessor 类型的实例对象 `lifecycleProcessor` ，AbstractApplicationContext 中关于 Lifecycle 接口的实现都是委托给 `lifecycleProcessor` 实现的。代码如下：

```
// AbstractApplicationContext.java

/** LifecycleProcessor for managing the lifecycle of beans within this context. */
@Nullable
private LifecycleProcessor lifecycleProcessor;

@Override
public void start() {
    getLifecycleProcessor().start();
    publishEvent(new ContextStartedEvent(this));
}

@Override
public void stop() {
    getLifecycleProcessor().stop();
    publishEvent(new ContextStoppedEvent(this));
}

@Override
public boolean isRunning() {
    return (this.lifecycleProcessor != null && this.lifecycleProcessor.isRunning());
}
```

- 在启动、停止的时候会分别发布 ContextStartedEvent 和 ContextStoppedEvent 事件。

## 3.6 Closable

Closable 接口用于关闭和释放资源，提供了 `#close()` 方法，以释放对象所持有的资源。在 ApplicationContext 体系中由AbstractApplicationContext 实现，用于关闭 ApplicationContext 销毁所有 Bean ，此外如果注册有 JVM shutdown hook ，同样要将其移除。代码如下：

```
// AbstractApplicationContext.java

public void close() {
    synchronized (this.startupShutdownMonitor) {
        doClose();
        // If we registered a JVM shutdown hook, we don't need it anymore now:
        // We've already explicitly closed the context.
        if (this.shutdownHook != null) {
            try {
                Runtime.getRuntime().removeShutdownHook(this.shutdownHook);
            } catch (IllegalStateException ex) {
             // ignore - VM is already shutting down
            }
        }
    }
}
```

- 调用 `#doClose()` 方法，发布 ContextClosedEvent 事件，销毁所有 Bean（单例），关闭 BeanFactory 。代码如下：

  ```
  // AbstractApplicationContext.java
  
  protected void doClose() {
      // ... 省略部分代码
      try {
          // Publish shutdown event.
          publishEvent(new ContextClosedEvent(this));
      } catch (Throwable ex) {
          logger.warn("Exception thrown from ApplicationListener handling ContextClosedEvent", ex);
      }
  
      // ... 省略部分代码
      destroyBeans();
      closeBeanFactory();
      onClose();
  
      this.active.set(false);
  
  }
  ```

## 3.7 InitializingBean

InitializingBean 为 Bean 提供了初始化方法的方式，它提供的 `#afterPropertiesSet()` 方法，用于执行初始化动作。在 ApplicationContext 体系中，该方法由 AbstractRefreshableConfigApplicationContext 实现，代码如下：

```
// AbstractRefreshableConfigApplicationContext.java
public void afterPropertiesSet() {
    if (!isActive()) {
        refresh();
    }
}
```

- 执行 `refresh()` 方法，该方法在 AbstractApplicationContext 中执行，执行整个 Spring 容器的初始化过程。**该方法将在下篇文章进行详细分析说明**。

## 3.8 BeanNameAware

BeanNameAware ，设置 Bean Name 的接口。接口在 AbstractRefreshableConfigApplicationContext 中实现。

```
// AbstractRefreshableConfigApplicationContext.java
public void setBeanName(String name) {
    if (!this.setIdCalled) {
        super.setId(name);
        setDisplayName("ApplicationContext '" + name + "'");
    }
}
```



# 4、refresh() 方法。

> 艿艿：我也这么认为，`#refresh()` 方法是关键的关键！

`#refresh()` 方法，是定义在 ConfigurableApplicationContext 类中的，如下：

```
// ConfigurableApplicationContext.java

/**
 * Load or refresh the persistent representation of the configuration,
 * which might an XML file, properties file, or relational database schema.
 * As this is a startup method, it should destroy already created singletons
 * if it fails, to avoid dangling resources. In other words, after invocation
 * of that method, either all or no singletons at all should be instantiated.
 * @throws BeansException if the bean factory could not be initialized
 * @throws IllegalStateException if already initialized and multiple refresh
 * attempts are not supported
 */
void refresh() throws BeansException, IllegalStateException;
```

- 作用就是：**刷新 Spring 的应用上下文**。

其实现是在 AbstractApplicationContext 中实现。如下：

```
// AbstractApplicationContext.java

@Override
public void refresh() throws BeansException, IllegalStateException {
	synchronized (this.startupShutdownMonitor) {
		// 准备刷新上下文环境
		prepareRefresh();

		// 创建并初始化 BeanFactory
		ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();

		// 填充BeanFactory功能
		prepareBeanFactory(beanFactory);

		try {
			// 提供子类覆盖的额外处理，即子类处理自定义的BeanFactoryPostProcess
			postProcessBeanFactory(beanFactory);

			// 激活各种BeanFactory处理器
			invokeBeanFactoryPostProcessors(beanFactory);

			// 注册拦截Bean创建的Bean处理器，即注册 BeanPostProcessor
			registerBeanPostProcessors(beanFactory);

			// 初始化上下文中的资源文件，如国际化文件的处理等
			initMessageSource();

			// 初始化上下文事件广播器
			initApplicationEventMulticaster();

			// 给子类扩展初始化其他Bean
			onRefresh();

			// 在所有bean中查找listener bean，然后注册到广播器中
			registerListeners();

			// 初始化剩下的单例Bean(非延迟加载的)
			finishBeanFactoryInitialization(beanFactory);

			// 完成刷新过程,通知生命周期处理器lifecycleProcessor刷新过程,同时发出ContextRefreshEvent通知别人
			finishRefresh();
		} catch (BeansException ex) {
			if (logger.isWarnEnabled()) {
				logger.warn("Exception encountered during context initialization - " +
						"cancelling refresh attempt: " + ex);
			}

			//  销毁已经创建的Bean
			destroyBeans();

			// 重置容器激活标签
			cancelRefresh(ex);

			// 抛出异常
			throw ex;
		} finally {
			// Reset common introspection caches in Spring's core, since we
			// might not ever need metadata for singleton beans anymore...
			resetCommonCaches();
		}
	}
}
```

这里每一个方法都非常重要，需要一个一个地解释说明。

## 1. prepareRefresh()

> 初始化上下文环境，对系统的环境变量或者系统属性进行准备和校验,如环境变量中必须设置某个值才能运行，否则不能运行，这个时候可以在这里加这个校验，重写 initPropertySources 方法就好了

```
// AbstractApplicationContext.java

protected void prepareRefresh() {
   // 设置启动日期
	this.startupDate = System.currentTimeMillis();
	// 设置 context 当前状态
	this.closed.set(false);
	this.active.set(true);

	if (logger.isInfoEnabled()) {
		logger.info("Refreshing " + this);
	}

	// 初始化context environment（上下文环境）中的占位符属性来源
	initPropertySources();

	// 对属性进行必要的验证
	getEnvironment().validateRequiredProperties();

	this.earlyApplicationEvents = new LinkedHashSet<>();
}
```

该方法主要是做一些准备工作，如：

1. 设置 context 启动时间
2. 设置 context 的当前状态
3. 初始化 context environment 中占位符
4. 对属性进行必要的验证

## 2. obtainFreshBeanFactory()

> 创建并初始化 BeanFactory

```
// AbstractApplicationContext.java

protected ConfigurableListableBeanFactory obtainFreshBeanFactory() {
   // 刷新 BeanFactory
	refreshBeanFactory();
	// 获取 BeanFactory
	ConfigurableListableBeanFactory beanFactory = getBeanFactory();
	if (logger.isDebugEnabled()) {
		logger.debug("Bean factory for " + getDisplayName() + ": " + beanFactory);
	}
	return beanFactory;
}
```

核心方法就在 `#refreshBeanFactory()` 方法，该方法的核心任务就是创建 BeanFactory 并对其就行一番初始化。如下：

```
// AbstractRefreshableApplicationContext.java

@Override
protected final void refreshBeanFactory() throws BeansException {
    // 若已有 BeanFactory ，销毁它的 Bean 们，并销毁 BeanFactory
    if (hasBeanFactory()) {
        destroyBeans();
        closeBeanFactory();
    }
    try {
        // 创建 BeanFactory 对象
        DefaultListableBeanFactory beanFactory = createBeanFactory();
        // 指定序列化编号
        beanFactory.setSerializationId(getId());
        // 定制 BeanFactory 设置相关属性
        customizeBeanFactory(beanFactory);
        // 加载 BeanDefinition 们
        loadBeanDefinitions(beanFactory);
        // 设置 Context 的 BeanFactory
        synchronized (this.beanFactoryMonitor) {
            this.beanFactory = beanFactory;
        }
    } catch (IOException ex) {
        throw new ApplicationContextException("I/O error parsing bean definition source for " + getDisplayName(), ex);
    }
}
```

1. 判断当前容器是否存在一个 BeanFactory，如果存在则对其进行销毁和关闭
2. 调用 `#createBeanFactory()` 方法，创建一个 BeanFactory 实例，其实就是 DefaultListableBeanFactory 。
3. 自定义 BeanFactory
4. 加载 BeanDefinition 。
5. 将创建好的 bean 工厂的引用交给的 context 来管理

上面 5 个步骤，都是比较简单的，但是有必要讲解下第 4 步：加载 BeanDefinition。如果各位看过 【死磕 Spring】系列的话，在刚刚开始分析源码的时候，小编就是以 `BeanDefinitionReader#loadBeanDefinitions(Resource resource)` 方法，作为入口来分析的，示例如下：

```
// 示例代码

ClassPathResource resource = new ClassPathResource("bean.xml");
DefaultListableBeanFactory factory = new DefaultListableBeanFactory();
XmlBeanDefinitionReader reader = new XmlBeanDefinitionReader(factory);
reader.loadBeanDefinitions(resource);
```

只不过这段代码的 `BeanDefinitionReader#loadBeanDefinitions(Resource)` 方法，是定义在 BeanDefinitionReader 中，而此处的 `#loadBeanDefinitions(DefaultListableBeanFactory beanFactory)` 则是定义在 AbstractRefreshableApplicationContext 中，如下：

```
// AbstractRefreshableApplicationContext.java

protected abstract void loadBeanDefinitions(DefaultListableBeanFactory beanFactory) throws BeansException, IOException
```

由具体的子类实现，我们以 AbstractXmlApplicationContext 为例，实现如下：

```
// AbstractXmlApplicationContext.java

@Override
protected void loadBeanDefinitions(DefaultListableBeanFactory beanFactory) throws BeansException, IOException {
    // Create a new XmlBeanDefinitionReader for the given BeanFactory.
    // 创建 XmlBeanDefinitionReader 对象
    XmlBeanDefinitionReader beanDefinitionReader = new XmlBeanDefinitionReader(beanFactory);

    // Configure the bean definition reader with this context's
    // resource loading environment.
    // 对 XmlBeanDefinitionReader 进行环境变量的设置
    beanDefinitionReader.setEnvironment(this.getEnvironment());
    beanDefinitionReader.setResourceLoader(this);
    beanDefinitionReader.setEntityResolver(new ResourceEntityResolver(this));

    // Allow a subclass to provide custom initialization of the reader,
    // then proceed with actually loading the bean definitions.
    // 对 XmlBeanDefinitionReader 进行设置，可以进行覆盖
    initBeanDefinitionReader(beanDefinitionReader);

    // 从 Resource 们中，加载 BeanDefinition 们
    loadBeanDefinitions(beanDefinitionReader);
}
```

- 新建 XmlBeanDefinitionReader 实例对象 beanDefinitionReader，调用 `initBeanDefinitionReader()` 对其进行初始化，然后调用 `loadBeanDefinitions()` 加载 BeanDefinition。代码如下：

  ```
  // AbstractXmlApplicationContext.java
  
  protected void loadBeanDefinitions(XmlBeanDefinitionReader reader) throws BeansException, IOException {
      // 从配置文件 Resource 中，加载 BeanDefinition 们
      Resource[] configResources = getConfigResources();
      if (configResources != null) {
          reader.loadBeanDefinitions(configResources);
      }
      // 从配置文件地址中，加载 BeanDefinition 们
      String[] configLocations = getConfigLocations();
      if (configLocations != null) {
          reader.loadBeanDefinitions(configLocations);
      }
  }
  ```

  - 到这里我们发现，其实内部依然是调用 `BeanDefinitionReader#loadBeanDefinitionn()` 进行 BeanDefinition 的加载进程。

## 3. prepareBeanFactory(beanFactory)

> 填充 BeanFactory 功能

上面获取获取的 BeanFactory 除了加载了一些 BeanDefinition 就没有其他任何东西了，这个时候其实还不能投入生产，因为还少配置了一些东西，比如 context的 ClassLoader 和 后置处理器等等。

```
// AbstractApplicationContext.java

protected void prepareBeanFactory(ConfigurableListableBeanFactory beanFactory) {
	// 设置beanFactory的classLoader
	beanFactory.setBeanClassLoader(getClassLoader());

	// 设置beanFactory的表达式语言处理器,Spring3开始增加了对语言表达式的支持,默认可以使用#{bean.xxx}的形式来调用相关属性值
	beanFactory.setBeanExpressionResolver(new StandardBeanExpressionResolver(beanFactory.getBeanClassLoader()));
	// 为beanFactory增加一个默认的propertyEditor
	beanFactory.addPropertyEditorRegistrar(new ResourceEditorRegistrar(this, getEnvironment()));

	// 添加ApplicationContextAwareProcessor
	beanFactory.addBeanPostProcessor(new ApplicationContextAwareProcessor(this));
	// 设置忽略自动装配的接口
    beanFactory.ignoreDependencyInterface(EnvironmentAware.class);
	beanFactory.ignoreDependencyInterface(EmbeddedValueResolverAware.class);
	beanFactory.ignoreDependencyInterface(ResourceLoaderAware.class);
	beanFactory.ignoreDependencyInterface(ApplicationEventPublisherAware.class);
	beanFactory.ignoreDependencyInterface(MessageSourceAware.class);
	beanFactory.ignoreDependencyInterface(ApplicationContextAware.class);

	// 设置几个自动装配的特殊规则
	beanFactory.registerResolvableDependency(BeanFactory.class, beanFactory);
	beanFactory.registerResolvableDependency(ResourceLoader.class, this);
	beanFactory.registerResolvableDependency(ApplicationEventPublisher.class, this);
	beanFactory.registerResolvableDependency(ApplicationContext.class, this);

	// Register early post-processor for detecting inner beans as ApplicationListeners.
	beanFactory.addBeanPostProcessor(new ApplicationListenerDetector(this));

	// 增加对AspectJ的支持
	if (beanFactory.containsBean(LOAD_TIME_WEAVER_BEAN_NAME)) {
		beanFactory.addBeanPostProcessor(new LoadTimeWeaverAwareProcessor(beanFactory));
		// Set a temporary ClassLoader for type matching.
		beanFactory.setTempClassLoader(new ContextTypeMatchClassLoader(beanFactory.getBeanClassLoader()));
	}

	// 注册默认的系统环境bean
	if (!beanFactory.containsLocalBean(ENVIRONMENT_BEAN_NAME)) {
		beanFactory.registerSingleton(ENVIRONMENT_BEAN_NAME, getEnvironment());
	}
	if (!beanFactory.containsLocalBean(SYSTEM_PROPERTIES_BEAN_NAME)) {
		beanFactory.registerSingleton(SYSTEM_PROPERTIES_BEAN_NAME, getEnvironment().getSystemProperties());
	}
	if (!beanFactory.containsLocalBean(SYSTEM_ENVIRONMENT_BEAN_NAME)) {
		beanFactory.registerSingleton(SYSTEM_ENVIRONMENT_BEAN_NAME, getEnvironment().getSystemEnvironment());
	}
}
```

看上面的源码知道这个就是对 BeanFactory 设置各种各种的功能。

## 4. postProcessBeanFactory()

> 提供子类覆盖的额外处理，即子类处理自定义的BeanFactoryPostProcess

```
// AbstractApplicationContext.java

protected void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) {
	beanFactory.addBeanPostProcessor(new ServletContextAwareProcessor(this.servletContext, this.servletConfig));
	beanFactory.ignoreDependencyInterface(ServletContextAware.class);
	beanFactory.ignoreDependencyInterface(ServletConfigAware.class);

	WebApplicationContextUtils.registerWebApplicationScopes(beanFactory, this.servletContext);
	WebApplicationContextUtils.registerEnvironmentBeans(beanFactory, this.servletContext, this.servletConfig);
}
```

1. 添加 ServletContextAwareProcessor 到 BeanFactory 容器中，该 processor 实现 BeanPostProcessor 接口，主要用于将ServletContext 传递给实现了 ServletContextAware 接口的 bean
2. 忽略 ServletContextAware、ServletConfigAware
3. 注册 WEB 应用特定的域（scope）到 beanFactory 中，以便 WebApplicationContext 可以使用它们。比如 “request” , “session” , “globalSession” , “application”
4. 注册 WEB 应用特定的 Environment bean 到 beanFactory 中，以便WebApplicationContext 可以使用它们。如：”contextParameters”, “contextAttributes”

## 5. invokeBeanFactoryPostProcessors()

> 激活各种BeanFactory处理器

```
// AbstractApplicationContext.java

public static void invokeBeanFactoryPostProcessors(
		ConfigurableListableBeanFactory beanFactory, List<BeanFactoryPostProcessor> beanFactoryPostProcessors) {

	// 定义一个 set 保存所有的 BeanFactoryPostProcessors
	Set<String> processedBeans = new HashSet<>();

	// 如果当前 BeanFactory 为 BeanDefinitionRegistry
	if (beanFactory instanceof BeanDefinitionRegistry) {
		BeanDefinitionRegistry registry = (BeanDefinitionRegistry) beanFactory;
		// BeanFactoryPostProcessor 集合
		List<BeanFactoryPostProcessor> regularPostProcessors = new ArrayList<>();
		// BeanDefinitionRegistryPostProcessor 集合
		List<BeanDefinitionRegistryPostProcessor> registryProcessors = new ArrayList<>();

		// 迭代注册的 beanFactoryPostProcessors
		for (BeanFactoryPostProcessor postProcessor : beanFactoryPostProcessors) {
			// 如果是 BeanDefinitionRegistryPostProcessor，则调用 postProcessBeanDefinitionRegistry 进行注册，
			// 同时加入到 registryProcessors 集合中
			if (postProcessor instanceof BeanDefinitionRegistryPostProcessor) {
				BeanDefinitionRegistryPostProcessor registryProcessor =
						(BeanDefinitionRegistryPostProcessor) postProcessor;
				registryProcessor.postProcessBeanDefinitionRegistry(registry);
				registryProcessors.add(registryProcessor);
			}
			else {
				// 否则当做普通的 BeanFactoryPostProcessor 处理
				// 添加到 regularPostProcessors 集合中即可，便于后面做后续处理
				regularPostProcessors.add(postProcessor);
			}
		}

		// 用于保存当前处理的 BeanDefinitionRegistryPostProcessor
		List<BeanDefinitionRegistryPostProcessor> currentRegistryProcessors = new ArrayList<>();

		// 首先处理实现了 PriorityOrdered (有限排序接口)的 BeanDefinitionRegistryPostProcessor
		String[] postProcessorNames =
				beanFactory.getBeanNamesForType(BeanDefinitionRegistryPostProcessor.class, true, false);
		for (String ppName : postProcessorNames) {
			if (beanFactory.isTypeMatch(ppName, PriorityOrdered.class)) {
				currentRegistryProcessors.add(beanFactory.getBean(ppName, BeanDefinitionRegistryPostProcessor.class));
				processedBeans.add(ppName);
			}
		}

		// 排序
		sortPostProcessors(currentRegistryProcessors, beanFactory);

		// 加入registryProcessors集合
		registryProcessors.addAll(currentRegistryProcessors);

		// 调用所有实现了 PriorityOrdered 的 BeanDefinitionRegistryPostProcessors 的 postProcessBeanDefinitionRegistry()
		invokeBeanDefinitionRegistryPostProcessors(currentRegistryProcessors, registry);

		// 清空，以备下次使用
		currentRegistryProcessors.clear();

		// 其次，调用是实现了 Ordered（普通排序接口）的 BeanDefinitionRegistryPostProcessors
		// 逻辑和 上面一样
		postProcessorNames = beanFactory.getBeanNamesForType(BeanDefinitionRegistryPostProcessor.class, true, false);
		for (String ppName : postProcessorNames) {
			if (!processedBeans.contains(ppName) && beanFactory.isTypeMatch(ppName, Ordered.class)) {
				currentRegistryProcessors.add(beanFactory.getBean(ppName, BeanDefinitionRegistryPostProcessor.class));
				processedBeans.add(ppName);
			}
		}
		sortPostProcessors(currentRegistryProcessors, beanFactory);
		registryProcessors.addAll(currentRegistryProcessors);
		invokeBeanDefinitionRegistryPostProcessors(currentRegistryProcessors, registry);
		currentRegistryProcessors.clear();

		// 最后调用其他的 BeanDefinitionRegistryPostProcessors
		boolean reiterate = true;
		while (reiterate) {
			reiterate = false;
			// 获取 BeanDefinitionRegistryPostProcessor
			postProcessorNames = beanFactory.getBeanNamesForType(BeanDefinitionRegistryPostProcessor.class, true, false);
			for (String ppName : postProcessorNames) {

				// 没有包含在 processedBeans 中的（因为包含了的都已经处理了）
				if (!processedBeans.contains(ppName)) {
					currentRegistryProcessors.add(beanFactory.getBean(ppName, BeanDefinitionRegistryPostProcessor.class));
					processedBeans.add(ppName);
					reiterate = true;
				}
			}

			// 与上面处理逻辑一致
			sortPostProcessors(currentRegistryProcessors, beanFactory);
			registryProcessors.addAll(currentRegistryProcessors);
			invokeBeanDefinitionRegistryPostProcessors(currentRegistryProcessors, registry);
			currentRegistryProcessors.clear();
		}

		// 调用所有 BeanDefinitionRegistryPostProcessor (包括手动注册和通过配置文件注册)
		// 和 BeanFactoryPostProcessor(只有手动注册)的回调函数(postProcessBeanFactory())
		invokeBeanFactoryPostProcessors(registryProcessors, beanFactory);
		invokeBeanFactoryPostProcessors(regularPostProcessors, beanFactory);
	}

	else {
		// 如果不是 BeanDefinitionRegistry 只需要调用其回调函数（postProcessBeanFactory()）即可
		invokeBeanFactoryPostProcessors(beanFactoryPostProcessors, beanFactory);
	}

	//
	String[] postProcessorNames =
			beanFactory.getBeanNamesForType(BeanFactoryPostProcessor.class, true, false);

	// 这里同样需要区分 PriorityOrdered 、Ordered 和 no Ordered
	List<BeanFactoryPostProcessor> priorityOrderedPostProcessors = new ArrayList<>();
	List<String> orderedPostProcessorNames = new ArrayList<>();
	List<String> nonOrderedPostProcessorNames = new ArrayList<>();
	for (String ppName : postProcessorNames) {
		// 已经处理过了的，跳过
		if (processedBeans.contains(ppName)) {
			// skip - already processed in first phase above
		}
		// PriorityOrdered
		else if (beanFactory.isTypeMatch(ppName, PriorityOrdered.class)) {
			priorityOrderedPostProcessors.add(beanFactory.getBean(ppName, BeanFactoryPostProcessor.class));
		}
		// Ordered
		else if (beanFactory.isTypeMatch(ppName, Ordered.class)) {
			orderedPostProcessorNames.add(ppName);
		}
		// no Ordered
		else {
			nonOrderedPostProcessorNames.add(ppName);
		}
	}

	// First, PriorityOrdered 接口
	sortPostProcessors(priorityOrderedPostProcessors, beanFactory);
	invokeBeanFactoryPostProcessors(priorityOrderedPostProcessors, beanFactory);

	// Next, Ordered 接口
	List<BeanFactoryPostProcessor> orderedPostProcessors = new ArrayList<>();
	for (String postProcessorName : orderedPostProcessorNames) {
		orderedPostProcessors.add(beanFactory.getBean(postProcessorName, BeanFactoryPostProcessor.class));
	}
	sortPostProcessors(orderedPostProcessors, beanFactory);
	invokeBeanFactoryPostProcessors(orderedPostProcessors, beanFactory);

	// Finally, no ordered
	List<BeanFactoryPostProcessor> nonOrderedPostProcessors = new ArrayList<>();
	for (String postProcessorName : nonOrderedPostProcessorNames) {
		nonOrderedPostProcessors.add(beanFactory.getBean(postProcessorName, BeanFactoryPostProcessor.class));
	}
	invokeBeanFactoryPostProcessors(nonOrderedPostProcessors, beanFactory);

	// Clear cached merged bean definitions since the post-processors might have
	// modified the original metadata, e.g. replacing placeholders in values...
	beanFactory.clearMetadataCache();
}
```

上述代码较长，但是处理逻辑较为单一，就是对所有的 BeanDefinitionRegistryPostProcessors 、手动注册的 BeanFactoryPostProcessor 以及通过配置文件方式的 BeanFactoryPostProcessor 按照 PriorityOrdered 、 Ordered、no ordered 三种方式分开处理、调用。

## 6. registerBeanPostProcessors

> 注册拦截Bean创建的Bean处理器，即注册 BeanPostProcessor

与 BeanFactoryPostProcessor 一样，也是委托给 PostProcessorRegistrationDelegate 来实现的。

```
// AbstractApplicationContext.java

public static void registerBeanPostProcessors(
		ConfigurableListableBeanFactory beanFactory, AbstractApplicationContext applicationContext) {

	// 所有的 BeanPostProcessors
	String[] postProcessorNames = beanFactory.getBeanNamesForType(BeanPostProcessor.class, true, false);

	// 注册 BeanPostProcessorChecker
	// 主要用于记录一些 bean 的信息，这些 bean 不符合所有 BeanPostProcessors 处理的资格时
	int beanProcessorTargetCount = beanFactory.getBeanPostProcessorCount() + 1 + postProcessorNames.length;
	beanFactory.addBeanPostProcessor(new BeanPostProcessorChecker(beanFactory, beanProcessorTargetCount));

	// 区分 PriorityOrdered、Ordered 、 no ordered
	List<BeanPostProcessor> priorityOrderedPostProcessors = new ArrayList<>();
	List<String> orderedPostProcessorNames = new ArrayList<>();
	List<String> nonOrderedPostProcessorNames = new ArrayList<>();
	// MergedBeanDefinition
	List<BeanPostProcessor> internalPostProcessors = new ArrayList<>();
	for (String ppName : postProcessorNames) {
		if (beanFactory.isTypeMatch(ppName, PriorityOrdered.class)) {
			BeanPostProcessor pp = beanFactory.getBean(ppName, BeanPostProcessor.class);
			priorityOrderedPostProcessors.add(pp);
			if (pp instanceof MergedBeanDefinitionPostProcessor) {
				internalPostProcessors.add(pp);
			}
		}
		else if (beanFactory.isTypeMatch(ppName, Ordered.class)) {
			orderedPostProcessorNames.add(ppName);
		}
		else {
			nonOrderedPostProcessorNames.add(ppName);
		}
	}

	// First, PriorityOrdered
	sortPostProcessors(priorityOrderedPostProcessors, beanFactory);
	registerBeanPostProcessors(beanFactory, priorityOrderedPostProcessors);

	// Next, Ordered
	List<BeanPostProcessor> orderedPostProcessors = new ArrayList<>();
	for (String ppName : orderedPostProcessorNames) {
		BeanPostProcessor pp = beanFactory.getBean(ppName, BeanPostProcessor.class);
		orderedPostProcessors.add(pp);
		if (pp instanceof MergedBeanDefinitionPostProcessor) {
			internalPostProcessors.add(pp);
		}
	}
	sortPostProcessors(orderedPostProcessors, beanFactory);
	registerBeanPostProcessors(beanFactory, orderedPostProcessors);

	// onOrdered
	List<BeanPostProcessor> nonOrderedPostProcessors = new ArrayList<>();
	for (String ppName : nonOrderedPostProcessorNames) {
		BeanPostProcessor pp = beanFactory.getBean(ppName, BeanPostProcessor.class);
		nonOrderedPostProcessors.add(pp);
		if (pp instanceof MergedBeanDefinitionPostProcessor) {
			internalPostProcessors.add(pp);
		}
	}
	registerBeanPostProcessors(beanFactory, nonOrderedPostProcessors);

	// Finally, all internal BeanPostProcessors.
	sortPostProcessors(internalPostProcessors, beanFactory);
	registerBeanPostProcessors(beanFactory, internalPostProcessors);

	// 重新注册用来自动探测内部ApplicationListener的post-processor，这样可以将他们移到处理器链条的末尾
	beanFactory.addBeanPostProcessor(new ApplicationListenerDetector(applicationContext));
}
```

## 7. initMessageSource

> 初始化上下文中的资源文件，如国际化文件的处理等

其实该方法就是初始化一个 MessageSource 接口的实现类，主要用于国际化/i18n。

```
// AbstractApplicationContext.java

protected void initMessageSource() {
	ConfigurableListableBeanFactory beanFactory = getBeanFactory();
	// 包含 “messageSource” bean
	if (beanFactory.containsLocalBean(MESSAGE_SOURCE_BEAN_NAME)) {
		this.messageSource = beanFactory.getBean(MESSAGE_SOURCE_BEAN_NAME, MessageSource.class);
		// 如果有父类
		// HierarchicalMessageSource 分级处理的 MessageSource
		if (this.parent != null && this.messageSource instanceof HierarchicalMessageSource) {
			HierarchicalMessageSource hms = (HierarchicalMessageSource) this.messageSource;
			if (hms.getParentMessageSource() == null) {
				// 如果没有注册父 MessageSource，则设置为父类上下文的的 MessageSource
				hms.setParentMessageSource(getInternalParentMessageSource());
			}
		}
		if (logger.isDebugEnabled()) {
			logger.debug("Using MessageSource [" + this.messageSource + "]");
		}
	}
	else {
		// 使用 空 MessageSource
		DelegatingMessageSource dms = new DelegatingMessageSource();
		dms.setParentMessageSource(getInternalParentMessageSource());
		this.messageSource = dms;
		beanFactory.registerSingleton(MESSAGE_SOURCE_BEAN_NAME, this.messageSource);
		if (logger.isDebugEnabled()) {
			logger.debug("Unable to locate MessageSource with name '" + MESSAGE_SOURCE_BEAN_NAME +
					"': using default [" + this.messageSource + "]");
		}
	}
}
```

## 8. initApplicationEventMulticaster

> 初始化上下文事件广播器

```
// AbstractApplicationContext.java

protected void initApplicationEventMulticaster() {
	ConfigurableListableBeanFactory beanFactory = getBeanFactory();

	// 如果存在 applicationEventMulticaster bean，则获取赋值
	if (beanFactory.containsLocalBean(APPLICATION_EVENT_MULTICASTER_BEAN_NAME)) {
		this.applicationEventMulticaster =
				beanFactory.getBean(APPLICATION_EVENT_MULTICASTER_BEAN_NAME, ApplicationEventMulticaster.class);
		if (logger.isDebugEnabled()) {
			logger.debug("Using ApplicationEventMulticaster [" + this.applicationEventMulticaster + "]");
		}
	}
	else {
		// 没有则新建 SimpleApplicationEventMulticaster，并完成 bean 的注册
		this.applicationEventMulticaster = new SimpleApplicationEventMulticaster(beanFactory);
		beanFactory.registerSingleton(APPLICATION_EVENT_MULTICASTER_BEAN_NAME, this.applicationEventMulticaster);
		if (logger.isDebugEnabled()) {
			logger.debug("Unable to locate ApplicationEventMulticaster with name '" +
					APPLICATION_EVENT_MULTICASTER_BEAN_NAME +
					"': using default [" + this.applicationEventMulticaster + "]");
		}
	}
}
```

如果当前容器中存在 applicationEventMulticaster 的 bean，则对 applicationEventMulticaster 赋值，否则新建一个 SimpleApplicationEventMulticaster 的对象（默认的），并完成注册。

## 9. onRefresh

> 给子类扩展初始化其他Bean

预留给 AbstractApplicationContext 的子类用于初始化其他特殊的 bean，该方法需要在所有单例 bean 初始化之前调用。

## 10. registerListeners

> 在所有 bean 中查找 listener bean，然后注册到广播器中

```
// AbstractApplicationContext.java

protected void registerListeners() {
	// 注册静态 监听器
	for (ApplicationListener<?> listener : getApplicationListeners()) {
		getApplicationEventMulticaster().addApplicationListener(listener);
	}

	String[] listenerBeanNames = getBeanNamesForType(ApplicationListener.class, true, false);
	for (String listenerBeanName : listenerBeanNames) {
		getApplicationEventMulticaster().addApplicationListenerBean(listenerBeanName);
	}

	// 至此，已经完成将监听器注册到ApplicationEventMulticaster中，下面将发布前期的事件给监听器。
	Set<ApplicationEvent> earlyEventsToProcess = this.earlyApplicationEvents;
	this.earlyApplicationEvents = null;
	if (earlyEventsToProcess != null) {
		for (ApplicationEvent earlyEvent : earlyEventsToProcess) {
			getApplicationEventMulticaster().multicastEvent(earlyEvent);
		}
	}
}
```

## 11. finishBeanFactoryInitialization

> 初始化剩下的单例Bean(非延迟加载的)

```
// AbstractApplicationContext.java

protected void finishBeanFactoryInitialization(ConfigurableListableBeanFactory beanFactory) {
	// 初始化转换器
	if (beanFactory.containsBean(CONVERSION_SERVICE_BEAN_NAME) &&
			beanFactory.isTypeMatch(CONVERSION_SERVICE_BEAN_NAME, ConversionService.class)) {
		beanFactory.setConversionService(
				beanFactory.getBean(CONVERSION_SERVICE_BEAN_NAME, ConversionService.class));
	}

	// 如果之前没有注册 bean 后置处理器（例如PropertyPlaceholderConfigurer），则注册默认的解析器
	if (!beanFactory.hasEmbeddedValueResolver()) {
		beanFactory.addEmbeddedValueResolver(strVal -> getEnvironment().resolvePlaceholders(strVal));
	}

	// 初始化 Initialize LoadTimeWeaverAware beans early to allow for registering their transformers early.
	String[] weaverAwareNames = beanFactory.getBeanNamesForType(LoadTimeWeaverAware.class, false, false);
	for (String weaverAwareName : weaverAwareNames) {
		getBean(weaverAwareName);
	}

	// 停止使用临时的 ClassLoader
	beanFactory.setTempClassLoader(null);

	//
	beanFactory.freezeConfiguration();

	// 初始化所有剩余的单例（非延迟初始化）
	beanFactory.preInstantiateSingletons();
}
```

## 12. finishRefresh

> 完成刷新过程,通知生命周期处理器 lifecycleProcessor 刷新过程,同时发出 ContextRefreshEvent 通知别人

主要是调用 `LifecycleProcessor#onRefresh()` ，并且发布事件（ContextRefreshedEvent）。

```
// AbstractApplicationContext.java
	
protected void finishRefresh() {
	// Clear context-level resource caches (such as ASM metadata from scanning).
	clearResourceCaches();

	// Initialize lifecycle processor for this context.
	initLifecycleProcessor();

	// Propagate refresh to lifecycle processor first.
	getLifecycleProcessor().onRefresh();

	// Publish the final event.
	publishEvent(new ContextRefreshedEvent(this));

	// Participate in LiveBeansView MBean, if active.
	LiveBeansView.registerApplicationContext(this);
}
```



# 5. 小结

由于篇幅问题，再加上大部分接口小编都已经在前面文章进行了详细的阐述，所以本文主要是以 Spring Framework 的 ApplicationContext 为中心，对其结构和功能的实现进行了简要的说明。

这里不得不说 Spring 真的是一个非常优秀的框架，具有良好的结构设计和接口抽象，它的每一个接口职能单一，且都是具体功能到各个模块的高度抽象，且几乎每套接口都提供了一个默认的实现（defaultXXX）。

对于 ApplicationContext 体系而言，他继承 Spring 中众多的核心接口，能够为客户端提供一个相对完整的 Spring 容器，接口 ConfigurableApplicationContext 对 ApplicationContext 接口再次进行扩展，提供了生命周期的管理功能。
抽象类 ApplicationContext 对整套接口提供了大部分的默认实现，将其中“不易变动”的部分进行了封装，通过“组合”的方式将“容易变动”的功能委托给其他类来实现，同时利用模板方法模式将一些方法的实现开放出去由子类实现，从而实现“**对扩展开放，对修改封闭**”的设计原则。

最后我们再来领略下图的风采：

[![ClassPathXmlApplicationContext 的类图](http://static.iocoder.cn/dde0bf4ae9014ec73c80f4c45045850a)](http://static.iocoder.cn/dde0bf4ae9014ec73c80f4c45045850a)ClassPathXmlApplicationContext 的类图