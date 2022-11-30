**本文主要基于 Spring 5.0.6.RELEASE**

摘要: 原创出处 http://cmsblogs.com/?p=2697 「小明哥」，谢谢！

作为「小明哥」的忠实读者，略作修改，记录在理解过程中，参考的资料。

------

获取 XML Document 对象后，会根据该对象和 Resource 资源对象调用 `XmlBeanDefinitionReader#registerBeanDefinitions(Document doc, Resource resource)` 方法，开始注册 BeanDefinitions 之旅。代码如下：

```java
// AbstractBeanDefinitionReader.java
private final BeanDefinitionRegistry registry;

// XmlBeanDefinitionReader.java
public int registerBeanDefinitions(Document doc, Resource resource) throws BeanDefinitionStoreException {
	// <1> 创建 BeanDefinitionDocumentReader 对象
	BeanDefinitionDocumentReader documentReader = createBeanDefinitionDocumentReader();
	// <2> 获取已注册的 BeanDefinition 数量
	int countBefore = getRegistry().getBeanDefinitionCount();
	// <3> 创建 XmlReaderContext 对象
	// <4> 注册 BeanDefinition
	documentReader.registerBeanDefinitions(doc, createReaderContext(resource));
	// 计算新注册的 BeanDefinition 数量
	return getRegistry().getBeanDefinitionCount() - countBefore;
}
```

- `<1>` 处，调用 `#createBeanDefinitionDocumentReader()` 方法，实例化 BeanDefinitionDocumentReader 对象。

> FROM 《Spring 源码深度解析》P16 页
>
> 定义读取 Document 并注册 BeanDefinition 功能

- `<2>` 处，调用 `BeanDefinitionRegistry#getBeanDefinitionCount()` 方法，获取**已注册**的 BeanDefinition 数量。
- `<3>` 处，调用 `#createReaderContext(Resource resource)` 方法，创建 XmlReaderContext 对象。
- `<4>` 处，调用 `BeanDefinitionDocumentReader#registerBeanDefinitions(Document doc, XmlReaderContext readerContext)` 方法，读取 XML 元素，注册 BeanDefinition 们。
- `<5>` 处，计**算新注册**的 BeanDefinition 数量。

# 1. createBeanDefinitionDocumentReader

`#createBeanDefinitionDocumentReader()`，实例化 BeanDefinitionDocumentReader 对象。代码如下：

```java
/**
 * documentReader 的类
 *
 * @see #createBeanDefinitionDocumentReader() 
 */
private Class<? extends BeanDefinitionDocumentReader> documentReaderClass = DefaultBeanDefinitionDocumentReader.class;

protected BeanDefinitionDocumentReader createBeanDefinitionDocumentReader() {
	return BeanUtils.instantiateClass(this.documentReaderClass);
}
```

- `documentReaderClass` 的默认值为 `DefaultBeanDefinitionDocumentReader.class` 。关于它，我们在后续的文章，详细解析。

# 2. registerBeanDefinitions

`BeanDefinitionDocumentReader#registerBeanDefinitions(Document doc, XmlReaderContext readerContext)` 方法，注册 BeanDefinition ，在接口 BeanDefinitionDocumentReader 中定义。代码如下：

```java
public interface BeanDefinitionDocumentReader {

	/**
	 * Read bean definitions from the given DOM document and
	 * register them with the registry in the given reader context.
	 * @param doc the DOM document
	 * @param readerContext the current context of the reader
	 * (includes the target registry and the resource being parsed)
	 * @throws BeanDefinitionStoreException in case of parsing errors
	 */
	void registerBeanDefinitions(Document doc, XmlReaderContext readerContext)
			throws BeanDefinitionStoreException;

}
```

**从给定的 Document 对象中解析定义的 BeanDefinition 并将他们注册到注册表中**。方法接收两个参数：

