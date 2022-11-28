**本文主要基于 Spring 5.0.6.RELEASE**

摘要: 原创出处 http://cmsblogs.com/?p=2763 「小明哥」，谢谢！

作为「小明哥」的忠实读者，「老艿艿」略作修改，记录在理解过程中，参考的资料。

------

DefaultBeanDefinitionDocumentReader 的 ﻿`#processBeanDefinition()` 方法，完成 Bean 标签解析的核心工作。代码如下：

```
// DefaultBeanDefinitionDocumentReader.java

protected void processBeanDefinition(Element ele, BeanDefinitionParserDelegate delegate) {
    // 进行 bean 元素解析。
    // 如果解析成功，则返回 BeanDefinitionHolder 对象。而 BeanDefinitionHolder 为 name 和 alias 的 BeanDefinition 对象
    // 如果解析失败，则返回 null 。
    BeanDefinitionHolder bdHolder = delegate.parseBeanDefinitionElement(ele);
    if (bdHolder != null) {
        // 进行自定义标签处理
        bdHolder = delegate.decorateBeanDefinitionIfRequired(ele, bdHolder);
        try {
            // 进行 BeanDefinition 的注册
            // Register the final decorated instance.
            BeanDefinitionReaderUtils.registerBeanDefinition(bdHolder, getReaderContext().getRegistry());
        } catch (BeanDefinitionStoreException ex) {
            getReaderContext().error("Failed to register bean definition with name '" +
                    bdHolder.getBeanName() + "'", ele, ex);
        }
        // 发出响应事件，通知相关的监听器，已完成该 Bean 标签的解析。
        // Send registration event.
        getReaderContext().fireComponentRegistered(new BeanComponentDefinition(bdHolder));
    }
}
```

- 解析工作分为三步：
  - 1、解析默认标签。
  - 2、解析默认标签后下得自定义标签。
  - 3、注册解析后的 BeanDefinition 。
- 经过前面两个步骤的解析，这时的 BeanDefinition 已经可以满足后续的使用要求了，**那么接下来的工作就是将这些 BeanDefinition 进行注册，也就是完成第三步**。

# 1. BeanDefinitionReaderUtils

注册 BeanDefinition ，由 `BeanDefinitionReaderUtils.registerBeanDefinition()` 完成。代码如下：

```
// BeanDefinitionReaderUtils.java
 
public static void registerBeanDefinition(
        BeanDefinitionHolder definitionHolder, BeanDefinitionRegistry registry)
        throws BeanDefinitionStoreException {

    // 注册 beanName
    // Register bean definition under primary name.
    String beanName = definitionHolder.getBeanName();
    registry.registerBeanDefinition(beanName, definitionHolder.getBeanDefinition());

    // 注册 alias
    // Register aliases for bean name, if any.
    String[] aliases = definitionHolder.getAliases();
    if (aliases != null) {
        for (String alias : aliases) {
            registry.registerAlias(beanName, alias);
        }
    }
}
```

