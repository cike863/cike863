# 1、开启解析进程

**本文主要基于 Spring 5.0.6.RELEASE**

摘要: 原创出处 http://cmsblogs.com/?p=2731 「小明哥」，谢谢！

作为「小明哥」的忠实读者，「老艿艿」略作修改，记录在理解过程中，参考的资料。

------

`import` 标签解析完毕了，我们一起来看看 Spring 中**最复杂也是最重要**的标签 `bean` 标签的解析过程。

## 1. processBeanDefinition

在方法 `#parseDefaultElement(...)` 方法中，如果遇到标签为 `bean` 时，则调用 `#processBeanDefinition(Element ele, BeanDefinitionParserDelegate delegate)` 方法，进行 `bean` 标签的解析。代码如下：

```
// DefaultBeanDefinitionDocumentReader.java

/**
 * Process the given bean element, parsing the bean definition
 * and registering it with the registry.
 */
protected void processBeanDefinition(Element ele, BeanDefinitionParserDelegate delegate) {
    // 进行 bean 元素解析。
    // <1> 如果解析成功，则返回 BeanDefinitionHolder 对象。而 BeanDefinitionHolder 为 name 和 alias 的 BeanDefinition 对象
    // 如果解析失败，则返回 null 。
    BeanDefinitionHolder bdHolder = delegate.parseBeanDefinitionElement(ele);
    if (bdHolder != null) {
        // <2> 进行自定义标签处理
        bdHolder = delegate.decorateBeanDefinitionIfRequired(ele, bdHolder);
        try {
            // <3> 进行 BeanDefinition 的注册
            // Register the final decorated instance.
            BeanDefinitionReaderUtils.registerBeanDefinition(bdHolder, getReaderContext().getRegistry());
        } catch (BeanDefinitionStoreException ex) {
            getReaderContext().error("Failed to register bean definition with name '" +
                    bdHolder.getBeanName() + "'", ele, ex);
        }
        // <4> 发出响应事件，通知相关的监听器，已完成该 Bean 标签的解析。
        // Send registration event.
        getReaderContext().fireComponentRegistered(new BeanComponentDefinition(bdHolder));
    }
}
```

整个过程分为四个步骤：