- `doc` 方法参数：待解析的 Document 对象。
- `readerContext` 方法，解析器的当前上下文，包括目标注册表和被解析的资源。它是根据 Resource 来创建的，见 [「3. createReaderContext」](http://svip.iocoder.cn/Spring/IoC-register-BeanDefinitions/#) 。

## 2.1 DefaultBeanDefinitionDocumentReader

BeanDefinitionDocumentReader 有且只有一个默认实现类 DefaultBeanDefinitionDocumentReader 。它对 `#registerBeanDefinitions(...)` 方法的实现代码如下：

DefaultBeanDefinitionDocumentReader 对该方法提供了实现：

```java
@Nullable
private XmlReaderContext readerContext;

@Nullable
private BeanDefinitionParserDelegate delegate;
    
/**
 * This implementation parses bean definitions according to the "spring-beans" XSD
 * (or DTD, historically).
 * <p>Opens a DOM Document; then initializes the default settings
 * specified at the {@code <beans/>} level; then parses the contained bean definitions.
 */
@Override
public void registerBeanDefinitions(Document doc, XmlReaderContext readerContext) {
    this.readerContext = readerContext;
    // 获得 XML Document Root Element
    // 执行注册 BeanDefinition
    doRegisterBeanDefinitions(doc.getDocumentElement());
}

/**
 * Register each bean definition within the given root {@code <beans/>} element.
 */
@SuppressWarnings("deprecation")  // for Environment.acceptsProfiles(String...)
protected void doRegisterBeanDefinitions(Element root) {
    // Any nested <beans> elements will cause recursion in this method. In
    // order to propagate and preserve <beans> default-* attributes correctly,
    // keep track of the current (parent) delegate, which may be null. Create
    // the new (child) delegate with a reference to the parent for fallback purposes,
    // then ultimately reset this.delegate back to its original (parent) reference.
    // this behavior emulates a stack of delegates without actually necessitating one.
    // 记录老的 BeanDefinitionParserDelegate 对象
    BeanDefinitionParserDelegate parent = this.delegate;
    // <1> 创建 BeanDefinitionParserDelegate 对象，并进行设置到 delegate
    this.delegate = createDelegate(getReaderContext(), root, parent);
    // <2> 检查 <beans /> 根标签的命名空间是否为空，或者是 http://www.springframework.org/schema/beans
    if (this.delegate.isDefaultNamespace(root)) {
        // <2.1> 处理 profile 属性。可参见《Spring3自定义环境配置 <beans profile="">》http://nassir.iteye.com/blog/1535799
        String profileSpec = root.getAttribute(PROFILE_ATTRIBUTE);
        if (StringUtils.hasText(profileSpec)) {
            // <2.2> 使用分隔符切分，可能有多个 profile 。
            String[] specifiedProfiles = StringUtils.tokenizeToStringArray(
                    profileSpec, BeanDefinitionParserDelegate.MULTI_VALUE_ATTRIBUTE_DELIMITERS);
            // <2.3> 如果所有 profile 都无效，则不进行注册
            // We cannot use Profiles.of(...) since profile expressions are not supported
            // in XML config. See SPR-12458 for details.
            if (!getReaderContext().getEnvironment().acceptsProfiles(specifiedProfiles)) {
                if (logger.isDebugEnabled()) {
                    logger.debug("Skipped XML bean definition file due to specified profiles [" + profileSpec +
                            "] not matching: " + getReaderContext().getResource());
                }
                return;
            }
        }
    }

    // <3> 解析前处理
    preProcessXml(root);
    // <4> 解析
    parseBeanDefinitions(root, this.delegate);
    // <5> 解析后处理
    postProcessXml(root);

    // 设置 delegate 回老的 BeanDefinitionParserDelegate 对象
    this.delegate = parent;
}
```

- `<1>` 处，创建 BeanDefinitionParserDelegate 对象，并进行设置到 `delegate` 。BeanDefinitionParserDelegate 是一个重要的类，它负责**解析 BeanDefinition**。代码如下：

  > FROM 《Spring 源码深度解析》P16
  >
  > 定义解析 XML Element 的各种方法

  ```java
  protected BeanDefinitionParserDelegate createDelegate(
          XmlReaderContext readerContext, Element root, @Nullable BeanDefinitionParserDelegate parentDelegate) {
      // 创建 BeanDefinitionParserDelegate 对象
      BeanDefinitionParserDelegate delegate = new BeanDefinitionParserDelegate(readerContext);
      // 初始化默认
      delegate.initDefaults(root, parentDelegate);
      return delegate;
  }
  ```

- `<2>` 处，检查 `<beans />` **根**标签的命名空间是否为空，或者是 http://www.springframework.org/schema/beans 。

  - `<2.1>` 处，判断是否 `<beans />` 上配置了 `profile` 属性。不了解这块的胖友，可以看下 [《《Spring3自定义环境配置 》》](http://nassir.iteye.com/blog/1535799) 。
  - `<2.2>` 处，使用分隔符切分，可能有**多个** profile 。
  - `<2.3>` 处，判断，如果所有 profile 都无效，则 `return` 不进行注册。

- `<4>` 处，调用 `#parseBeanDefinitions(Element root, BeanDefinitionParserDelegate delegate)` 方法，进行解析逻辑。详细解析，见 [「3.1 parseBeanDefinitions」](http://svip.iocoder.cn/Spring/IoC-register-BeanDefinitions/#) 。

- `<3>` / `<5>` 处，解析**前后**的处理，目前这两个方法都是空实现，交由子类来实现。代码如下：

  ```java
  protected void preProcessXml(Element root) {}
  
  protected void postProcessXml(Element root) {}
  ```

### 2.1.1 parseBeanDefinitions

`#parseBeanDefinitions(Element root, BeanDefinitionParserDelegate delegate)` 方法，进行解析逻辑。代码如下：

```java
/**
 * Parse the elements at the root level in the document:
 * "import", "alias", "bean".
 * @param root the DOM root element of the document
 */
protected void parseBeanDefinitions(Element root, BeanDefinitionParserDelegate delegate) {
    // <1> 如果根节点使用默认命名空间，执行默认解析
    if (delegate.isDefaultNamespace(root)) {
        // 遍历子节点
        NodeList nl = root.getChildNodes();
        for (int i = 0; i < nl.getLength(); i++) {
            Node node = nl.item(i);
            if (node instanceof Element) {
                Element ele = (Element) node;
                // <1> 如果该节点使用默认命名空间，执行默认解析
                if (delegate.isDefaultNamespace(ele)) {
                    parseDefaultElement(ele, delegate);
                // 如果该节点非默认命名空间，执行自定义解析
                } else {
                    delegate.parseCustomElement(ele);
                }
            }
        }
    // <2> 如果根节点非默认命名空间，执行自定义解析
    } else {
        delegate.parseCustomElement(root);
    }
}
```

- Spring 有两种Bean 声明方式：

  - 配置文件式声明：`<bean id="studentService" class="org.springframework.core.StudentService" />` 。对应 `<1>` 处。
  - 自定义注解方式：`<tx:annotation-driven>` 。对应 `<2>` 处。
  
- `<1>` 处，如果**根**节点或**子**节点**使用**默认命名空间，调用 `#parseDefaultElement(Element ele, BeanDefinitionParserDelegate delegate)` 方法，执行默认解析。代码如下：

  ```java
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

  - 详细的解析，见后续文章。

- `<2>` 处，如果**根**节点或**子**节点**不使用**默认命名空间，调用 `BeanDefinitionParserDelegate#parseCustomElement(Element ele)` 方法，执行**自定义**解析。详细的解析，见后续文章。

# 3. createReaderContext

`#createReaderContext(Resource resource)` 方法，创建 XmlReaderContext 对象。代码如下：

```java
private ProblemReporter problemReporter = new FailFastProblemReporter();

private ReaderEventListener eventListener = new EmptyReaderEventListener();

private SourceExtractor sourceExtractor = new NullSourceExtractor();

@Nullable
private NamespaceHandlerResolver namespaceHandlerResolver;

/**
 * Create the {@link XmlReaderContext} to pass over to the document reader.
 */
public XmlReaderContext createReaderContext(Resource resource) {
	return new XmlReaderContext(resource, this.problemReporter, this.eventListener,
			this.sourceExtractor, this, getNamespaceHandlerResolver());
}
```

关于 XmlReaderContext 的详细解析，见后续文章。



# 4、注册表：BeanDefinitionRegistry

将定义 Bean 的资源文件解析成 BeanDefinition 后需要将其注入容器中，这个过程由 BeanDefinitionRegistry 来完成。

**BeanDefinitionRegistry：向注册表中注册 BeanDefinition 实例，完成注册的过程。**

下图是 BeanDefinitionRegistry 类结构图：

[![BeanDefinitionRegistry 类图](http://static.iocoder.cn/c91d5c1d310f4257bb0edae3444e7cd9)](http://static.iocoder.cn/c91d5c1d310f4257bb0edae3444e7cd9)BeanDefinitionRegistry 类图

BeanDefinitionRegistry 继承了 AliasRegistry 接口，其核心子类有三个：SimpleBeanDefinitionRegistry、DefaultListableBeanFactory、GenericApplicationContext 。

## 1. AliasRegistry

**用于别名管理的通用型接口，作为 BeanDefinitionRegistry 的顶层接口。** AliasRegistry 定义了一些别名管理的方法。

```java
// AliasRegistry.java

public interface AliasRegistry {

    void registerAlias(String name, String alias);
    void removeAlias(String alias);

    boolean isAlias(String name);
    String[] getAliases(String name);

}
```

## 2. BeanDefinitionRegistry

**BeanDefinition 的注册接口，如 RootBeanDefinition 和 ChildBeanDefinition。它通常由 BeanFactories 实现，在 Spring 中已知的实现者为：DefaultListableBeanFactory 和 GenericApplicationContext。BeanDefinitionRegistry 是 Spring 的 Bean 工厂包中唯一封装 BeanDefinition 注册的接口。**

BeanDefinitionRegistry 接口定义了关于 BeanDefinition 注册、注销、查询等一系列的操作。

```java
// BeanDefinitionRegistry.java

    public interface BeanDefinitionRegistry extends AliasRegistry {

    // 往注册表中注册一个新的 BeanDefinition 实例
    void registerBeanDefinition(String beanName, BeanDefinition beanDefinition) throws BeanDefinitionStoreException;

    // 移除注册表中已注册的 BeanDefinition 实例
    void removeBeanDefinition(String beanName) throws NoSuchBeanDefinitionException;

    // 从注册中取得指定的 BeanDefinition 实例
    BeanDefinition getBeanDefinition(String beanName) throws NoSuchBeanDefinitionException;

    // 判断 BeanDefinition 实例是否在注册表中（是否注册）
    boolean containsBeanDefinition(String beanName);

    // 取得注册表中所有 BeanDefinition 实例的 beanName（标识）
    String[] getBeanDefinitionNames();

    // 返回注册表中 BeanDefinition 实例的数量
    int getBeanDefinitionCount();

    // beanName（标识）是否被占用
    boolean isBeanNameInUse(String beanName);

}
```

## 3. SimpleBeanDefinitionRegistry

**SimpleBeanDefinitionRegistry 是 BeanDefinitionRegistry 一个简单的实现，它还继承 SimpleAliasRegistry（ AliasRegistry 的简单实现），它仅仅只提供注册表功能，无工厂功能**。

SimpleBeanDefinitionRegistry 使用 ConcurrentHashMap 来存储注册的 BeanDefinition。

```java
// SimpleBeanDefinitionRegistry.java

private final Map<String, BeanDefinition> beanDefinitionMap = new ConcurrentHashMap<>(64);
```

他对注册其中的 BeanDefinition 都是基于 `beanDefinitionMap` 这个集合来实现的，如下：

```java
// SimpleBeanDefinitionRegistry.java

@Override
public void registerBeanDefinition(String beanName, BeanDefinition beanDefinition)
   throws BeanDefinitionStoreException {
	Assert.hasText(beanName, "'beanName' must not be empty");
	Assert.notNull(beanDefinition, "BeanDefinition must not be null");
	this.beanDefinitionMap.put(beanName, beanDefinition);
}

@Override
public void removeBeanDefinition(String beanName) throws NoSuchBeanDefinitionException {
	if (this.beanDefinitionMap.remove(beanName) == null) {
		throw new NoSuchBeanDefinitionException(beanName);
	}
}

@Override
public BeanDefinition getBeanDefinition(String beanName) throws NoSuchBeanDefinitionException {
	BeanDefinition bd = this.beanDefinitionMap.get(beanName);
	if (bd == null) {
		throw new NoSuchBeanDefinitionException(beanName);
	}
	return bd;
}
```

实现简单、粗暴。

## 4. DefaultListableBeanFactory

**DefaultListableBeanFactory，ConfigurableListableBeanFactory（其实就是 BeanFactory ） 和 BeanDefinitionRegistry 接口的默认实现：一个基于 BeanDefinition 元数据的完整 bean 工厂**。所以相对于 SimpleBeanDefinitionRegistry 而言，DefaultListableBeanFactory 则是一个具有注册功能的完整 Bean 工厂。它同样是用 ConcurrentHashMap 数据结构来存储注册的 BeanDefinition 。

```java
// DefaultListableBeanFactory.java

// 注册表，由 BeanDefinition 的标识 （beanName） 与其实例组成
private final Map<String, BeanDefinition> beanDefinitionMap = new ConcurrentHashMap<String, bean>(64);

// 标识（beanName）集合
private final List<String> beanDefinitionNames = new ArrayList<String>(64);
```

### 4.1 registerBeanDefinition

在看看 `#registerBeanDefinition(String beanName, BeanDefinition beanDefinition)` 方法，代码如下：

```java
// DefaultListableBeanFactory.java

public void registerBeanDefinition(String beanName, BeanDefinition beanDefinition)
   throws BeanDefinitionStoreException {

    // ... 省略其他代码

    // 如果未存在
    } else {
        // 检测创建 Bean 阶段是否已经开启，如果开启了则需要对 beanDefinitionMap 进行并发控制
        if (hasBeanCreationStarted()) {
            // beanDefinitionMap 为全局变量，避免并发情况
            // Cannot modify startup-time collection elements anymore (for stable iteration)
            synchronized (this.beanDefinitionMap) {
                // <x> 添加到 BeanDefinition 到 beanDefinitionMap 中。
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
            // <x> 添加到 BeanDefinition 到 beanDefinitionMap 中。
            this.beanDefinitionMap.put(beanName, beanDefinition);
            // 添加 beanName 到 beanDefinitionNames 中
            this.beanDefinitionNames.add(beanName);
            // 从 manualSingletonNames 移除 beanName
            this.manualSingletonNames.remove(beanName);
        }

        this.frozenBeanDefinitionNames = null;
    }

    // 重新设置 beanName 对应的缓存
    if (existingDefinition != null || containsSingleton(beanName)) {
        resetBeanDefinition(beanName);
    }

}
```

- 其实上面一堆代码最重要就只有一句，就是 `<x>` 处：

  ```java
  // DefaultListableBeanFactory.java
  
  this.beanDefinitionMap.put(beanName, beanDefinition);
  ```

### 4.2 removeBeanDefinition

在看看 `#removeBeanDefinition(String beanName)` 方法，其实也是调用 `beanDefinitionMap.remove(beanName)` 的逻辑。

## 5. GenericApplicationContext

对于类 GenericApplicationContext ，查看源码你会发现他实现注册、注销功能都是委托 DefaultListableBeanFactory 实现的。简化代码如下：

```java
// GenericApplicationContext.java

private final DefaultListableBeanFactory beanFactory;

@Override
public void registerBeanDefinition(String beanName, BeanDefinition beanDefinition)
		throws BeanDefinitionStoreException {

	this.beanFactory.registerBeanDefinition(beanName, beanDefinition);
}

@Override
public void removeBeanDefinition(String beanName) throws NoSuchBeanDefinitionException {
	this.beanFactory.removeBeanDefinition(beanName);
}

// ... 省略其它 N 多方法
```

# 5. 小结

至此，`XmlBeanDefinitionReader#doLoadBeanDefinitions(InputSource inputSource, Resource resource)` 方法中，做的三件事情已经全部分析完毕，下面将对 **BeanDefinition 的解析过程**做详细分析说明。

另外，`XmlBeanDefinitionReader#doLoadBeanDefinitions(InputSource inputSource, Resource resource)` 方法，整体时序图如下：

[![时序图](http://static.iocoder.cn/images/Spring/2019-02-16/01.jpg)](http://static.iocoder.cn/images/Spring/2019-02-16/01.jpg)时序图

- 红框部分，就是 **BeanDefinition 的解析过程**。