- 首先，通过 `beanName` 注册 BeanDefinition 。详细解析，见 [2.1 通过 beanName 注册](http://svip.iocoder.cn/Spring/IoC-register-BeanDefinitions-really/#) 。
- 然后，再通过注册别名 `alias` 和 `beanName` 的映射。详细解析，见 [2.2 注册 alias 和 beanName 的映射](http://svip.iocoder.cn/Spring/IoC-register-BeanDefinitions-really/#) 。

# 2. BeanDefinitionRegistry

BeanDefinition 的注册，由接口 `org.springframework.beans.factory.support.BeanDefinitionRegistry` 定义。

## 2.1 通过 beanName 注册

调用 BeanDefinitionRegistry 的 `#registerBeanDefinition(String beanName, BeanDefinition beanDefinition)` 方法，实现通过 `beanName` 注册 BeanDefinition 。代码如下：

```
// DefaultListableBeanFactory.java

/** Whether to allow re-registration of a different definition with the same name. */
private boolean allowBeanDefinitionOverriding = true;

/** Map of bean definition objects, keyed by bean name. */
private final Map<String, BeanDefinition> beanDefinitionMap = new ConcurrentHashMap<>(256);
/** List of bean definition names, in registration order. */
private volatile List<String> beanDefinitionNames = new ArrayList<>(256);
/** List of names of manually registered singletons, in registration order. */
private volatile Set<String> manualSingletonNames = new LinkedHashSet<>(16);
/** Cached array of bean definition names in case of frozen configuration. */
@Nullable
private volatile String[] frozenBeanDefinitionNames;

@Override
public void registerBeanDefinition(String beanName, BeanDefinition beanDefinition)
        throws BeanDefinitionStoreException {

    // 校验 beanName 与 beanDefinition 非空
    Assert.hasText(beanName, "Bean name must not be empty");
    Assert.notNull(beanDefinition, "BeanDefinition must not be null");

    // <1> 校验 BeanDefinition 。
    // 这是注册前的最后一次校验了，主要是对属性 methodOverrides 进行校验。
    if (beanDefinition instanceof AbstractBeanDefinition) {
        try {
            ((AbstractBeanDefinition) beanDefinition).validate();
        } catch (BeanDefinitionValidationException ex) {
            throw new BeanDefinitionStoreException(beanDefinition.getResourceDescription(), beanName,
                    "Validation of bean definition failed", ex);
        }
    }

    // <2> 从缓存中获取指定 beanName 的 BeanDefinition
    BeanDefinition existingDefinition = this.beanDefinitionMap.get(beanName);
    // <3> 如果已经存在
    if (existingDefinition != null) {
        // 如果存在但是不允许覆盖，抛出异常
        if (!isAllowBeanDefinitionOverriding()) {
            throw new BeanDefinitionOverrideException(beanName, beanDefinition, existingDefinition);
        // 覆盖 beanDefinition 大于 被覆盖的 beanDefinition 的 ROLE ，打印 info 日志
        } else if (existingDefinition.getRole() < beanDefinition.getRole()) {
            // e.g. was ROLE_APPLICATION, now overriding with ROLE_SUPPORT or ROLE_INFRASTRUCTURE
            if (logger.isInfoEnabled()) {
                logger.info("Overriding user-defined bean definition for bean '" + beanName +
                        "' with a framework-generated bean definition: replacing [" +
                        existingDefinition + "] with [" + beanDefinition + "]");
            }
        // 覆盖 beanDefinition 与 被覆盖的 beanDefinition 不相同，打印 debug 日志
        } else if (!beanDefinition.equals(existingDefinition)) {
            if (logger.isDebugEnabled()) {
                logger.debug("Overriding bean definition for bean '" + beanName +
                        "' with a different definition: replacing [" + existingDefinition +
                        "] with [" + beanDefinition + "]");
            }
        // 其它，打印 debug 日志
        } else {
            if (logger.isTraceEnabled()) {
                logger.trace("Overriding bean definition for bean '" + beanName +
                        "' with an equivalent definition: replacing [" + existingDefinition +
                        "] with [" + beanDefinition + "]");
            }
        }
        // 允许覆盖，直接覆盖原有的 BeanDefinition 到 beanDefinitionMap 中。
        this.beanDefinitionMap.put(beanName, beanDefinition);
    // <4> 如果未存在
    } else {
        // 检测创建 Bean 阶段是否已经开启，如果开启了则需要对 beanDefinitionMap 进行并发控制
        if (hasBeanCreationStarted()) {
            // beanDefinitionMap 为全局变量，避免并发情况
            // Cannot modify startup-time collection elements anymore (for stable iteration)
            synchronized (this.beanDefinitionMap) {
                // 添加到 BeanDefinition 到 beanDefinitionMap 中。
                this.beanDefinitionMap.put(beanName, beanDefinition);
                // 添加 beanName 到 beanDefinitionNames 中
                List<String> updatedDefinitions = new ArrayList<>(this.beanDefinitionNames.size() + 1);
                updatedDefinitions.addAll(this.beanDefinitionNames);
                updatedDefinitions.add(beanName);
                this.beanDefinitionNames = updatedDefinitions;
                // 从 manualSingletonNames 移除 beanName
                if (this.manualSingletonNames.contains(beanName)) {
                    Set<String> updatedSingletons = new LinkedHashSet<>(this.manualSingletonNames);
                    updatedSingletons.remove(beanName);
                    this.manualSingletonNames = updatedSingletons;
                }
            }
        } else {
            // Still in startup registration phase
            // 添加到 BeanDefinition 到 beanDefinitionMap 中。
            this.beanDefinitionMap.put(beanName, beanDefinition);
            // 添加 beanName 到 beanDefinitionNames 中
            this.beanDefinitionNames.add(beanName);
            // 从 manualSingletonNames 移除 beanName
            this.manualSingletonNames.remove(beanName);
        }
        
        this.frozenBeanDefinitionNames = null;
    }

    // <5> 重新设置 beanName 对应的缓存
    if (existingDefinition != null || containsSingleton(beanName)) {
        resetBeanDefinition(beanName);
    }
}
```

处理过程如下：

- `<1>` 对 BeanDefinition 进行校验，该校验也是注册过程中的最后一次校验了，主要是对 AbstractBeanDefinition 的 `methodOverrides` 属性进行校验。

- `<2>` 根据 `beanName` 从缓存中获取 BeanDefinition 对象。

- `<3>` 如果缓存中存在，则根据 `allowBeanDefinitionOverriding` 标志来判断是否允许覆盖。如果允许则直接覆盖。否则，抛出 BeanDefinitionStoreException 异常。

- ```
  <4>
  ```

   

  若缓存中没有指定

   

  ```
  beanName
  ```

   

  的 BeanDefinition，则判断当前阶段是否已经开始了 Bean 的创建阶段？如果是，则需要对 beanDefinitionMap 进行加锁控制并发问题，否则直接设置即可。

  - 对于 `#hasBeanCreationStarted()` 方法，后续做详细介绍，这里不过多阐述。

- `<5>` 若缓存中存在该 `beanName` 或者单例 bean 集合中存在该 `beanName` ，则调用 `#resetBeanDefinition(String beanName)` 方法，重置 BeanDefinition 缓存。

😈 其实整段代码的核心就在于 `this.beanDefinitionMap.put(beanName, beanDefinition);` 代码块。而 BeanDefinition 的缓存也不是神奇的东西，就是定义一个 Map ：

- `key` 为 `beanName` 。
- `value` 为 BeanDefinition 对象。

## 2.2 注册 alias 和 beanName 的映射

调用 BeanDefinitionRegistry 的 `#registerAlias(String name, String alias)` 方法，注册 `alias` 和 `beanName` 的映射关系。代码如下：

```
// SimpleAliasRegistry.java

/** Map from alias to canonical name. */
// key: alias
// value: beanName
private final Map<String, String> aliasMap = new ConcurrentHashMap<>(16);

@Override
public void registerAlias(String name, String alias) {
    // 校验 name 、 alias
    Assert.hasText(name, "'name' must not be empty");
    Assert.hasText(alias, "'alias' must not be empty");
    synchronized (this.aliasMap) {
        // name == alias 则去掉alias
        if (alias.equals(name)) {
            this.aliasMap.remove(alias);
            if (logger.isDebugEnabled()) {
                logger.debug("Alias definition '" + alias + "' ignored since it points to same name");
            }
        } else {
            // 获取 alias 已注册的 beanName
            String registeredName = this.aliasMap.get(alias);
            // 已存在
            if (registeredName != null) {
                // 相同，则 return ，无需重复注册
                if (registeredName.equals(name)) {
                    // An existing alias - no need to re-register
                    return;
                }
                // 不允许覆盖，则抛出 IllegalStateException 异常
                if (!allowAliasOverriding()) {
                    throw new IllegalStateException("Cannot define alias '" + alias + "' for name '" +
                            name + "': It is already registered for name '" + registeredName + "'.");
                }
                if (logger.isDebugEnabled()) {
                    logger.debug("Overriding alias '" + alias + "' definition for registered name '" +
                            registeredName + "' with new target name '" + name + "'");
                }
            }
            // 校验，是否存在循环指向
            checkForAliasCircle(name, alias);
            // 注册 alias
            this.aliasMap.put(alias, name);
            if (logger.isTraceEnabled()) {
                logger.trace("Alias definition '" + alias + "' registered for name '" + name + "'");
            }
        }
    }
}
```

- 注册 `alias` 和注册 BeanDefinition 的过程差不多。

- 在最后，调用了 `#checkForAliasCircle()` 来对别名进行了**循环**检测。代码如下：

  ```
  protected void checkForAliasCircle(String name, String alias) {
      if (hasAlias(alias, name)) {
          throw new IllegalStateException("Cannot register alias '" + alias +
                  "' for name '" + name + "': Circular reference - '" +
                  name + "' is a direct or indirect alias for '" + alias + "' already");
      }
  }
  public boolean hasAlias(String name, String alias) {
      for (Map.Entry<String, String> entry : this.aliasMap.entrySet()) {
          String registeredName = entry.getValue();
          if (registeredName.equals(name)) {
              String registeredAlias = entry.getKey();
              if (registeredAlias.equals(alias) || hasAlias(registeredAlias, alias)) {
                  return true;
              }
          }
      }
      return false;
  }
  ```

  - 如果 `name`、`alias` 分别为 1 和 3 ，则构成 `（1,3）` 的映射。加入，此时集合中存在`（A,1）`、`（3,A）` 的映射，意味着出现循环指向的情况，则抛出 IllegalStateException 异常。

# 3. 小结

前面 从源码层次，分析了 IoC BeanDefinition 装载的整个过程，这篇就这些内容做一个总结将其连贯起来。

在前文提过，IoC 容器的初始化过程分为三步骤：Resource 定位、BeanDefinition 的载入和解析，BeanDefinition 注册。

[![整体步骤](http://static.iocoder.cn/1e395ffbda4fe58a49897854a7d72ce9)](http://static.iocoder.cn/1e395ffbda4fe58a49897854a7d72ce9)整体步骤

- **Resource 定位**。我们一般用外部资源来描述 Bean 对象，所以在初始化 IoC 容器的第一步就是需要定位这个外部资源。在上一篇博客（[《【死磕 Spring】—— IoC 之 Spring 统一资源加载策略》](http://svip.iocoder.cn/Spring/IoC-load-Resource)）已经详细说明了资源加载的过程。

- BeanDefinition 的装载和解析

  。装载就是 BeanDefinition 的载入。BeanDefinitionReader 读取、解析 Resource 资源，也就是将用户定义的 Bean 表示成 IoC 容器的内部数据结构：BeanDefinition 。

  - 在 IoC 容器内部维护着一个 BeanDefinition Map 的数据结构
  - 在配置文件中每一个 `<bean>` 都对应着一个 BeanDefinition 对象。

- BeanDefinition 注册

  。向 IoC 容器注册在第二步解析好的 BeanDefinition，这个过程是通过 BeanDefinitionRegistry 接口来实现的。在 IoC 容器内部其实是将第二个过程解析得到的 BeanDefinition 注入到一个 HashMap 容器中，IoC 容器就是通过这个 HashMap 来维护这些 BeanDefinition 的。

  - 在这里需要注意的一点是这个过程并没有完成依赖注入（Bean 创建），Bean 创建是发生在应用第一次调用 `#getBean(...)` 方法，向容器索要 Bean 时。
  - 当然我们可以通过设置预处理，即对某个 Bean 设置 `lazyinit = false` 属性，那么这个 Bean 的依赖注入就会在容器初始化的时候完成。

还记得在博客 [《【死磕 Spring】—— IoC 之加载 BeanDefinition》](http://svip.iocoder.cn/Spring/IoC-load-BeanDefinitions) 中提供的一段代码吗？这里我们同样也以这段代码作为我们研究 IoC 初始化过程的开端，如下：

```
ClassPathResource resource = new ClassPathResource("bean.xml");
DefaultListableBeanFactory factory = new DefaultListableBeanFactory();
XmlBeanDefinitionReader reader = new XmlBeanDefinitionReader(factory);
reader.loadBeanDefinitions(resource);
```

刚刚开始的时候可能对上面这几行代码不知道什么意思，现在应该就一目了然了：

- `ClassPathResource resource = new ClassPathResource("bean.xml");` ： 根据 Xml 配置文件创建 Resource 资源对象。ClassPathResource 是 Resource 接口的子类，`bean.xml` 文件中的内容是我们定义的 Bean 信息。
- `DefaultListableBeanFactory factory = new DefaultListableBeanFactory();` ：创建一个 BeanFactory 。DefaultListableBeanFactory 是 BeanFactory 的一个子类，BeanFactory 作为一个接口，其实它本身是不具有独立使用的功能的，而 DefaultListableBeanFactory 则是真正可以独立使用的 IoC 容器，它是整个 Spring IoC 的始祖，在后续会有专门的文章来分析它。
- `XmlBeanDefinitionReader reader = new XmlBeanDefinitionReader(factory);` ：创建 XmlBeanDefinitionReader 读取器，用于载入 BeanDefinition 。
- `reader.loadBeanDefinitions(resource);`：开始 BeanDefinition 的载入和注册进程，完成后的 BeanDefinition 放置在 IoC 容器中。

# 4. Resource 定位

Spring 为了解决资源定位的问题，提供了两个接口：Resource、ResourceLoader，其中：

- Resource 接口是 Spring 统一资源的抽象接口
- ResourceLoader 则是 Spring 资源加载的统一抽象。
- 关于Resource、ResourceLoader 的更多知识请关注 [《【死磕 Spring】—— IoC 之 Spring 统一资源加载策略》](http://svip.iocoder.cn/Spring/IoC-load-Resource)

Resource 资源的定位需要 Resource 和 ResourceLoader 两个接口互相配合，在上面那段代码中 `new ClassPathResource("bean.xml")` 为我们定义了资源，那么 ResourceLoader 则是在什么时候初始化的呢？看 XmlBeanDefinitionReader 构造方法：

```
// XmlBeanDefinitionReader.java
public XmlBeanDefinitionReader(BeanDefinitionRegistry registry) {
	super(registry);
}
```

- 直接调用父类 AbstractBeanDefinitionReader 构造方法，代码如下：

  ```
  // AbstractBeanDefinitionReader.java
  
  protected AbstractBeanDefinitionReader(BeanDefinitionRegistry registry) {
  	Assert.notNull(registry, "BeanDefinitionRegistry must not be null");
  	this.registry = registry;
  	// Determine ResourceLoader to use.
  	if (this.registry instanceof ResourceLoader) {
  		this.resourceLoader = (ResourceLoader) this.registry;
  	}	else {
  		this.resourceLoader = new PathMatchingResourcePatternResolver();
  	}
  
  	// Inherit Environment if possible
  	if (this.registry instanceof EnvironmentCapable) {
  		this.environment = ((EnvironmentCapable) this.registry).getEnvironment();
  	} else {
  		this.environment = new StandardEnvironment();
  	}
  }
  ```

  - 核心在于设置 resourceLoader 这段，如果设置了 ResourceLoader 则用设置的，否则使用 PathMatchingResourcePatternResolver ，该类是一个集大成者的 ResourceLoader。

# 5. BeanDefinition 的载入和解析

`reader.loadBeanDefinitions(resource);` 代码段，开启 BeanDefinition 的解析过程。如下：

```
// XmlBeanDefinitionReader.java
@Override
public int loadBeanDefinitions(Resource resource) throws BeanDefinitionStoreException {
	return loadBeanDefinitions(new EncodedResource(resource));
}
```

- 在这个方法会将资源 resource 包装成一个 EncodedResource 实例对象，然后调用 `#loadBeanDefinitions(EncodedResource encodedResource)` 方法。而将 Resource 封装成 EncodedResource 主要是为了对 Resource 进行编码，保证内容读取的正确性。代码如下：

  ```
  // XmlBeanDefinitionReader.java
  
  public int loadBeanDefinitions(EncodedResource encodedResource) throws BeanDefinitionStoreException {
  	// ... 省略一些代码
  	try {
  		// 将资源文件转为 InputStream 的 IO 流
  		InputStream inputStream = encodedResource.getResource().getInputStream();
  		try {
  			// 从 InputStream 中得到 XML 的解析源
  			InputSource inputSource = new InputSource(inputStream);
  			if (encodedResource.getEncoding() != null) {
  				inputSource.setEncoding(encodedResource.getEncoding());
  			}
  			// ... 具体的读取过程
  			return doLoadBeanDefinitions(inputSource, encodedResource.getResource());
  		}
  		finally {
  			inputStream.close();
  		}
  	}
  	// 省略一些代码
  }
  ```

  - 从 `encodedResource` 源中获取 xml 的解析源，然后调用 `#doLoadBeanDefinitions(InputSource inputSource, Resource resource)` 方法，执行具体的解析过程。

    ```
    // XmlBeanDefinitionReader.java
    
    protected int doLoadBeanDefinitions(InputSource inputSource, Resource resource)
    			throws BeanDefinitionStoreException {
    	try {
    		// 获取 XML Document 实例
    		Document doc = doLoadDocument(inputSource, resource);
    		// 根据 Document 实例，注册 Bean 信息
    		int count = registerBeanDefinitions(doc, resource);
    		return count;
    	}
    	// ... 省略一堆配置
    }
    ```

    - 在该方法中主要做两件事：
    - 1、根据 xml 解析源获取相应的 Document 对象。详细解析，见 [「2.1 转换为 Document 对象」](http://svip.iocoder.cn/Spring/IoC-load-BeanDefinitions-summary/#) 。
    - 2、调用 `#registerBeanDefinitions(Document doc, Resource resource)` 方法，开启 BeanDefinition 的解析注册过程。详细解析，见 [「2.2 注册 BeanDefinition」](http://svip.iocoder.cn/Spring/IoC-load-BeanDefinitions-summary/#) 。

## 2.1 转换为 Document 对象

调用 `#doLoadDocument(InputSource inputSource, Resource resource)` 方法，会将 Bean 定义的资源转换为 Document 对象。代码如下：

```
// XmlBeanDefinitionReader.java

protected Document doLoadDocument(InputSource inputSource, Resource resource) throws Exception {
	return this.documentLoader.loadDocument(inputSource, getEntityResolver(), this.errorHandler,
			getValidationModeForResource(resource), isNamespaceAware());
}
```

该方法接受五个参数：

- `inputSource` ：加载 Document 的 Resource 源。

- ```
  entityResolver
  ```

   

  ：解析文件的解析器。

  - 【重要】详细解析，见 [《【死磕 Spring】—— IoC 之获取 Document 对象》](http://svip.iocoder.cn/Spring/IoC-load-Document) 。

- `errorHandler` ：处理加载 Document 对象的过程的错误。

- ```
  validationMode
  ```

   

  ：验证模式。

  - 【重要】详细解析，见 [《【死磕 Spring】—— IoC 之获取验证模型》](http://svip.iocoder.cn/Spring/IoC-Validation-Mode-For-Resource) 。

- `namespaceAware` ：命名空间支持。如果要提供对 XML 名称空间的支持，则为 `true` 。

------

`#loadDocument(InputSource inputSource, EntityResolver entityResolver, ErrorHandler errorHandler, int validationMode, boolean namespaceAware)` 方法，在类 DefaultDocumentLoader 中提供了实现。代码如下：

```
// DefaultDocumentLoader.java

@Override
public Document loadDocument(InputSource inputSource, EntityResolver entityResolver,
		ErrorHandler errorHandler, int validationMode, boolean namespaceAware) throws Exception {
	// 创建 DocumentBuilderFactory
	DocumentBuilderFactory factory = createDocumentBuilderFactory(validationMode, namespaceAware);
	// 创建 DocumentBuilder
	DocumentBuilder builder = createDocumentBuilder(factory, entityResolver, errorHandler);
	// 解析 XML InputSource 返回 Document 对象
	return builder.parse(inputSource);
}
```

## 2.2 注册 BeanDefinition 流程

这到这里，就已经将定义的 Bean 资源文件，载入并转换为 Document 对象了。那么，下一步就是如何将其解析为 SpringIoC 管理的 BeanDefinition 对象，并将其注册到容器中。这个过程由方法 `#registerBeanDefinitions(Document doc, Resource resource)` 方法来实现。代码如下：

```
// XmlBeanDefinitionReader.java

public int registerBeanDefinitions(Document doc, Resource resource) throws BeanDefinitionStoreException {
	// 创建 BeanDefinitionDocumentReader 对象
	BeanDefinitionDocumentReader documentReader = createBeanDefinitionDocumentReader();
	// 获取已注册的 BeanDefinition 数量
	int countBefore = getRegistry().getBeanDefinitionCount();
	// 创建 XmlReaderContext 对象
	// 注册 BeanDefinition
	documentReader.registerBeanDefinitions(doc, createReaderContext(resource));
	// 计算新注册的 BeanDefinition 数量
	return getRegistry().getBeanDefinitionCount() - countBefore;
}
```

- 首先，创建 BeanDefinition 的解析器 BeanDefinitionDocumentReader 。

- 然后，调用该 BeanDefinitionDocumentReader 的 `#registerBeanDefinitions(Document doc, XmlReaderContext readerContext)` 方法，开启解析过程，这里使用的是委派模式，具体的实现由子类 DefaultBeanDefinitionDocumentReader 完成。代码如下：

  ```
  // DefaultBeanDefinitionDocumentReader.java
  
  @Override
  public void registerBeanDefinitions(Document doc, XmlReaderContext readerContext) {
      this.readerContext = readerContext;
      // 获得 XML Document Root Element
      // 执行注册 BeanDefinition
      doRegisterBeanDefinitions(doc.getDocumentElement());
  }
  ```

### 2.2.1 对 Document 对象的解析

从 Document 对象中获取根元素 root，然后调用 ``#doRegisterBeanDefinitions(Element root)` 方法，开启真正的解析过程。代码如下：

```
// DefaultBeanDefinitionDocumentReader.java

protected void doRegisterBeanDefinitions(Element root) {
    // ... 省略部分代码（非核心）
    this.delegate = createDelegate(getReaderContext(), root, parent);

    // 解析前处理
    preProcessXml(root);
    // 解析
    parseBeanDefinitions(root, this.delegate);
    // 解析后处理
    postProcessXml(root);

}
```

- `#preProcessXml(Element root)`、`#postProcessXml(Element root)` 为前置、后置增强处理，目前 Spring 中都是空实现。

- `#parseBeanDefinitions(Element root, BeanDefinitionParserDelegate delegate)` 是对根元素 root 的解析注册过程。代码如下：

  ```
  // DefaultBeanDefinitionDocumentReader.java
  
  protected void parseBeanDefinitions(Element root, BeanDefinitionParserDelegate delegate) {
      // 如果根节点使用默认命名空间，执行默认解析
      if (delegate.isDefaultNamespace(root)) {
          // 遍历子节点
          NodeList nl = root.getChildNodes();
          for (int i = 0; i < nl.getLength(); i++) {
              Node node = nl.item(i);
              if (node instanceof Element) {
                  Element ele = (Element) node;
                  // 如果该节点使用默认命名空间，执行默认解析
                  if (delegate.isDefaultNamespace(ele)) {
                      parseDefaultElement(ele, delegate);
                  // 如果该节点非默认命名空间，执行自定义解析
                  } else {
                      delegate.parseCustomElement(ele);
                  }
              }
          }
      // 如果根节点非默认命名空间，执行自定义解析
      } else {
          delegate.parseCustomElement(root);
      }
  }
  ```

  - 迭代 root 元素的所有子节点，对其进行判断：
    - 若节点为默认命名空间，则调用 `#parseDefaultElement(Element ele, BeanDefinitionParserDelegate delegate)` 方法，开启默认标签的解析注册过程。详细解析，见 [「2.2.1.1 默认标签解析」](http://svip.iocoder.cn/Spring/IoC-load-BeanDefinitions-summary/#) 。
    - 否则，调用 `BeanDefinitionParserDelegate#parseCustomElement(Element ele)` 方法，开启自定义标签的解析注册过程。详细解析，见 [「2.2.1.2 自定义标签解析」](http://svip.iocoder.cn/Spring/IoC-load-BeanDefinitions-summary/#) 。

#### 2.2.1.1 默认标签解析

若定义的元素节点使用的是 Spring 默认命名空间，则调用 `#parseDefaultElement(Element ele, BeanDefinitionParserDelegate delegate)` 方法，进行默认标签解析。代码如下：

```
// DefaultBeanDefinitionDocumentReader.java

private void parseDefaultElement(Element ele, BeanDefinitionParserDelegate delegate) {
	if (delegate.nodeNameEquals(ele, IMPORT_ELEMENT)) { // import
		importBeanDefinitionResource(ele);
	} else if (delegate.nodeNameEquals(ele, ALIAS_ELEMENT)) { // alias
		processAliasRegistration(ele);
	} else if (delegate.nodeNameEquals(ele, BEAN_ELEMENT)) { // bean
		processBeanDefinition(ele, delegate);
	} else if (delegate.nodeNameEquals(ele, NESTED_BEANS_ELEMENT)) { // beans
		// recurse
		doRegisterBeanDefinitions(ele);
	}
}
```

对四大标签：`<import>`、`<alias>`、`<bean>`、`<beans>` 进行解析。**其中 `<bean>` 标签的解析为核心工作**。关于各个标签的解析过程，见如下文章：

- [《【死磕 Spring】—— IoC 之解析 标签》](http://svip.iocoder.cn/Spring/IoC-parse-BeanDefinitions-for-import)
- [《【死磕 Spring】—— IoC 之解析 标签：开启解析进程》](http://svip.iocoder.cn/Spring/IoC-parse-BeanDefinitions-in-processBeanDefinition)
- [《【死磕 Spring】—— IoC 之解析 标签：BeanDefinition》](http://svip.iocoder.cn/Spring/IoC-parse-BeanDefinitions-for-BeanDefinition)
- [《【死磕 Spring】—— IoC 之解析 标签：meta、lookup-method、replace-method》](http://svip.iocoder.cn/Spring/IoC-parse-BeanDefinitions-for-meta-and-look-method-and-replace-method)
- [《【死磕 Spring】—— IoC 之解析 标签：constructor-arg、property、qualifier》](http://svip.iocoder.cn/Spring/IoC-parse-BeanDefinitions-for-constructor-arg-and-property-and-qualifier)
- [《【死磕 Spring】—— IoC 之解析 标签：解析自定义标签》](http://svip.iocoder.cn/Spring/IoC-parse-BeanDefinitions-for-decorateBeanDefinitionIfRequired)

#### 2.2.1.2 自定义标签解析

对于默认标签则由 `parseCustomElement(Element ele)` 方法，负责解析。代码如下：

```
// BeanDefinitionParserDelegate.java

@Nullable
public BeanDefinition parseCustomElement(Element ele) {
    return parseCustomElement(ele, null);
}

@Nullable
public BeanDefinition parseCustomElement(Element ele, @Nullable BeanDefinition containingBd) {
    // 获取 namespaceUri
    String namespaceUri = getNamespaceURI(ele);
    if (namespaceUri == null) {
        return null;
    }
    // 根据 namespaceUri 获取相应的 Handler
    NamespaceHandler handler = this.readerContext.getNamespaceHandlerResolver().resolve(namespaceUri);
    if (handler == null) {
        error("Unable to locate Spring NamespaceHandler for XML schema namespace [" + namespaceUri + "]", ele);
        return null;
    }
    // 调用自定义的 Handler 处理
    return handler.parse(ele, new ParserContext(this.readerContext, this, containingBd));
}
```

获取节点的 `namespaceUri`，然后根据该 `namespaceUri` 获取相对应的 NamespaceHandler，最后调用 NamespaceHandler 的 `#parse(Element element, ParserContext parserContext)` 方法，即完成自定义标签的解析和注入。

想了解更多，可参考：[《【死磕 Spring】—— IoC 之解析自定义标签》](http://svip.iocoder.cn/Spring/IoC-parse-BeanDefinitions-in-parseCustomElement) 。

### 2.2.2 注册 BeanDefinition

经过上面的解析，则将 Document 对象里面的 Bean 标签解析成了一个个的 BeanDefinition ，下一步则是将这些 BeanDefinition 注册到 IoC 容器中。动作的触发是在解析 Bean 标签完成后，代码如下：

```
// DefaultBeanDefinitionDocumentReader.java

protected void processBeanDefinition(Element ele, BeanDefinitionParserDelegate delegate) {
    // 进行 bean 元素解析。
    // 如果解析成功，则返回 BeanDefinitionHolder 对象。而 BeanDefinitionHolder 为 name 和 alias 的 BeanDefinition 对象
    // 如果解析失败，则返回 null 。
    BeanDefinitionHolder bdHolder = delegate.parseBeanDefinitionElement(ele);
    if (bdHolder != null) {
        // 进行自定义标签处理
        bdHolder = delegate.decorateBeanDefinitionIfRequired(ele, bdHolder);
        try {
            // 进行 BeanDefinition 的注册
            // Register the final decorated instance.
            BeanDefinitionReaderUtils.registerBeanDefinition(bdHolder, getReaderContext().getRegistry());
        } catch (BeanDefinitionStoreException ex) {
            getReaderContext().error("Failed to register bean definition with name '" +
                    bdHolder.getBeanName() + "'", ele, ex);
        }
        // 发出响应事件，通知相关的监听器，已完成该 Bean 标签的解析。
        // Send registration event.
        getReaderContext().fireComponentRegistered(new BeanComponentDefinition(bdHolder));
    }
}
```

- 调用 `BeanDefinitionReaderUtils.registerBeanDefinition()` 方法，来注册。其实，这里面也是调用 BeanDefinitionRegistry 的 `#registerBeanDefinition(String beanName, BeanDefinition beanDefinition)` 方法，来注册 BeanDefinition 。不过，最终的实现是在 DefaultListableBeanFactory 中实现，代码如下：

  ```
  // DefaultListableBeanFactory.java
  @Override
  public void registerBeanDefinition(String beanName, BeanDefinition beanDefinition)
          throws BeanDefinitionStoreException {
      // ...省略校验相关的代码
      // 从缓存中获取指定 beanName 的 BeanDefinition
      BeanDefinition existingDefinition = this.beanDefinitionMap.get(beanName);
      // 如果已经存在
      if (existingDefinition != null) {
          // 如果存在但是不允许覆盖，抛出异常
          if (!isAllowBeanDefinitionOverriding()) {
               throw new BeanDefinitionOverrideException(beanName, beanDefinition, existingDefinition);
          } else {
             // ...省略 logger 打印日志相关的代码
          }
          // 【重点】允许覆盖，直接覆盖原有的 BeanDefinition 到 beanDefinitionMap 中。
          this.beanDefinitionMap.put(beanName, beanDefinition);
      // 如果未存在
      } else {
          // ... 省略非核心的代码
          // 【重点】添加到 BeanDefinition 到 beanDefinitionMap 中。
          this.beanDefinitionMap.put(beanName, beanDefinition);
      }
      // 重新设置 beanName 对应的缓存
      if (existingDefinition != null || containsSingleton(beanName)) {
          resetBeanDefinition(beanName);
      }
  }
  ```

  - 这段代码最核心的部分是这句 `this.beanDefinitionMap.put(beanName, beanDefinition)` 代码段。所以，注册过程也不是那么的高大上，就是利用一个 Map 的集合对象来存放：`key` 是 `beanName` ，`value` 是 BeanDefinition 对象。

想了解更多，可参考：[《【死磕 Spring】—— IoC 之注册解析的 BeanDefinitions》](http://svip.iocoder.cn/Spring/IoC-register-BeanDefinitions-really) 。

# 6. 小结

至此，整个 IoC 的初始化过程就已经完成了，从 Bean 资源的定位，转换为 Document 对象，接着对其进行解析，最后注册到 IoC 容器中，都已经完美地完成了。现在 IoC 容器中已经建立了整个 Bean 的配置信息，这些 Bean 可以被检索、使用、维护，他们是控制反转的基础，是后面注入 Bean 的依赖。最后用一张流程图来结束这篇总结之文。

[![整体流程](http://static.iocoder.cn/38419d23d29c83a4758f73f85281e076)](http://static.iocoder.cn/38419d23d29c83a4758f73f85281e076)整体流程

另外，艿艿推荐几篇不错的 Srping IoC 容器相关的博客：

- JavaDoop [《Spring IOC 容器源码分析》](https://javadoop.com/post/spring-ioc)
- Yikun [《Spring IOC 核心源码学习》](https://yikun.github.io/2015/05/29/Spring-IOC核心源码学习/)
- DearBelinda [《Spring专题之 IOC 源码分析》](https://segmentfault.com/a/1190000016261917)