1. 调用

    

   ```
   BeanDefinitionParserDelegate#parseBeanDefinitionElement(Element ele, BeanDefinitionParserDelegate delegate)
   ```

    

   方法，进行元素解析。

   - 如果解析**失败**，则返回 `null`，错误由 ProblemReporter 处理。
   - 如果解析**成功**，则返回 BeanDefinitionHolder 实例 `bdHolder` 。BeanDefinitionHolder 为持有 `name` 和 `alias` 的 BeanDefinition。
   - 详细解析，见 [「2. parseBeanDefinitionElement」](http://svip.iocoder.cn/Spring/IoC-parse-BeanDefinitions-in-processBeanDefinition/#) 。

2. 若实例 `bdHolder` 不为空，则调用 `BeanDefinitionParserDelegate#decorateBeanDefinitionIfRequired(Element ele, BeanDefinitionHolder bdHolder)` 方法，进行自定义标签处理。

3. 解析完成后，则调用 `BeanDefinitionReaderUtils#registerBeanDefinition(BeanDefinitionHolder definitionHolder, BeanDefinitionRegistry registry)` 方法，对 `bdHolder` 进行 BeanDefinition 的注册。

4. 发出响应事件，通知相关的监听器，完成 Bean 标签解析。

## 2. parseBeanDefinitionElement

`BeanDefinitionParserDelegate#parseBeanDefinitionElement(Element ele, BeanDefinitionParserDelegate delegate)` 方法，进行 `<bean>` 元素解析。代码如下：

```
// BeanDefinitionParserDelegate.java

/**
 * Parses the supplied {@code <bean>} element. May return {@code null}
 * if there were errors during parse. Errors are reported to the
 * {@link org.springframework.beans.factory.parsing.ProblemReporter}.
 */
@Nullable
public BeanDefinitionHolder parseBeanDefinitionElement(Element ele) {
    return parseBeanDefinitionElement(ele, null);
}

/**
 * Parses the supplied {@code <bean>} element. May return {@code null}
 * if there were errors during parse. Errors are reported to the
 * {@link org.springframework.beans.factory.parsing.ProblemReporter}.
 *
 * @param containingBean TODO 芋艿，需要进一步确认
 */
@Nullable
public BeanDefinitionHolder parseBeanDefinitionElement(Element ele, @Nullable BeanDefinition containingBean) {
    // <1> 解析 id 和 name 属性
    String id = ele.getAttribute(ID_ATTRIBUTE);
    String nameAttr = ele.getAttribute(NAME_ATTRIBUTE);

    // <1> 计算别名集合
    List<String> aliases = new ArrayList<>();
    if (StringUtils.hasLength(nameAttr)) {
        String[] nameArr = StringUtils.tokenizeToStringArray(nameAttr, MULTI_VALUE_ATTRIBUTE_DELIMITERS);
        aliases.addAll(Arrays.asList(nameArr));
    }

    // <3.1> beanName ，优先，使用 id
    String beanName = id;
    // <3.2> beanName ，其次，使用 aliases 的第一个
    if (!StringUtils.hasText(beanName) && !aliases.isEmpty()) {
        beanName = aliases.remove(0); // 移除出别名集合
        if (logger.isTraceEnabled()) {
            logger.trace("No XML 'id' specified - using '" + beanName +
                    "' as bean name and " + aliases + " as aliases");
        }
    }

    // <2> 检查 beanName 的唯一性
    if (containingBean == null) {
        checkNameUniqueness(beanName, aliases, ele);
    }

    // <4> 解析属性，构造 AbstractBeanDefinition 对象
    AbstractBeanDefinition beanDefinition = parseBeanDefinitionElement(ele, beanName, containingBean);
    if (beanDefinition != null) {
        // <3.3> beanName ，再次，使用 beanName 生成规则
        if (!StringUtils.hasText(beanName)) {
            try {
                if (containingBean != null) {
                    // <3.3> 生成唯一的 beanName
                    beanName = BeanDefinitionReaderUtils.generateBeanName(
                            beanDefinition, this.readerContext.getRegistry(), true);
                } else {
                    // <3.3> 生成唯一的 beanName
                    beanName = this.readerContext.generateBeanName(beanDefinition);
                    // TODO 芋艿，需要进一步确认
                    // Register an alias for the plain bean class name, if still possible,
                    // if the generator returned the class name plus a suffix.
                    // This is expected for Spring 1.2/2.0 backwards compatibility.
                    String beanClassName = beanDefinition.getBeanClassName();
                    if (beanClassName != null &&
                            beanName.startsWith(beanClassName) && beanName.length() > beanClassName.length() &&
                            !this.readerContext.getRegistry().isBeanNameInUse(beanClassName)) {
                        aliases.add(beanClassName);
                    }
                }
                if (logger.isTraceEnabled()) {
                    logger.trace("Neither XML 'id' nor 'name' specified - " +
                            "using generated bean name [" + beanName + "]");
                }
            } catch (Exception ex) {
                error(ex.getMessage(), ele);
                return null;
            }
        }
        // <5> 创建 BeanDefinitionHolder 对象
        String[] aliasesArray = StringUtils.toStringArray(aliases);
        return new BeanDefinitionHolder(beanDefinition, beanName, aliasesArray);
    }
    return null;
}
```

这个方法还没有对 `bean` 标签进行解析，只是在解析动作之前做了一些功能架构，主要的工作有：

- `<1>` 处，解析 `id`、`name` 属性，确定 `aliases` 集合

- `<2>` 处，检测 `beanName` 是否唯一。代码如下：

  ```
  /**
   * 已使用 Bean 名字的集合
   *
   * Stores all used bean names so we can enforce uniqueness on a per
   * beans-element basis. Duplicate bean ids/names may not exist within the
   * same level of beans element nesting, but may be duplicated across levels.
   */
  private final Set<String> usedNames = new HashSet<>();
  
  /**
   * Validate that the specified bean name and aliases have not been used already
   * within the current level of beans element nesting.
   */
  protected void checkNameUniqueness(String beanName, List<String> aliases, Element beanElement) {
  	// 寻找是否 beanName 已经使用
      String foundName = null;
  	if (StringUtils.hasText(beanName) && this.usedNames.contains(beanName)) {
  		foundName = beanName;
  	}
  	if (foundName == null) {
  		foundName = CollectionUtils.findFirstMatch(this.usedNames, aliases);
  	}
  	// 若已使用，使用 problemReporter 提示错误
  	if (foundName != null) {
  		error("Bean name '" + foundName + "' is already used in this <beans> element", beanElement);
  	}
  
  	// 添加到 usedNames 集合
  	this.usedNames.add(beanName);
  	this.usedNames.addAll(aliases);
  }
  ```

- 这里有必要说下 `beanName` 的命名规则：

  - `<3.1>` 处，如果 `id` 不为空，则 `beanName = id` 。
  - `<3.2>` 处，如果 `id` 为空，但是 `aliases` 不空，则 `beanName` 为 `aliases` 的**第一个**元素
  - `<3.3>` 处，如果两者都为空，则根据**默认规则**来设置 beanName 。因为**默认规则**不是本文的重点，所以暂时省略。感兴趣的胖友，自己研究下哈。

- `<4>` 处，调用 `#parseBeanDefinitionElement(Element ele, String beanName, BeanDefinition containingBean)` 方法，对属性进行解析并封装成 AbstractBeanDefinition 实例 `beanDefinition` 。详细解析，见 [「2.1 parseBeanDefinitionElement」](http://svip.iocoder.cn/Spring/IoC-parse-BeanDefinitions-in-processBeanDefinition/#) 。

- `<5>` 处，根据所获取的信息（`beanName`、`aliases`、`beanDefinition`）构造 BeanDefinitionHolder 实例对象并返回。其中，BeanDefinitionHolder 的简化代码如下：

  ```
  /**
   * BeanDefinition 对象
   */
  private final BeanDefinition beanDefinition;
  /**
   * Bean 名字
   */
  private final String beanName;
  /**
   * 别名集合
   */
  @Nullable
  private final String[] aliases;
  ```

> TODO 芋艿，需要进一步确认，未来参考下 [《Spring专题之IOC源码分析》](https://segmentfault.com/a/1190000016261917) ，进行细化。

#### 2.1 parseBeanDefinitionElement

`#parseBeanDefinitionElement(Element ele, String beanName, BeanDefinition containingBean)` 方法，对属性进行解析并封装成 AbstractBeanDefinition 实例，代码如下：

```
/**
 * Parse the bean definition itself, without regard to name or aliases. May return
 * {@code null} if problems occurred during the parsing of the bean definition.
 */
@Nullable
public AbstractBeanDefinition parseBeanDefinitionElement(
        Element ele, String beanName, @Nullable BeanDefinition containingBean) {

    this.parseState.push(new BeanEntry(beanName));

    // 解析 class 属性
    String className = null;
    if (ele.hasAttribute(CLASS_ATTRIBUTE)) {
        className = ele.getAttribute(CLASS_ATTRIBUTE).trim();
    }
    // 解析 parent 属性
    String parent = null;
    if (ele.hasAttribute(PARENT_ATTRIBUTE)) {
        parent = ele.getAttribute(PARENT_ATTRIBUTE);
    }

    try {
        // 创建用于承载属性的 AbstractBeanDefinition 实例
        AbstractBeanDefinition bd = createBeanDefinition(className, parent);

        // 解析默认 bean 的各种属性
        parseBeanDefinitionAttributes(ele, beanName, containingBean, bd);
        // 提取 description
        bd.setDescription(DomUtils.getChildElementValueByTagName(ele, DESCRIPTION_ELEMENT));

        // tips：
        // 下面的一堆是解析 <bean>......</bean> 内部的子元素，
        // 解析出来以后的信息都放到 bd 的属性中

        // 解析元数据 <meta />
        parseMetaElements(ele, bd);
        // 解析 lookup-method 属性 <lookup-method />
        parseLookupOverrideSubElements(ele, bd.getMethodOverrides());
        // 解析 replaced-method 属性 <replaced-method />
        parseReplacedMethodSubElements(ele, bd.getMethodOverrides());

        // 解析构造函数参数 <constructor-arg />
        parseConstructorArgElements(ele, bd);
        // 解析 property 子元素 <property />
        parsePropertyElements(ele, bd);
        // 解析 qualifier 子元素 <qualifier />
        parseQualifierElements(ele, bd);

        bd.setResource(this.readerContext.getResource());
        bd.setSource(extractSource(ele));

        return bd;
    } catch (ClassNotFoundException ex) {
        error("Bean class [" + className + "] not found", ele, ex);
    } catch (NoClassDefFoundError err) {
        error("Class that bean class [" + className + "] depends on not found", ele, err);
    } catch (Throwable ex) {
        error("Unexpected failure during bean definition parsing", ele, ex);
    } finally {
        this.parseState.pop();
    }

    return null;
}
```

到这里，`bean` 标签的所有属性我们都可以看到其解析的过程，也就说到这里我们已经解析一个基本可用的 BeanDefinition 。

#### 2.2 createBeanDefinition

`#createBeanDefinition(String className, String parentName)` 方法，创建 AbstractBeanDefinition 对象。代码如下：

```
/**
 * Create a bean definition for the given class name and parent name.
 * @param className the name of the bean class
 * @param parentName the name of the bean's parent bean
 * @return the newly created bean definition
 * @throws ClassNotFoundException if bean class resolution was attempted but failed
 */
protected AbstractBeanDefinition createBeanDefinition(@Nullable String className, @Nullable String parentName)
		throws ClassNotFoundException {
	return BeanDefinitionReaderUtils.createBeanDefinition(
			parentName, className, this.readerContext.getBeanClassLoader());
}
```

## 3. 小节

由于解析过程 `bean` 标签的**属性**较为漫长，篇幅较大，为了更好的观看体验，将这篇博文进行**拆分**。

下篇博客主要介绍 BeanDefinition ，以及解析默认 `bean` 标签的各种属性的过程，即 `#parseBeanDefinitionAttributes(Element ele, String beanName, BeanDefinition containingBean, AbstractBeanDefinition bd)` 方法。

# 2、BeanDefinition

前面历经千辛万苦终于到达解析 `bean` 标签步骤来了，解析 bean 标签的过程其实就是构造一个 BeanDefinition 对象的过程。`<bean>` 元素标签拥有的配置属性，BeanDefinition 均提供了相应的属性，与之一一对应。所以，我们有必要对 BeanDefinition **先**有一个整体的认识。

## 1. BeanDefinition

`org.springframework.beans.factory.config.BeanDefinition` ，是一个接口，它描述了一个 Bean 实例的**定义**，包括属性值、构造方法值和继承自它的类的更多信息。代码如下：

```
String SCOPE_SINGLETON = ConfigurableBeanFactory.SCOPE_SINGLETON;
String SCOPE_PROTOTYPE = ConfigurableBeanFactory.SCOPE_PROTOTYPE;


int ROLE_APPLICATION = 0;
int ROLE_SUPPORT = 1;
int ROLE_INFRASTRUCTURE = 2;

void setParentName(@Nullable String parentName);
@Nullable
String getParentName();

void setBeanClassName(@Nullable String beanClassName);
@Nullable
String getBeanClassName();

void setScope(@Nullable String scope);
@Nullable
String getScope();

void setLazyInit(boolean lazyInit);
boolean isLazyInit();

void setDependsOn(@Nullable String... dependsOn);
@Nullable
String[] getDependsOn();

void setAutowireCandidate(boolean autowireCandidate);
boolean isAutowireCandidate();

void setPrimary(boolean primary);
boolean isPrimary();

void setFactoryBeanName(@Nullable String factoryBeanName);
@Nullable
String getFactoryBeanName();

void setFactoryMethodName(@Nullable String factoryMethodName);
@Nullable
String getFactoryMethodName();

ConstructorArgumentValues getConstructorArgumentValues();
default boolean hasConstructorArgumentValues() {
	return !getConstructorArgumentValues().isEmpty();
}

MutablePropertyValues getPropertyValues();
default boolean hasPropertyValues() {
	return !getPropertyValues().isEmpty();
}

void setInitMethodName(@Nullable String initMethodName);
@Nullable
String getInitMethodName();

void setDestroyMethodName(@Nullable String destroyMethodName);
@Nullable
String getDestroyMethodName();

void setRole(int role);
int getRole();

void setDescription(@Nullable String description);
@Nullable
String getDescription();

boolean isSingleton();

boolean isPrototype();

boolean isAbstract();

@Nullable
String getResourceDescription();

@Nullable
BeanDefinition getOriginatingBeanDefinition();
```

虽然接口方法比较多，但是是不是一下子和我们平时使用 `<bean>` 标签的属性，能够对应上落。

#### 1.1 BeanDefinition 的父关系

BeanDefinition 继承 AttributeAccessor 和 BeanMetadataElement 接口。两个接口定义如下：

- `org.springframework.cor.AttributeAccessor` 接口，定义了与其它对象的（元数据）进行连接和访问的约定，即对属性的修改，包括获取、设置、删除。代码如下：

  ```
  public interface AttributeAccessor {
  
  	void setAttribute(String name, @Nullable Object value);
  
  	@Nullable
  	Object getAttribute(String name);
  
  	@Nullable
  	Object removeAttribute(String name);
  
  	boolean hasAttribute(String name);
  
  	String[] attributeNames();
  
  }
  ```

- `org.springframework.beans.BeanMetadataElement` 接口，Bean 元对象持有的配置元素可以通过 `#getSource()` 方法来获取。代码如下：

  ```
  public interface BeanMetadataElement {
  
  	@Nullable
  	Object getSource();
  
  }
  ```

#### 1.2 BeanDefinition 的子关系

BeanDefinition 子关系，结构如下图：

![类图](http://static.iocoder.cn/6abb56b3651c0eee105c510bb9fc6473)

我们常用的三个实现类有：

- `org.springframework.beans.factory.support.ChildBeanDefinition`
- `org.springframework.beans.factory.support.RootBeanDefinition`
- `org.springframework.beans.factory.support.GenericBeanDefinition`
- ChildBeanDefinition、RootBeanDefinition、GenericBeanDefinition 三者都继承 AbstractBeanDefinition 抽象类，即 AbstractBeanDefinition 对三个子类的共同的类信息进行抽象。
- 如果配置文件中定义了父 `<bean>` 和 子 `<bean>` ，则父 `<bean>` 用 RootBeanDefinition 表示，子 `<bean>` 用 ChildBeanDefinition 表示，而没有父 `<bean>` 的就使用RootBeanDefinition 表示。
- GenericBeanDefinition 为一站式服务类。😈 这个解释一脸懵逼？没事，继续往下看。

## 2. 解析 Bean 标签

在 `BeanDefinitionParserDelegate#parseBeanDefinitionElement(Element ele, String beanName, BeanDefinition containingBean)` 方法中，完成解析后，返回的是一个已经完成对 `<bean>` 标签解析的 BeanDefinition 实例。

#### 2.1 createBeanDefinition

在该方法内部，首先调用 `#createBeanDefinition(String className, String parentName)` 方法，创建 AbstractBeanDefinition 对象。代码如下：

```
protected AbstractBeanDefinition createBeanDefinition(@Nullable String className, @Nullable String parentName)
		throws ClassNotFoundException {
	return BeanDefinitionReaderUtils.createBeanDefinition(
			parentName, className, this.readerContext.getBeanClassLoader());
}
```

- 委托 BeanDefinitionReaderUtils 创建，代码如下：

  ```
  // BeanDefinitionReaderUtils.java
  
  public static AbstractBeanDefinition createBeanDefinition(
          @Nullable String parentName, @Nullable String className, @Nullable ClassLoader classLoader) throws ClassNotFoundException {
      // 创建 GenericBeanDefinition 对象
      GenericBeanDefinition bd = new GenericBeanDefinition();
      // 设置 parentName
      bd.setParentName(parentName);
      if (className != null) {
          // 设置 beanClass
          if (classLoader != null) {
              bd.setBeanClass(ClassUtils.forName(className, classLoader));
          // 设置 beanClassName
          } else {
              bd.setBeanClassName(className);
          }
      }
      return bd;
  }
  ```

  - 该方法主要是，创建 GenericBeanDefinition 对象，并设置 `parentName`、`className`、`beanClass` 属性。

#### 2.2 parseBeanDefinitionAttributes

创建完 GenericBeanDefinition 实例后，再调用 `#parseBeanDefinitionAttributes(Element ele, String beanName, BeanDefinition containingBean, AbstractBeanDefinition bd)` 方法，该方法将创建好的 GenericBeanDefinition 实例当做参数，对 `bean` 标签的所有属性进行解析，如下：

```
// BeanDefinitionParserDelegate.java

public AbstractBeanDefinition parseBeanDefinitionAttributes(Element ele, String beanName,
        @Nullable BeanDefinition containingBean, AbstractBeanDefinition bd) {
    // 解析 scope 属性
    if (ele.hasAttribute(SINGLETON_ATTRIBUTE)) {
        error("Old 1.x 'singleton' attribute in use - upgrade to 'scope' declaration", ele);
    } else if (ele.hasAttribute(SCOPE_ATTRIBUTE)) {
        bd.setScope(ele.getAttribute(SCOPE_ATTRIBUTE));
    } else if (containingBean != null) {
        // Take default from containing bean in case of an inner bean definition.
        bd.setScope(containingBean.getScope());
    }

    // 解析 abstract 属性
    if (ele.hasAttribute(ABSTRACT_ATTRIBUTE)) {
        bd.setAbstract(TRUE_VALUE.equals(ele.getAttribute(ABSTRACT_ATTRIBUTE)));
    }

    // 解析 lazy-init 属性
    String lazyInit = ele.getAttribute(LAZY_INIT_ATTRIBUTE);
    if (DEFAULT_VALUE.equals(lazyInit)) {
        lazyInit = this.defaults.getLazyInit();
    }
    bd.setLazyInit(TRUE_VALUE.equals(lazyInit));

    // 解析 autowire 属性
    String autowire = ele.getAttribute(AUTOWIRE_ATTRIBUTE);
    bd.setAutowireMode(getAutowireMode(autowire));

    // 解析 depends-on 属性
    if (ele.hasAttribute(DEPENDS_ON_ATTRIBUTE)) {
        String dependsOn = ele.getAttribute(DEPENDS_ON_ATTRIBUTE);
        bd.setDependsOn(StringUtils.tokenizeToStringArray(dependsOn, MULTI_VALUE_ATTRIBUTE_DELIMITERS));
    }

    // 解析 autowire-candidate 属性
    String autowireCandidate = ele.getAttribute(AUTOWIRE_CANDIDATE_ATTRIBUTE);
    if ("".equals(autowireCandidate) || DEFAULT_VALUE.equals(autowireCandidate)) {
        String candidatePattern = this.defaults.getAutowireCandidates();
        if (candidatePattern != null) {
            String[] patterns = StringUtils.commaDelimitedListToStringArray(candidatePattern);
            bd.setAutowireCandidate(PatternMatchUtils.simpleMatch(patterns, beanName));
        }
    } else {
        bd.setAutowireCandidate(TRUE_VALUE.equals(autowireCandidate));
    }

    // 解析 primary 标签
    if (ele.hasAttribute(PRIMARY_ATTRIBUTE)) {
        bd.setPrimary(TRUE_VALUE.equals(ele.getAttribute(PRIMARY_ATTRIBUTE)));
    }

    // 解析 init-method 属性
    if (ele.hasAttribute(INIT_METHOD_ATTRIBUTE)) {
        String initMethodName = ele.getAttribute(INIT_METHOD_ATTRIBUTE);
        bd.setInitMethodName(initMethodName);
    } else if (this.defaults.getInitMethod() != null) {
        bd.setInitMethodName(this.defaults.getInitMethod());
        bd.setEnforceInitMethod(false);
    }

    // 解析 destroy-method 属性
    if (ele.hasAttribute(DESTROY_METHOD_ATTRIBUTE)) {
        String destroyMethodName = ele.getAttribute(DESTROY_METHOD_ATTRIBUTE);
        bd.setDestroyMethodName(destroyMethodName);
    } else if (this.defaults.getDestroyMethod() != null) {
        bd.setDestroyMethodName(this.defaults.getDestroyMethod());
        bd.setEnforceDestroyMethod(false);
    }

    // 解析 factory-method 属性
    if (ele.hasAttribute(FACTORY_METHOD_ATTRIBUTE)) {
        bd.setFactoryMethodName(ele.getAttribute(FACTORY_METHOD_ATTRIBUTE));
    }
    if (ele.hasAttribute(FACTORY_BEAN_ATTRIBUTE)) {
        bd.setFactoryBeanName(ele.getAttribute(FACTORY_BEAN_ATTRIBUTE));
    }

    return bd;
}
```

从上面代码我们可以清晰地看到对 `bean` 标签属性的解析，这些属性我们在工作中都或多或少用到过。



完成 `bean` 标签的基本属性解析后，会依次调用 BeanDefinitionParserDelegate 的 `#parseMetaElements(lement ele, BeanMetadataAttributeAccessor attributeAccessor)`、`#parseLookupOverrideSubElements(Element beanEle, MethodOverrides overrides)`、`#parseReplacedMethodSubElements(Element beanEle, MethodOverrides overrides)` 方法，分别对子元素 `meta`、`lookup-method`、`replace-method` 元素完成解析。

# 3、meta、lookup-method、replace-method

完成 `bean` 标签的基本属性解析后，会依次调用 BeanDefinitionParserDelegate 的 `#parseMetaElements(lement ele, BeanMetadataAttributeAccessor attributeAccessor)`、`#parseLookupOverrideSubElements(Element beanEle, MethodOverrides overrides)`、`#parseReplacedMethodSubElements(Element beanEle, MethodOverrides overrides)` 方法，分别对子元素 `meta`、`lookup-method`、`replace-method` 元素完成解析。三个子元素的作用如下：

- `<meta>` ：元数据。
- `<lookup-method>` ：Spring 动态改变 bean 里方法的实现。方法执行返回的对象，使用 Spring 内原有的这类对象替换，通过改变方法返回值来动态改变方法。内部实现为使用 cglib 方法，重新生成子类，重写配置的方法和返回对象，达到动态改变的效果。

- `<replace-method>` ：Spring 动态改变 bean 里方法的实现。需要改变的方法，使用 Spring 内原有其他类（需要继承接口`org.springframework.beans.factory.support.MethodReplacer`）的逻辑，替换这个方法。通过改变方法执行逻辑来动态改变方法。

## 1. meta 子元素

> **meta** ：元数据。当需要使用里面的信息时可以通过 key 获取。

meta 所声明的 key 并不会在 Bean 中体现，只是一个额外的声明，当我们需要使用里面的信息时，通过调用 BeanDefinition 的 `#getAttribute(String name)` 方法来获取。该子元素的解析过程，代码如下：

```
// BeanDefinitionParserDelegate.java

public void parseMetaElements(Element ele, BeanMetadataAttributeAccessor attributeAccessor) {
    NodeList nl = ele.getChildNodes();
    // 遍历子节点
    for (int i = 0; i < nl.getLength(); i++) {
        Node node = nl.item(i);
        // <meta key="special-data" value="sprecial stragey" />
        if (isCandidateElement(node) && nodeNameEquals(node, META_ELEMENT)) { // 标签名为 meta
            Element metaElement = (Element) node;
            String key = metaElement.getAttribute(KEY_ATTRIBUTE); // key
            String value = metaElement.getAttribute(VALUE_ATTRIBUTE); // value
            // 创建 BeanMetadataAttribute 对象
            BeanMetadataAttribute attribute = new BeanMetadataAttribute(key, value);
            attribute.setSource(extractSource(metaElement));
            // 添加到 BeanMetadataAttributeAccessor 中
            attributeAccessor.addMetadataAttribute(attribute);
        }
    }
}
```

- 解析过程较为简单，获取相应的 key - value 构建 BeanMetadataAttribute 对象，然后调用 `BeanMetadataAttributeAccessor#addMetadataAttribute(BeanMetadataAttribute)` 方法，添加 BeanMetadataAttribute 加入到 AbstractBeanDefinition 中。

  > 友情提示：
  >
  > AbstractBeanDefinition 继承 BeanMetadataAttributeAccessor 类
  > BeanMetadataAttributeAccessor 继承 AttributeAccessorSupport 类。

#### 1.1 addMetadataAttribute

调用 `BeanMetadataAttributeAccessor#addMetadataAttribute(BeanMetadataAttribute)` 方法，添加 BeanMetadataAttribute 加入到 AbstractBeanDefinition 中。代码如下：

```
// BeanMetadataAttributeAccessor.java

public void addMetadataAttribute(BeanMetadataAttribute attribute) {
	super.setAttribute(attribute.getName(), attribute);
}
```

- 委托 AttributeAccessorSupport 实现，如下：

```
// AttributeAccessorSupport.java

/** Map with String keys and Object values. */
private final Map<String, Object> attributes = new LinkedHashMap<>();

@Override
public void setAttribute(String name, @Nullable Object value) {
	Assert.notNull(name, "Name must not be null");
	if (value != null) {
		this.attributes.put(name, value);
	} else {
		removeAttribute(name);
	}
}
```

`org.springframework.core.AttributeAccessorSupport` ，是接口 AttributeAccessor 的实现者。 AttributeAccessor 接口定义了与其他对象的元数据进行连接和访问的约定，可以通过该接口对属性进行获取、设置、删除操作。

#### 1.2 getAttribute

设置元数据后，则可以通过调用 BeanDefinition 的 `#getAttribute(String name)` 方法来获取属性。代码如下：

```
// AttributeAccessorSupport.java

/** Map with String keys and Object values. */
private final Map<String, Object> attributes = new LinkedHashMap<>();

@Override
@Nullable
public Object getAttribute(String name) {
	Assert.notNull(name, "Name must not be null");
	return this.attributes.get(name);
}
```

## 2. lookup-method 子元素

> **lookup-method** ：获取器注入，是把一个方法声明为返回某种类型的 bean 但实际要返回的 bean 是在配置文件里面配置的。该方法可以用于设计一些可插拔的功能上，解除程序依赖。

#### 2.1 示例

直接上例子：

```
public interface Car {

    void display();

}

public class Bmw implements Car{

    @Override
    public void display() {
        System.out.println("我是 BMW");
    }

}

public class Hongqi implements Car{

    @Override
    public void display() {
        System.out.println("我是 hongqi");
    }

}

public abstract class Display {

    public void display(){
        getCar().display();
    }
    
    public abstract Car getCar();

}

public static void main(String[] args) {
    ApplicationContext context = new ClassPathXmlApplicationContext("classpath:spring.xml");
    Display display = (Display) context.getBean("display");
    display.display();
}
```

XML 配置内容如下：

```
<bean id="display" class="org.springframework.core.test1.Display">
    <lookup-method name="getCar" bean="hongqi"/>
</bean>
```

运行结果为：

```
我是 hongqi
```

如果将 `bean="hognqi"` 替换为 `bean="bmw"`，则运行结果变成：

```
我是 BMW
```

#### 2.2 parseLookupOverrideSubElements

看了这个示例，我们初步了解了 `looku-method` 子元素提供的功能了。其解析通过 `#parseLookupOverrideSubElements(Element beanEle, MethodOverrides overrides)` 方法，代码如下：

```
// BeanDefinitionParserDelegate.java

public void parseLookupOverrideSubElements(Element beanEle, MethodOverrides overrides) {
    NodeList nl = beanEle.getChildNodes();
    // 遍历子节点
    for (int i = 0; i < nl.getLength(); i++) {
        Node node = nl.item(i);
        if (isCandidateElement(node) && nodeNameEquals(node, LOOKUP_METHOD_ELEMENT)) { // 标签名为 lookup-method
            Element ele = (Element) node;
            String methodName = ele.getAttribute(NAME_ATTRIBUTE); // name
            String beanRef = ele.getAttribute(BEAN_ELEMENT); // bean
            // 创建 LookupOverride 对象
            LookupOverride override = new LookupOverride(methodName, beanRef);
            override.setSource(extractSource(ele));
            // 添加到 MethodOverrides 中
            overrides.addOverride(override);
        }
    }
}
```

解析过程和 `meta` 子元素没有多大区别，同样是解析 methodName、beanRef 构造一个 LookupOverride 对象，然后记录到 AbstractBeanDefinition 中的 `methodOverrides` 属性中。

在实例化 Bean 的时候，再详细阐述具体的实现过程，这里仅仅只是一个**标记**作用。

## 3. replace-method 子元素

> **replaced-method** ：可以在运行时调用新的方法替换现有的方法，还能动态的更新原有方法的逻辑。

#### 3.1 示例

该标签使用方法和 `lookup-method` 标签差不多，只不过替代方法的类需要实现 `org.springframework.beans.factory.support.MethodReplacer` 接口。如下:

```
public class Method {

    public void display(){
        System.out.println("我是原始方法");
    }

}

public class MethodReplace implements MethodReplacer {

    @Override
    public Object reimplement(Object obj, Method method, Object[] args) throws Throwable {
        System.out.println("我是替换方法");
        return null;
    }

}

public static void main(String[] args) {
    ApplicationContext context = new ClassPathXmlApplicationContext("classpath:spring.xml");
    Method method = (Method) context.getBean("method");
    method.display();
}
```

如果 `spring.xml` 文件如下：

```
<bean id="methodReplace" class="org.springframework.core.test1.MethodReplace"/>

<bean id="method" class="org.springframework.core.test1.Method"/>
```

则运行结果为：

```
我是原始方法
```

增加 `replaced-method` 子元素：

```
<bean id="methodReplace" class="org.springframework.core.test1.MethodReplace"/>

<bean id="method" class="org.springframework.core.test1.Method">

    <replaced-method name="display" replacer="methodReplace"/>

</bean>
```

运行结果为：

```
我是替换方法
```

#### 3.2 parseReplacedMethodSubElements

上面代码演示了 `replaced-method` 子元素的用法，其解析通过 `#parseReplacedMethodSubElements(Element beanEle, MethodOverrides overrides)` 方法，代码如下：

```
/**
 * Parse replaced-method sub-elements of the given bean element.
 */
public void parseReplacedMethodSubElements(Element beanEle, MethodOverrides overrides) {
    NodeList nl = beanEle.getChildNodes();
    // 遍历子节点
    for (int i = 0; i < nl.getLength(); i++) {
        Node node = nl.item(i);
        if (isCandidateElement(node) && nodeNameEquals(node, REPLACED_METHOD_ELEMENT)) { // 标签名为 replace-method
            Element replacedMethodEle = (Element) node;
            String name = replacedMethodEle.getAttribute(NAME_ATTRIBUTE); // name
            String callback = replacedMethodEle.getAttribute(REPLACER_ATTRIBUTE); // replacer
            // 创建 ReplaceOverride 对象
            ReplaceOverride replaceOverride = new ReplaceOverride(name, callback);
            // Look for arg-type match elements. 参见 《spring bean中lookup-method属性 replaced-method属性》 http://linql2010-126-com.iteye.com/blog/2018385
            List<Element> argTypeEles = DomUtils.getChildElementsByTagName(replacedMethodEle, ARG_TYPE_ELEMENT); // arg-type 子标签
            for (Element argTypeEle : argTypeEles) {
                String match = argTypeEle.getAttribute(ARG_TYPE_MATCH_ATTRIBUTE); // arg-type 子标签的 match 属性
                match = (StringUtils.hasText(match) ? match : DomUtils.getTextValue(argTypeEle));
                if (StringUtils.hasText(match)) {
                    replaceOverride.addTypeIdentifier(match);
                }
            }
            replaceOverride.setSource(extractSource(replacedMethodEle));
            // 添加到 MethodOverrides 中
            overrides.addOverride(replaceOverride);
        }
    }
}
```

该子元素和 `lookup-method` 标签的解析过程差不多，同样是提取 `name` 和 `replacer` 属性构建 ReplaceOverride 对象，然后记录到 AbstractBeanDefinition 中的 `methodOverrides` 属性中。

在实例化 Bean 的时候，再详细阐述具体的实现过程，这里仅仅只是一个**标记**作用。

## 4. 小结

对于 `lookup-method` 和 `replaced-method` 两个子元素是如何使用以完成他们所提供的功能，在后续实例化 Bean 的时候会做详细说明。

# 4、constructor-arg、property、qualifier

我们来分析 `constructor-arg`、`property`、`<qualifier>` 三个子元素。

## 1. constructor-arg 子元素

#### 1.1 示例

举个小栗子，代码如下：

```
public class StudentService {

    private String name;

    private Integer age;

    private BookService bookService;

    StudentService(String name, Integer age, BookService bookService){

        this.name = name;

        this.age = age;

        this.bookService = bookService;

    }

}
<bean id="bookService" class="org.springframework.core.service.BookService"/>

<bean id="studentService" class="org.springframework.core.service.StudentService">
    <constructor-arg index="0" value="chenssy"/>
    <constructor-arg name="age" value="100"/>
    <constructor-arg name="bookService" ref="bookService"/>
</bean>
```

StudentService 定义一个构造函数，配置文件中使用 `constructor-arg` 元素对其配置，该元素可以实现对 StudentService 自动寻找对应的构造函数，并在初始化的时候将值当做参数进行设置。

#### 1.2 parseConstructorArgElements

`#parseConstructorArgElements(Element beanEle, BeanDefinition bd)` 方法，完成 `constructor-arg` 子元素的解析。代码如下：

```
// BeanDefinitionParserDelegate.java

public void parseConstructorArgElements(Element beanEle, BeanDefinition bd) {
	NodeList nl = beanEle.getChildNodes();
	for (int i = 0; i < nl.getLength(); i++) {
		Node node = nl.item(i);
		if (isCandidateElement(node) && nodeNameEquals(node, CONSTRUCTOR_ARG_ELEMENT)) { // 标签名为 constructor-arg
			parseConstructorArgElement((Element) node, bd);
		}
	}
}
```

遍历所有子元素，如果为 `constructor-arg` 标签，则调用 `#parseConstructorArgElement(Element ele, BeanDefinition bd)` 方法，进行解析。

#### 1.3 parseConstructorArgElement

```
// BeanDefinitionParserDelegate.java

public void parseConstructorArgElement(Element ele, BeanDefinition bd) {
    // 提取 index、type、name 属性值
    String indexAttr = ele.getAttribute(INDEX_ATTRIBUTE);
    String typeAttr = ele.getAttribute(TYPE_ATTRIBUTE);
    String nameAttr = ele.getAttribute(NAME_ATTRIBUTE);
    if (StringUtils.hasLength(indexAttr)) {
        try {
            // 如果有 index
            int index = Integer.parseInt(indexAttr);
            if (index < 0) {
                error("'index' cannot be lower than 0", ele);
            } else {
                try {
                    // <1>
                    this.parseState.push(new ConstructorArgumentEntry(index));
                    // <2> 解析 ele 对应属性元素
                    Object value = parsePropertyValue(ele, bd, null);
                    // <3> 根据解析的属性元素构造一个 ValueHolder 对象
                    ConstructorArgumentValues.ValueHolder valueHolder = new ConstructorArgumentValues.ValueHolder(value);
                    if (StringUtils.hasLength(typeAttr)) {
                        valueHolder.setType(typeAttr);
                    }
                    if (StringUtils.hasLength(nameAttr)) {
                        valueHolder.setName(nameAttr);
                    }
                    valueHolder.setSource(extractSource(ele));
                    // 不允许重复指定相同参数
                    if (bd.getConstructorArgumentValues().hasIndexedArgumentValue(index)) {
                        error("Ambiguous constructor-arg entries for index " + index, ele);
                    } else {
                        // <4> 加入到 indexedArgumentValues 中
                        bd.getConstructorArgumentValues().addIndexedArgumentValue(index, valueHolder);
                    }
                } finally {
                    this.parseState.pop();
                }
            }
        } catch (NumberFormatException ex) {
            error("Attribute 'index' of tag 'constructor-arg' must be an integer", ele);
        }
    } else {
        try {
            this.parseState.push(new ConstructorArgumentEntry());
            // 解析 ele 对应属性元素
            Object value = parsePropertyValue(ele, bd, null);
            // 根据解析的属性元素构造一个 ValueHolder 对象
            ConstructorArgumentValues.ValueHolder valueHolder = new ConstructorArgumentValues.ValueHolder(value);
            if (StringUtils.hasLength(typeAttr)) {
                valueHolder.setType(typeAttr);
            }
            if (StringUtils.hasLength(nameAttr)) {
                valueHolder.setName(nameAttr);
            }
            valueHolder.setSource(extractSource(ele));
            // 加入到 indexedArgumentValues 中
            bd.getConstructorArgumentValues().addGenericArgumentValue(valueHolder);
        } finally {
            this.parseState.pop();
        }
    }
}
```

首先获取 `index`、`type`、`name` 三个属性值，然后根据是否存在 `index` 来区分，执行后续逻辑。其实两者逻辑都**差不多**，总共分为如下几个步骤（以有 `index` 为例）：

1. 在 `<1>` 处，构造 ConstructorArgumentEntry 对象并将其加入到 ParseState 队列中。ConstructorArgumentEntry 表示构造函数的参数。
2. 在 `<2>` 处，调用 `#parsePropertyValue(Element ele, BeanDefinition bd, String propertyName)` 方法，解析 `constructor-arg` 子元素，返回结果值。详细解析，见 [「1.4 parsePropertyValue」](http://svip.iocoder.cn/Spring/IoC-parse-BeanDefinitions-for-constructor-arg-and-property-and-qualifier/#) 。
3. 在 `<3>` 处，根据解析的结果值，构造ConstructorArgumentValues.ValueHolder 实例对象，并将 `type`、`name` 设置到 ValueHolder 中
4. 在 `<4>` 处，最后，将 ValueHolder 实例对象添加到 `indexedArgumentValues` 集合中。

无 `index` 的处理逻辑差不多，只有几点不同：

- 构造 ConstructorArgumentEntry 对象时是调用**无参**构造函数
- 最后是将 ValueHolder 实例添加到 `genericArgumentValues` 集合中。

#### 1.4 parsePropertyValue

调用 `#parsePropertyValue(Element ele, BeanDefinition bd, String propertyName)` 方法，解析 `constructor-arg` 子元素，返回结果值。代码如下：

```
/**
 * Get the value of a property element. May be a list etc.
 * Also used for constructor arguments, "propertyName" being null in this case.
 */
@Nullable
public Object parsePropertyValue(Element ele, BeanDefinition bd, @Nullable String propertyName) {
    String elementName = (propertyName != null ?
            "<property> element for property '" + propertyName + "'" :
            "<constructor-arg> element");

    // <1> 查找子节点中，是否有 ref、value、list 等元素
    // Should only have one child element: ref, value, list, etc.
    NodeList nl = ele.getChildNodes();
    Element subElement = null;
    for (int i = 0; i < nl.getLength(); i++) {
        Node node = nl.item(i);
        // meta 、description 不处理
        if (node instanceof Element && !nodeNameEquals(node, DESCRIPTION_ELEMENT) &&
                !nodeNameEquals(node, META_ELEMENT)) {
            // Child element is what we're looking for.
            if (subElement != null) {
                error(elementName + " must not contain more than one sub-element", ele);
            } else {
                subElement = (Element) node;
            }
        }
    }

    // <1> 是否有 ref 属性
    boolean hasRefAttribute = ele.hasAttribute(REF_ATTRIBUTE);
    // <1> 是否有 value 属性
    boolean hasValueAttribute = ele.hasAttribute(VALUE_ATTRIBUTE);
    // <1> 多个元素存在，报错，存在冲突。
    if ((hasRefAttribute && hasValueAttribute) || // 1. ref 和 value 都存在
            ((hasRefAttribute || hasValueAttribute) && subElement != null)) { // 2. ref he value 存在一，并且 subElement 存在
        error(elementName +
                " is only allowed to contain either 'ref' attribute OR 'value' attribute OR sub-element", ele);
    }

    // <2> 将 ref 属性值，构造为 RuntimeBeanReference 实例对象
    if (hasRefAttribute) {
        String refName = ele.getAttribute(REF_ATTRIBUTE);
        if (!StringUtils.hasText(refName)) {
            error(elementName + " contains empty 'ref' attribute", ele);
        }
        RuntimeBeanReference ref = new RuntimeBeanReference(refName);
        ref.setSource(extractSource(ele));
        return ref;
    // <3> 将 value 属性值，构造为 TypedStringValue 实例对象
    } else if (hasValueAttribute) {
        TypedStringValue valueHolder = new TypedStringValue(ele.getAttribute(VALUE_ATTRIBUTE));
        valueHolder.setSource(extractSource(ele));
        return valueHolder;
    // <4> 解析子元素
    } else if (subElement != null) {
        return parsePropertySubElement(subElement, bd);
    } else {
        // Neither child element nor "ref" or "value" attribute found.
        error(elementName + " must specify a ref or value", ele);
        return null;
    }
}
```

1. 在

    

   ```
   <1>
   ```

    

   处，提取

    

   ```
   constructor-arg
   ```

    

   的子元素、

   ```
   ref
   ```

    

   属性值和

    

   ```
   value
   ```

    

   属性值，对其进行判断。以下两种情况是不允许存在的：

   1. `ref` 和 `value` 属性同时存在 。
   2. 存在 `ref` 或者 `value` 且又有子元素。

2. 在 `<2>` 处，若存在 `ref` 属性，则获取其值并将其封装进 `org.springframework.beans.factory.config.RuntimeBeanReference` 实例对象中。

3. 在 `<3>` 处，若存在 `value` 属性，则获取其值并将其封装进 `org.springframework.beans.factory.config.TypedStringValue` 实例对象中。

4. 在 `<4>` 处，如果子元素不为空，则调用 `#parsePropertySubElement(Element ele, BeanDefinition bd)` 方法，对子元素进一步解析。详细解析，见 [「1.5 parsePropertySubElement」](http://svip.iocoder.cn/Spring/IoC-parse-BeanDefinitions-for-constructor-arg-and-property-and-qualifier/#) 中。

#### 1.5 parsePropertySubElement

对于 `constructor-arg` 子元素的嵌套子元素，需要调用 `#parsePropertySubElement(Element ele, BeanDefinition bd)` 方法，进一步处理。

```
/**
 * Parse a value, ref or collection sub-element of a property or
 * constructor-arg element.
 * @param ele subelement of property element; we don't know which yet
 * @param defaultValueType the default type (class name) for any
 * {@code <value>} tag that might be created
 */
@Nullable
public Object parsePropertySubElement(Element ele, @Nullable BeanDefinition bd, @Nullable String defaultValueType) {
	if (!isDefaultNamespace(ele)) {
		return parseNestedCustomElement(ele, bd);
	} else if (nodeNameEquals(ele, BEAN_ELEMENT)) { // bean 标签
		BeanDefinitionHolder nestedBd = parseBeanDefinitionElement(ele, bd);
		if (nestedBd != null) {
			nestedBd = decorateBeanDefinitionIfRequired(ele, nestedBd, bd);
		}
		return nestedBd;
	} else if (nodeNameEquals(ele, REF_ELEMENT)) { // ref 标签
		// A generic reference to any name of any bean.
		String refName = ele.getAttribute(BEAN_REF_ATTRIBUTE);
		boolean toParent = false;
		if (!StringUtils.hasLength(refName)) {
			// A reference to the id of another bean in a parent context.
			refName = ele.getAttribute(PARENT_REF_ATTRIBUTE);
			toParent = true;
			if (!StringUtils.hasLength(refName)) {
				error("'bean' or 'parent' is required for <ref> element", ele);
				return null;
			}
		}
		if (!StringUtils.hasText(refName)) {
			error("<ref> element contains empty target attribute", ele);
			return null;
		}
		RuntimeBeanReference ref = new RuntimeBeanReference(refName, toParent);
		ref.setSource(extractSource(ele));
		return ref;
	} else if (nodeNameEquals(ele, IDREF_ELEMENT)) { // idref 标签
		return parseIdRefElement(ele);
	} else if (nodeNameEquals(ele, VALUE_ELEMENT)) { // value 标签
		return parseValueElement(ele, defaultValueType);
	} else if (nodeNameEquals(ele, NULL_ELEMENT)) { // null 标签
		// It's a distinguished null value. Let's wrap it in a TypedStringValue
		// object in order to preserve the source location.
		TypedStringValue nullHolder = new TypedStringValue(null);
		nullHolder.setSource(extractSource(ele));
		return nullHolder;
	} else if (nodeNameEquals(ele, ARRAY_ELEMENT)) { // array 标签
		return parseArrayElement(ele, bd);
	} else if (nodeNameEquals(ele, LIST_ELEMENT)) { // list 标签
		return parseListElement(ele, bd);
	} else if (nodeNameEquals(ele, SET_ELEMENT)) { // set 标签
		return parseSetElement(ele, bd);
	} else if (nodeNameEquals(ele, MAP_ELEMENT)) { // map 标签
		return parseMapElement(ele, bd);
	} else if (nodeNameEquals(ele, PROPS_ELEMENT)) { // props 标签
		return parsePropsElement(ele);
	} else { // 未知标签
		error("Unknown property sub-element: [" + ele.getNodeName() + "]", ele);
		return null;
	}
}
```

上面对各个子类进行分类处理，详细情况，如果各位有兴趣，可以移步源码进行深一步的探究。本文，暂时不做深入分析。

## 2. property 子元素

#### 2.1 示例

我们一般使用如下方式，来使用 `property` 子元素。

```
<bean id="studentService" class="org.springframework.core.service.StudentService">
    <property name="name" value="chenssy"/>
    <property name="age" value="18"/>
</bean>
```

#### 2.2 parsePropertyElements

对于 property 子元素的解析，Spring 调用 `parsePropertyElements(Element beanEle, BeanDefinition bd)` 方法。代码如下：

```
/**
 * Parse property sub-elements of the given bean element.
 */
public void parsePropertyElements(Element beanEle, BeanDefinition bd) {
	NodeList nl = beanEle.getChildNodes();
	for (int i = 0; i < nl.getLength(); i++) {
		Node node = nl.item(i);
		if (isCandidateElement(node) && nodeNameEquals(node, PROPERTY_ELEMENT)) { // property 标签
			parsePropertyElement((Element) node, bd);
		}
	}
}
```

和 `constructor-arg` 子元素差不多，同样是“提取”( 遍历 )所有的 `property` 的子元素，然后调用 `#parsePropertyElement((Element ele, BeanDefinition b)` 进行解析。

#### 2.3 parsePropertyElement

```
/**
 * Parse a property element.
 */
public void parsePropertyElement(Element ele, BeanDefinition bd) {
    // 获取 name 属性
    String propertyName = ele.getAttribute(NAME_ATTRIBUTE);
    if (!StringUtils.hasLength(propertyName)) {
        error("Tag 'property' must have a 'name' attribute", ele);
        return;
    }
    this.parseState.push(new PropertyEntry(propertyName));
    try {
        // 如果存在相同的 name ，报错
        if (bd.getPropertyValues().contains(propertyName)) {
            error("Multiple 'property' definitions for property '" + propertyName + "'", ele);
            return;
        }
        // 解析属性值
        Object val = parsePropertyValue(ele, bd, propertyName);
        // 创建 PropertyValue 对象
        PropertyValue pv = new PropertyValue(propertyName, val);
        parseMetaElements(ele, pv);
        pv.setSource(extractSource(ele));
        // 添加到 PropertyValue 集合中
        bd.getPropertyValues().addPropertyValue(pv);
    } finally {
        this.parseState.pop();
    }
}
```

与解析 `constructor-arg` 子元素步骤差不多：

- 调用 `#parsePropertyElement((Element ele, BeanDefinition b)` 方法， 解析子元素属性值。
- 然后，根据该值构造 PropertyValue 实例对象。
- 最后，将 PropertyValue 添加到 BeanDefinition 中的 MutablePropertyValues 中。

## 3. qualifier 子元素

> 老艿艿：小明哥貌似忘记写 `<qualifier>` 标签的解析，不过实际情况下也用的比较少，所以老艿艿，在这里稍微简单写下。
>
> 不感兴趣的胖友，可以直接跳过这块内容。

#### 3.1 示例

见 [《Spring 注解实现Bean依赖注入之 @Qualifier》](http://www.voidcn.com/article/p-vdgrbkrm-bqu.html) 文章。

#### 3.2 parseQualifierElement

`#parseQualifierElement(Element ele, AbstractBeanDefinition bd)` 方法，完成 `qualifier` 子元素的解析。代码如下：

```
/**
 * Parse a qualifier element.
 */
public void parseQualifierElement(Element ele, AbstractBeanDefinition bd) {
    // 解析 type 属性
    String typeName = ele.getAttribute(TYPE_ATTRIBUTE);
    if (!StringUtils.hasLength(typeName)) { // 必须有 type
        error("Tag 'qualifier' must have a 'type' attribute", ele);
        return;
    }
    this.parseState.push(new QualifierEntry(typeName));
    try {
        // 创建 AutowireCandidateQualifier 对象
        AutowireCandidateQualifier qualifier = new AutowireCandidateQualifier(typeName);
        qualifier.setSource(extractSource(ele));
        // 解析 value 属性，并设置到 AutowireCandidateQualifier 中
        String value = ele.getAttribute(VALUE_ATTRIBUTE);
        if (StringUtils.hasLength(value)) {
            qualifier.setAttribute(AutowireCandidateQualifier.VALUE_KEY, value);
        }
        // 遍历子节点
        NodeList nl = ele.getChildNodes();
        for (int i = 0; i < nl.getLength(); i++) {
            Node node = nl.item(i);
            if (isCandidateElement(node) && nodeNameEquals(node, QUALIFIER_ATTRIBUTE_ELEMENT)) { // attribute 标签
                Element attributeEle = (Element) node;
                String attributeName = attributeEle.getAttribute(KEY_ATTRIBUTE); // attribute 标签的 key 属性
                String attributeValue = attributeEle.getAttribute(VALUE_ATTRIBUTE); // attribute 标签的 value 属性
                if (StringUtils.hasLength(attributeName) && StringUtils.hasLength(attributeValue)) {
                    // 创建 BeanMetadataAttribute 对象
                    BeanMetadataAttribute attribute = new BeanMetadataAttribute(attributeName, attributeValue);
                    attribute.setSource(extractSource(attributeEle));
                    // 添加到 attributes 中
                    qualifier.addMetadataAttribute(attribute);
                } else {
                    error("Qualifier 'attribute' tag must have a 'name' and 'value'", attributeEle);
                    return;
                }
            }
        }
        // 添加到 qualifiers 中
        bd.addQualifier(qualifier);
    } finally {
        this.parseState.pop();
    }
}
```

代码比较简单，胖友自己研究下哈。

# 5、解析自定义标签

DefaultBeanDefinitionDocumentReader 的 `#processBeanDefinition(Element ele, BeanDefinitionParserDelegate delegate)` 方法，负责 `<bean>` 标签的解析：

- 在解析过程中，首先调用 `BeanDefinitionParserDelegate#parseBeanDefinitionElement(Element ele)` 方法，完成**默认标签**的解析。
- 如果解析成功（返回的 `bdHolder != null` ），则调用 `BeanDefinitionParserDelegate#decorateBeanDefinitionIfRequired(Element ele, BeanDefinitionHolder definitionHolder)` 方法，完成**自定义标签**元素的解析。

## 1. decorateBeanDefinitionIfRequired

前面四篇文章已经分析了默认标签的解析，**所以这篇文章分析自定义标签的解析**。代码如下：

```
// BeanDefinitionParserDelegate.java

public BeanDefinitionHolder decorateBeanDefinitionIfRequired(
        Element ele, BeanDefinitionHolder definitionHolder, @Nullable BeanDefinition containingBd) {

    BeanDefinitionHolder finalDefinition = definitionHolder;

    // <1> 遍历属性，查看是否有适用于装饰的【属性】
    // Decorate based on custom attributes first.
    NamedNodeMap attributes = ele.getAttributes();
    for (int i = 0; i < attributes.getLength(); i++) {
        Node node = attributes.item(i);
        finalDefinition = decorateIfRequired(node, finalDefinition, containingBd);
    }

    // <2> 遍历子节点，查看是否有适用于修饰的【子节点】
    // Decorate based on custom nested elements.
    NodeList children = ele.getChildNodes();
    for (int i = 0; i < children.getLength(); i++) {
        Node node = children.item(i);
        if (node.getNodeType() == Node.ELEMENT_NODE) {
            finalDefinition = decorateIfRequired(node, finalDefinition, containingBd);
        }
    }
    return finalDefinition;
}
```

- `<1>` 和 `<2>` 处，都是遍历，前者遍历的是属性( `attributes` )，后者遍历的是子节点( `childNodes` )，最终调用的都是 `#decorateIfRequired(Node node, BeanDefinitionHolder originalDef, BeanDefinition containingBd)` 方法，装饰对应的节点( Node )。详细解析，见 [「2. decorateIfRequired」](http://svip.iocoder.cn/Spring/IoC-parse-BeanDefinitions-for-decorateBeanDefinitionIfRequired/#) 。

## 2. decorateIfRequired

`#decorateIfRequired(Node node, BeanDefinitionHolder originalDef, BeanDefinition containingBd)` 方法，装饰对应的节点( Node )。代码如下：

```
// BeanDefinitionParserDelegate.java

public BeanDefinitionHolder decorateIfRequired(
        Node node, BeanDefinitionHolder originalDef, @Nullable BeanDefinition containingBd) {
    // <1> 获取自定义标签的命名空间
    String namespaceUri = getNamespaceURI(node);
    // <2> 过滤掉默认命名标签
    if (namespaceUri != null && !isDefaultNamespace(namespaceUri)) {
        // <2> 获取相应的处理器
        NamespaceHandler handler = this.readerContext.getNamespaceHandlerResolver().resolve(namespaceUri);
        if (handler != null) {
            // <3> 进行装饰处理
            BeanDefinitionHolder decorated =
                    handler.decorate(node, originalDef, new ParserContext(this.readerContext, this, containingBd));
            if (decorated != null) {
                return decorated;
            }
        } else if (namespaceUri.startsWith("http://www.springframework.org/")) {
            error("Unable to locate Spring NamespaceHandler for XML schema namespace [" + namespaceUri + "]", node);
        } else {
            // A custom namespace, not to be handled by Spring - maybe "xml:...".
            if (logger.isDebugEnabled()) {
                logger.debug("No Spring NamespaceHandler found for XML schema namespace [" + namespaceUri + "]");
            }
        }
    }
    return originalDef;
}
```

- 在 `<1>` 处，首先获取自定义标签的命名空间。
- 在 `<2>` 处，如果**不是默认的命名空间**，则根据该命名空间获取相应的处理器。
- 在 `<3>` 处，如果处理器存在，则进行装饰处理。

上述过程的详细解析，见 [《【死磕 Spring】—— IoC 之解析 标签：解析自定义标签》](http://svip.iocoder.cn/Spring/IoC-parse-BeanDefinitions-in-parseCustomElement) 一文。

## 3. 小结

至此，BeanDefinition 的解析过程已经全部完成了，下面做一个简要的总结：

解析 BeanDefinition 的入口在 DefaultBeanDefinitionDocumentReader 的`#parseBeanDefinitions(Element root, BeanDefinitionParserDelegate delegate)` 方法。该方法会根据命令空间来判断标签是默认标签还是自定义标签，其中：

- 默认标签，由 `#parseDefaultElement(Element ele, BeanDefinitionParserDelegate delegate)` 方法来实现
- 自定义标签，由 BeanDefinitionParserDelegate 的 `#parseCustomElement(Element ele, @Nullable BeanDefinition containingBd)` 方法来实现。

在默认标签解析中，会根据标签名称的不同进行 `import`、`alias`、`bean`、`beans` 四大标签进行处理。其中 `bean` 标签的解析为核心，它由 `processBeanDefinition(Element ele, BeanDefinitionParserDelegate delegate)` 方法实现。

`processBeanDefinition(Element ele, BeanDefinitionParserDelegate delegate)` 方法，开始进入解析核心工作，分为三步：

1. 解析默认标签的**默认**标签：`BeanDefinitionParserDelegate#parseBeanDefinitionElement(Element ele, ...)` 方法。该方法会依次解析 `<bean>` 标签的属性、各个子元素，解析完成后返回一个 GenericBeanDefinition 实例对象。
2. 解析默认标签下的**自定义**标签：`BeanDefinitionParserDelegate#decorateBeanDefinitionIfRequired(Element ele, BeanDefinitionHolder definitionHolder)` 方法。
3. 注册解析的 BeanDefinition：`BeanDefinitionReaderUtils#registerBeanDefinition(BeanDefinitionHolder definitionHolder, BeanDefinitionRegistry registry)` 方法。

# 6、IoC 之解析自定义标签

## 1. 使用自定义标签

扩展 Spring 自定义标签配置一般需要以下几个步骤：

1. 创建一个需要扩展的组件。
2. 定义一个 XSD 文件，用于描述组件内容。
3. 创建一个实现 `org.springframework.beans.factory.xml.AbstractSingleBeanDefinitionParser` 接口的类，用来解析 XSD 文件中的定义和组件定义。
4. 创建一个 Handler，继承 `org.springframework.beans.factory.xml.NamespaceHandlerSupport` 抽象类 ，用于将组件注册到 Spring 容器。
5. 编写 `spring.handlers` 和 `Spring.schemas` 文件。

下面就按照上面的步骤来实现一个自定义标签组件。

#### 1.1 创建组件

该组件就是一个普通的 Java Bean，没有任何特别之处。代码如下：

```
public class User {

    private String id;
    private String userName;
    private String email;

}
```

#### 1.2 定义 XSD 文件

```
<?xml version="1.0" encoding="UTF-8"?>
<xsd:schema xmlns:xsd="http://www.w3.org/2001/XMLSchema" xmlns="http://www.cmsblogs.com/schema/user" targetNamespace="http://www.cmsblogs.com/schema/user" elementFormDefault="qualified">

    <xsd:element name="user">
        <xsd:complexType>
            <xsd:attribute name="id" type="xsd:string" />
            <xsd:attribute name="userName" type="xsd:string" />
            <xsd:attribute name="email" type="xsd:string" />
        </xsd:complexType>
    </xsd:element>

</xsd:schema>
```

上面除了对 User 这个 Java Bean 进行了描述外，还定义了 `xmlns="http://www.cmsblogs.com/schema/user"` 和 `targetNamespace="http://www.cmsblogs.com/schema/user"` 这两个值，这两个值在后面是有大作用的。

#### 1.3 定义 Parser 类

定义一个 Parser 类，该类继承 AbstractSingleBeanDefinitionParser ，并实现 `#getBeanClass(Element element)` 和 `#doParse(Element element, BeanDefinitionBuilder builder)` 两个方法。主要是用于解析 XSD 文件中的定义和组件定义。

```
public class UserDefinitionParser extends AbstractSingleBeanDefinitionParser {

    @Override
    protected Class<?> getBeanClass(Element element) {
        return User.class;
    }

    @Override
    protected void doParse(Element element, BeanDefinitionBuilder builder) {
        String id = element.getAttribute("id");
        String userName = element.getAttribute("userName");
        String email = element.getAttribute("email");

        if (StringUtils.hasText(id)) {
            builder.addPropertyValue("id", id);
        }

        if (StringUtils.hasText(userName)) {
            builder.addPropertyValue("userName", userName);
        }

        if (StringUtils.hasText(email)) {
            builder.addPropertyValue("email", email);
        }
    }

}
```

#### 1.4 定义 NamespaceHandler 类

定义 NamespaceHandler 类，继承 NamespaceHandlerSupport ,主要目的是将组件注册到 Spring 容器中。

```
public class UserNamespaceHandler extends NamespaceHandlerSupport {

    @Override
    public void init() {
        registerBeanDefinitionParser("user", new UserDefinitionParser());
    }

}
```

#### 1.5 定义 spring.handlers 文件

```
http\://www.cmsblogs.com/schema/user=org.springframework.core.customelement.UserNamespaceHandler
```

#### 1.6 定义 Spring.schemas 文件

```
http\://www.cmsblogs.com/schema/user.xsd=user.xsd
```

#### 1.7 运行

经过上面几个步骤，就可以使用自定义的标签了。在 xml 配置文件中使用如下：

```
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:myTag="http://www.cmsblogs.com/schema/user"
       xsi:schemaLocation="http://www.springframework.org/schema/beans
       http://www.springframework.org/schema/beans/spring-beans.xsd
        http://www.cmsblogs.com/schema/user http://www.cmsblogs.com/schema/user.xsd">

    <myTag:user id="user" email="12233445566@qq.com" userName="chenssy" />

</beans>
```

运行测试：

```
public static void main(String[] args){
    ApplicationContext context = new ClassPathXmlApplicationContext("spring.xml");
    User user = (User) context.getBean("user");
    System.out.println(user.getUserName() + "----" + user.getEmail());
}
```

运行结果如下图：

[![运行结果](http://static.iocoder.cn/64a6aa7283cd9e6ae5070162ae007872)](http://static.iocoder.cn/64a6aa7283cd9e6ae5070162ae007872)运行结果

## 2. 解析自定义标签

上面已经演示了 Spring 自定义标签的使用，下面就来分析自定义标签的解析过程。

#### 2.1 parseCustomElement

DefaultBeanDefinitionDocumentReader 的`#parseBeanDefinitions(Element root, BeanDefinitionParserDelegate delegate)` 方法，负责标签的解析工作，根据命名空间的不同进行不同标签的解析。其中，**自定义标签**由 BeanDefinitionParserDelegate 的 `#parseCustomElement(Element ele, BeanDefinition containingBd)` 方法来实现。代码如下：

```
@Nullable
public BeanDefinition parseCustomElement(Element ele) {
    return parseCustomElement(ele, null);
}

@Nullable
public BeanDefinition parseCustomElement(Element ele, @Nullable BeanDefinition containingBd) {
    // <1> 获取 namespaceUri
    String namespaceUri = getNamespaceURI(ele);
    if (namespaceUri == null) {
        return null;
    }
    // <2> 根据 namespaceUri 获取相应的 Handler
    NamespaceHandler handler = this.readerContext.getNamespaceHandlerResolver().resolve(namespaceUri);
    if (handler == null) {
        error("Unable to locate Spring NamespaceHandler for XML schema namespace [" + namespaceUri + "]", ele);
        return null;
    }
    // 调用自定义的 Handler 处理
    return handler.parse(ele, new ParserContext(this.readerContext, this, containingBd));
}
```

处理过程分为三步：

1. 调用 `#getNamespaceURI((Node node)` 方法，获取 `namespaceUri` 。代码如下：

   ```
   @Nullable
   public String getNamespaceURI(Node node) {
   	return node.getNamespaceURI();
   }
   ```

2. 调用 `XmlReaderContext#getNamespaceHandlerResolver()` 方法，获得命名空间的解析器。详细解析，见 [「2.2 getNamespaceHandlerResolver」](http://svip.iocoder.cn/Spring/IoC-parse-BeanDefinitions-in-parseCustomElement/#) 。

3. 调用 `NamespaceHandlerResolver#resolve(String namespaceUri)` 方法，根据 `namespaceUri` 获取相应的 Handler 对象。这个映射关系我们在 `spring.handlers` 中已经定义了，所以只需要找到该类，然后初始化返回。详细解析，见 [「2.3 resolve」](http://svip.iocoder.cn/Spring/IoC-parse-BeanDefinitions-in-parseCustomElement/#) 。

4. 调用 `NamespaceHandler#parse(Element element, ParserContext parserContext)` 方法，调用自定义的 Handler 处理。详细解析，见 [「2.4 parse」](http://svip.iocoder.cn/Spring/IoC-parse-BeanDefinitions-in-parseCustomElement/#) 。

#### 2.2 getNamespaceHandlerResolver

调用 XmlReaderContext 的 `#getNamespaceHandlerResolver()` 方法，返回的命名空间的解析器，代码如下：

```
/**
 * NamespaceHandler 解析器
 */
private final NamespaceHandlerResolver namespaceHandlerResolver;

public final NamespaceHandlerResolver getNamespaceHandlerResolver() {
	return this.namespaceHandlerResolver;
}
```

###### 2.2.1 NamespaceHandlerResolver 的初始化

那么，NamespaceHandlerResolver 是什么时候进行初始化的呢？

这里需要回退到博文 [《【死磕 Spring】—— IoC 之注册 BeanDefinitions》](http://svip.iocoder.cn/Spring/IoC-register-BeanDefinitions) ，在这篇博客中提到在注册 BeanDefinition 时：

- 首先，是通过 XmlBeanDefinitionReader 的 `#createBeanDefinitionDocumentReader()` 方法，获取 Document 解析器 BeanDefinitionDocumentReader 实例。
- 然后，调用 BeanDefinitionDocumentReader 实例的 `#registerBeanDefinitions(Document doc, XmlReaderContext readerContext)` 方法，进行注册。而该方法需要提供两个参数，一个是 Document 实例 `doc`，一个是 XmlReaderContext 实例 `readerContext` 。

`readerContext` 实例对象由 XmlBeanDefinitionReader 的 `#createReaderContext(Resource resource)` 方法创建。`namespaceHandlerResolver` 实例对象就是在这个时候初始化的，代码如下：

```
// XmlBeanDefinitionReader.java
public XmlReaderContext createReaderContext(Resource resource) {
	return new XmlReaderContext(resource, this.problemReporter, this.eventListener,
			this.sourceExtractor, this, getNamespaceHandlerResolver());
}
```

- XmlReaderContext 构造函数中最后一个参数就是 NamespaceHandlerResolver 对象，该对象由 `getNamespaceHandlerResolver()` 提供，如下：

  ```
  // XmlBeanDefinitionReader.java
  
  public NamespaceHandlerResolver getNamespaceHandlerResolver() {
  	if (this.namespaceHandlerResolver == null) {
  		this.namespaceHandlerResolver = createDefaultNamespaceHandlerResolver();
  	}
  	return this.namespaceHandlerResolver;
  }
  
  protected NamespaceHandlerResolver createDefaultNamespaceHandlerResolver() {
  	ClassLoader cl = (getResourceLoader() != null ? getResourceLoader().getClassLoader() : getBeanClassLoader());
  	return new DefaultNamespaceHandlerResolver(cl); // <x>
  }
  ```

  - 从 `<x>` 处，我们可以看到，NamespaceHandlerResolver 对象的**最终类型**是 `org.springframework.beans.factory.xml.DefaultNamespaceHandlerResolver` 。

#### 2.3 resolve

所以， `getNamespaceHandlerResolver().resolve(namespaceUri)` 调用的就是 DefaultNamespaceHandlerResolver 的 `#resolve(String namespaceUri)` 方法。代码如下：

```
@Override
@Nullable
public NamespaceHandler resolve(String namespaceUri) {
    // <1> 获取所有已经配置的 Handler 映射
    Map<String, Object> handlerMappings = getHandlerMappings();
    // <2> 根据 namespaceUri 获取 handler 的信息
    Object handlerOrClassName = handlerMappings.get(namespaceUri);
    // <3.1> 不存在
    if (handlerOrClassName == null) {
        return null;
    // <3.2> 已经初始化
    } else if (handlerOrClassName instanceof NamespaceHandler) {
        return (NamespaceHandler) handlerOrClassName;
    // <3.3> 需要进行初始化
    } else {
        String className = (String) handlerOrClassName;
        try {
            // 获得类，并创建 NamespaceHandler 对象
            Class<?> handlerClass = ClassUtils.forName(className, this.classLoader);
            if (!NamespaceHandler.class.isAssignableFrom(handlerClass)) {
                throw new FatalBeanException("Class [" + className + "] for namespace [" + namespaceUri +
                        "] does not implement the [" + NamespaceHandler.class.getName() + "] interface");
            }
            NamespaceHandler namespaceHandler = (NamespaceHandler) BeanUtils.instantiateClass(handlerClass);
            // 初始化 NamespaceHandler 对象
            namespaceHandler.init();
            // 添加到缓存
            handlerMappings.put(namespaceUri, namespaceHandler);
            return namespaceHandler;
        } catch (ClassNotFoundException ex) {
            throw new FatalBeanException("Could not find NamespaceHandler class [" + className +
                    "] for namespace [" + namespaceUri + "]", ex);
        } catch (LinkageError err) {
            throw new FatalBeanException("Unresolvable class definition for NamespaceHandler class [" +
                    className + "] for namespace [" + namespaceUri + "]", err);
        }
    }
}
```

- `<1>` 处，首先，调用 `#getHandlerMappings()` 方法，获取所有配置文件中的映射关系 `handlerMappings` 。详细解析，胖友先跳到 [「2.3.1 getHandlerMappings」](http://svip.iocoder.cn/Spring/IoC-parse-BeanDefinitions-in-parseCustomElement/#) ，看完就回到此处，继续往下走。

- `<2>` 处，然后，根据 `namespaceUri` 获取 handler 的信息。

- `<3.1>` 处，`handlerOrClassName` 不存在，则返回 `null` 空。

- `<3.2>` 处，`handlerOrClassName` 已经初始化成 NamespaceHandler 对象，直接返回它。

- ```
  <3.3>
  ```

   

  处，

  ```
  handlerOrClassName
  ```

   

  还是类路径，则创建 NamespaceHandler 对象，并调用

   

  ```
  NamespaceHandler#init()
  ```

   

  方法，初始化 NamespaceHandler 对象。详细解析，见

   

  「2.3.2 init」

   

  。

  - 另外，创建的 NamespaceHandler 对象，会添加到 `handlerMappings` 中，进行缓存。

###### 2.3.1 getHandlerMappings

```
/** ClassLoader to use for NamespaceHandler classes. */
@Nullable
private final ClassLoader classLoader;

/**
 * NamespaceHandler 映射配置文件地址
 *
 * Resource location to search for.
 */
private final String handlerMappingsLocation;

/**
 * Stores the mappings from namespace URI to NamespaceHandler class name / instance.
 *
 * NamespaceHandler 映射。
 *
 * key：命名空间
 * value：分成两种情况：1）未初始化时，对应的 NamespaceHandler 的类路径；2）已初始化，对应的 NamespaceHandler 对象
 */
@Nullable
private volatile Map<String, Object> handlerMappings;

/**
 * Load the specified NamespaceHandler mappings lazily.
 */
private Map<String, Object> getHandlerMappings() {
	// 双重检查锁，延迟加载
	Map<String, Object> handlerMappings = this.handlerMappings;
	if (handlerMappings == null) {
		synchronized (this) {
			handlerMappings = this.handlerMappings;
			if (handlerMappings == null) {
				if (logger.isTraceEnabled()) {
					logger.trace("Loading NamespaceHandler mappings from [" + this.handlerMappingsLocation + "]");
				}
				try {
					// 读取 handlerMappingsLocation
					Properties mappings = PropertiesLoaderUtils.loadAllProperties(this.handlerMappingsLocation, this.classLoader);
					if (logger.isTraceEnabled()) {
						logger.trace("Loaded NamespaceHandler mappings: " + mappings);
					}
					// 初始化到 handlerMappings 中
					handlerMappings = new ConcurrentHashMap<>(mappings.size());
					CollectionUtils.mergePropertiesIntoMap(mappings, handlerMappings);
					this.handlerMappings = handlerMappings;
				} catch (IOException ex) {
					throw new IllegalStateException(
							"Unable to load NamespaceHandler mappings from location [" + this.handlerMappingsLocation + "]", ex);
				}
			}
		}
	}
	return handlerMappings;
}
```

- 虽然代码比较长，但是逻辑实际很简单。
- 通过延迟加载( lazy-init )的方式，加载 `handlerMappingsLocation` 中配置的 NamespaceHandler 的映射，到 `handlerMappings` 中。
- `handlerMappings` 的**值属性**有 2 种情况，胖友仔细看下注释。

###### 2.3.2 init

实现 NamespaceHandler 的 `#init()` 方法，主要是将自定义标签解析器进行注册。例如，我们自定义 UserNamespaceHandler 的 `#init()` 方法，代码如下：

```
// UserNamespaceHandler.java

@Override
public void init() {
    registerBeanDefinitionParser("user",new UserDefinitionParser());
}
```

- 直接调用父类 NamespaceHandlerSupport 的 `#registerBeanDefinitionParser(String elementName, BeanDefinitionParser parser)` 方法，注册指定元素的 BeanDefinitionParser 解析器。

####### 2.3.2.1 registerBeanDefinitionParser

NamespaceHandlerSupport 的 `#registerBeanDefinitionParser(String elementName, BeanDefinitionParser parser)` 方法，注册指定元素的 BeanDefinitionParser 解析器。代码如下：

```
// NamespaceHandlerSupport.java
/**
 * Stores the {@link BeanDefinitionParser} implementations keyed by the
 * local name of the {@link Element Elements} they handle.
 *
 * key：元素名
 * value：对应 BeanDefinitionParser 的解析器
 */
private final Map<String, BeanDefinitionParser> parsers = new HashMap<>();

protected final void registerBeanDefinitionParser(String elementName, BeanDefinitionParser parser) {
	this.parsers.put(elementName, parser);
}
```

- 其实就是将映射关系放在一个 Map 结构的 `parsers` 对象中。

#### 2.4 parse

完成后返回 NamespaceHandler 对象，然后调用其 `#parse(Element element, ParserContext parserContext)` 方法开始自定义标签的解析。代码如下：

```
// NamespaceHandlerSupport.java
@Override
@Nullable
public BeanDefinition parse(Element element, ParserContext parserContext) {
	// <1> 获得元素对应的 BeanDefinitionParser 对象
	BeanDefinitionParser parser = findParserForElement(element, parserContext);
	// <2> 执行解析
	return (parser != null ? parser.parse(element, parserContext) : null);
}
```

- `<1>` 处，调用 `#findParserForElement(Element element, ParserContext parserContext)` 方法，获取对应的 BeanDefinitionParser 实例。实际上，其实就是获取在 NamespaceHandlerSupport 的 `#registerBeanDefinitionParser()` 方法里面注册的实例对象。代码如下：

  ```
    /**
     * Locates the {@link BeanDefinitionParser} from the register implementations using
     * the local name of the supplied {@link Element}.
     */
    @Nullable
    private BeanDefinitionParser findParserForElement(Element element, ParserContext parserContext) {
  // 获得元素名
    	String localName = parserContext.getDelegate().getLocalName(element);
    	// 获得 BeanDefinitionParser 对象
    	BeanDefinitionParser parser = this.parsers.get(localName);
    	if (parser == null) {
    		parserContext.getReaderContext().fatal(
    				"Cannot locate BeanDefinitionParser for element [" + localName + "]", element);
    	}
    	return parser;
    }
  ```

  - 首先，获取 `localName`，在上面的例子中就是：`"user` 。
  - 然后，从 Map 实例 `parsers` 中获取 BeanDefinitionParser 对象。在上面的例子中就是：UserBeanDefinitionParser 对象。

- `<2>` 处，返回 BeanDefinitionParser 对象后，调用其 `#parse(Element element, ParserContext parserContext)` 方法。该方法在 AbstractBeanDefinitionParser 中实现，代码如下：

  ```
    // AbstractBeanDefinitionParser.java
  
    @Override
    @Nullable
    public final BeanDefinition parse(Element element, ParserContext parserContext) {
  // <1> 内部解析，返回 AbstractBeanDefinition 对象
    	AbstractBeanDefinition definition = parseInternal(element, parserContext);
    	if (definition != null && !parserContext.isNested()) {
    		try {
    		    // 解析 id 属性
    			String id = resolveId(element, definition, parserContext);
    			if (!StringUtils.hasText(id)) {
    				parserContext.getReaderContext().error(
    						"Id is required for element '" + parserContext.getDelegate().getLocalName(element)
    								+ "' when used as a top-level tag", element);
    			}
    			// 解析 aliases 属性
    			String[] aliases = null;
    			if (shouldParseNameAsAliases()) {
    				String name = element.getAttribute(NAME_ATTRIBUTE);
    				if (StringUtils.hasLength(name)) {
    					aliases = StringUtils.trimArrayElements(StringUtils.commaDelimitedListToStringArray(name));
    				}
    			}
    			// 创建 BeanDefinitionHolder 对象
    			BeanDefinitionHolder holder = new BeanDefinitionHolder(definition, id, aliases);
    			// 注册 BeanDefinition
    			registerBeanDefinition(holder, parserContext.getRegistry());
    			// 触发事件
    			if (shouldFireEvents()) {
    				BeanComponentDefinition componentDefinition = new BeanComponentDefinition(holder);
    				postProcessComponentDefinition(componentDefinition);
    				parserContext.registerComponent(componentDefinition);
    			}
    		} catch (BeanDefinitionStoreException ex) {
    			String msg = ex.getMessage();
    			parserContext.getReaderContext().error((msg != null ? msg : ex.toString()), element);
    			return null;
    		}
    	}
    	return definition;
    }
  ```

  - 核心在 `<1>` 处 `#parseInternal(Element element, ParserContext parserContext)` 方法。为什么这么说？因为该方法返回的是 AbstractBeanDefinition 对象。从前面**默认标签**的解析过程来看，我们就可以判断该方法就是将标签解析为 AbstractBeanDefinition ，且后续代码都是将 AbstractBeanDefinition 转换为 BeanDefinitionHolder 对象。所以真正的解析工作都交由 `#parseInternal(Element element, ParserContext parserContext)` 方法来实现。关于该方法，详细解析，见 [「2.4.1 parseInternal」](http://svip.iocoder.cn/Spring/IoC-parse-BeanDefinitions-in-parseCustomElement/#) 。
  - 其它逻辑，例如 `#resolveId(Element element, AbstractBeanDefinition definition, ParserContext parserContext)` 方法，都比较简单，感兴趣的胖友，可以自己去看。

###### 2.4.1 parseInternal

`#parseInternal(Element element, ParserContext parserContext)` 方法，解析 XML 元素为 AbstractBeanDefinition 对象。代码如下：

```
// AbstractSingleBeanDefinitionParser.java

@Override
protected final AbstractBeanDefinition parseInternal(Element element, ParserContext parserContext) {
    // 创建 BeanDefinitionBuilder 对象
    BeanDefinitionBuilder builder = BeanDefinitionBuilder.genericBeanDefinition();
    // 获取父类元素
    String parentName = getParentName(element);
    if (parentName != null) {
        builder.getRawBeanDefinition().setParentName(parentName);
    }
    // 获取自定义标签中的 class，这个时候会去调用自定义解析中的 getBeanClass()
    Class<?> beanClass = getBeanClass(element);
    if (beanClass != null) {
        builder.getRawBeanDefinition().setBeanClass(beanClass);
    } else {
        // beanClass 为 null，意味着子类并没有重写 getBeanClass() 方法，则尝试去判断是否重写了 getBeanClassName()
        String beanClassName = getBeanClassName(element);
        if (beanClassName != null) {
            builder.getRawBeanDefinition().setBeanClassName(beanClassName);
        }
    }
    // 设置 source 属性
    builder.getRawBeanDefinition().setSource(parserContext.extractSource(element));
    // 设置 scope 属性
    BeanDefinition containingBd = parserContext.getContainingBeanDefinition();
    if (containingBd != null) {
        // Inner bean definition must receive same scope as containing bean.
        builder.setScope(containingBd.getScope());
    }
    // 设置 lazy-init 属性
    if (parserContext.isDefaultLazyInit()) {
        // Default-lazy-init applies to custom bean definitions as well.
        builder.setLazyInit(true);
    }
    // 调用子类的 doParse() 进行解析
    doParse(element, parserContext, builder);
    return builder.getBeanDefinition();
}
```

- 在该方法中我们主要关注两个方法：`#getBeanClass((Element element)` 、`#doParse(Element element, BeanDefinitionBuilder builder)`。
- 对于 `getBeanClass()` 方法，AbstractSingleBeanDefinitionParser 类并没有提供具体实现，而是直接返回 `null` ，**意味着它希望子类能够重写该方法**。当然，如果没有重写该方法，这会去调用 `#getBeanClassName()` ，判断子类是否已经重写了该方法。
- 对于 `#doParse(Element element, BeanDefinitionBuilder builder)` 方法，则是直接**空实现**。

😈 所以对于 `#parseInternal(Element element, ParserContext parserContext)` 方法 而言，它总是期待它的子类能够实现 `#getBeanClass((Element element)` 、`#doParse(Element element, BeanDefinitionBuilder builder)` 方法。其中，`#doParse(Element element, BeanDefinitionBuilder builder)` **方法尤为重要**！如果，你不提供该方法的实现，怎么来解析自定义标签呢？此时，胖友可以回过头，再看一眼在 [「1.3 定义 Parser 类」](http://svip.iocoder.cn/Spring/IoC-parse-BeanDefinitions-in-parseCustomElement/#) 的 UserDefinitionParser 实现类，是不是已经能够很好理解咧。

## 3. 小结

至此，自定义标签的解析过程已经分析完成了。其实整个过程还是较为简单：

- 首先，会加载 `spring.handlers` 文件，将其中内容进行一个解析，形成 `<namespaceUri, 类路径>` 这样的一个映射。
- 然后，根据获取的 `namespaceUri` 就可以得到相应的类路径，对其进行初始化等到相应的 NamespaceHandler 对象。
- 之后，调用该 NamespaceHandler 的 `#parse(...)` 方法，在该方法中根据标签的 `localName` 得到相应的 BeanDefinitionParser 实例对象。
- 最后，调用该 BeanDefinitionParser 的 `#parse(...)` 方法。该方法定义在 AbstractBeanDefinitionParser 抽象类中，核心逻辑封装在其 `#parseInternal(...)` 方法中，该方法返回一个 AbstractBeanDefinition 实例对象，其主要是在 AbstractSingleBeanDefinitionParser 中实现。对于自定义的 Parser 类，其需要实现 `#getBeanClass()` 或者 `#getBeanClassName()` 任一方法，和 `#doParse(...)` 方法。

整体流程如[下图](https://gitee.com/chenssy/blog-home/raw/master/image/201811/spring-201807151001.png)：

[![spring-201807151001](http://static.iocoder.cn/b53b2484800ee0e35be4510d9ebe2891)](http://static.iocoder.cn/b53b2484800ee0e35be4510d9ebe2891)