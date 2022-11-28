# 1、Aware 接口

**本文主要基于 Spring 5.0.6.RELEASE**

摘要: 原创出处 http://cmsblogs.com/?p=3335 「小明哥」，谢谢！

作为「小明哥」的忠实读者，「老艿艿」略作修改，记录在理解过程中，参考的资料。

------

﻿AbstractAutowireCapableBeanFactory 的 `#doCreateBean(final String beanName, final RootBeanDefinition mbd, final Object[] args)` 方法，主要干三件事情：

1. 实例化 bean 对象：`#createBeanInstance(String beanName, RootBeanDefinition mbd, Object[] args)` 方法。
2. 属性注入：`#populateBean(String beanName, RootBeanDefinition mbd, BeanWrapper bw)` 方法。
3. 初始化 bean 对象：`#initializeBean(final String beanName, final Object bean, RootBeanDefinition mbd)` 方法。

而初始化 bean 对象时，也是干了三件事情：

1. 激活 Aware 方法
2. 后置处理器的应用
3. 激活自定义的 init 方法

接下来三篇文章将会详细分析这三件事情，这篇主要分析 Aware 接口。

## 1. Aware 接口

`org.springframework.beans.factory.Aware` 接口，定义如下：

```
/**
 * Marker superinterface indicating that a bean is eligible to be
 * notified by the Spring container of a particular framework object
 * through a callback-style method. Actual method signature is
 * determined by individual subinterfaces, but should typically
 * consist of just one void-returning method that accepts a single
 * argument.
 *
 * <p>Note that merely implementing {@link Aware} provides no default
 * functionality. Rather, processing must be done explicitly, for example
 * in a {@link org.springframework.beans.factory.config.BeanPostProcessor BeanPostProcessor}.
 * Refer to {@link org.springframework.context.support.ApplicationContextAwareProcessor}
 * and {@link org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory}
 * for examples of processing {@code *Aware} interface callbacks.
 *
 * @author Chris Beams
 * @since 3.1
 */
public interface Aware {

}
```

Aware 接口为 Spring 容器的核心接口，是一个具有标识作用的超级接口，实现了该接口的 bean 是具有被 Spring 容器通知的能力，通知的方式是采用**回调**的方式。

Aware 接口是一个空接口，实际的方法签名由各个子接口来确定，且该接口通常只会有一个接收单参数的 set 方法，该 set 方法的命名方式为 set + 去掉接口名中的 Aware 后缀，即 XxxAware 接口，则方法定义为 setXxx()，例如 BeanNameAware（setBeanName），ApplicationContextAware（setApplicationContext）。

Aware 的子接口需要提供一个 `setXxx` 方法，我们知道 set 是设置属性值的方法，即 Aware 类接口的 `setXxx` 方法其实就是设置 xxx 属性值的。 Aware 的含义是感知的、感应的，那么在 Spring 容器中是如何实现感知并设置属性值得呢？我们可以从初始化 bean 中的激活 Aware 的方法 `#invokeAwareMethods(final String beanName, final Object bean)` 中看到一点点，代码如下：

```
// AbstractAutowireCapableBeanFactory.java

private void invokeAwareMethods(final String beanName, final Object bean) {
	if (bean instanceof Aware) {
	    // BeanNameAware
		if (bean instanceof BeanNameAware) {
			((BeanNameAware) bean).setBeanName(beanName);
		}
		// BeanClassLoaderAware
		if (bean instanceof BeanClassLoaderAware) {
			ClassLoader bcl = getBeanClassLoader();
			if (bcl != null) {
				((BeanClassLoaderAware) bean).setBeanClassLoader(bcl);
			}
		}
		// BeanFactoryAware
		if (bean instanceof BeanFactoryAware) {
			((BeanFactoryAware) bean).setBeanFactory(AbstractAutowireCapableBeanFactory.this);
		}
	}
}
```

- 首先，判断 bean 实例是否属于 Aware 接口的范畴，如果是的话，则调用实例的 `setXxx()` 方法给实例设置 xxx 属性值，在 `#invokeAwareMethods(...)` 方法，主要是设置 beanName，beanClassLoader、BeanFactory 中三个属性值。

## 2. Aware 子类

Spring 提供了一系列的 Aware 接口，如下图（部分）：

[![201808210001](http://static.iocoder.cn/242d5c99b48d717a4e1ce7782cb57bab)](http://static.iocoder.cn/242d5c99b48d717a4e1ce7782cb57bab)201808210001

上面只是一部分子类，从这里我们可以看到 Spring 提供的 Aware 接口是是何其多。同时从上图我们也看到了几个比较熟悉的接口，如 BeanClassLoaderAware、BeanFactoryAware、BeanNameAware，下面就这三个接口来做一个简单的演示，先看各自的定义：

```
public interface BeanClassLoaderAware extends Aware {

    /**
    * 将 BeanClassLoader 提供给 bean 实例回调
    * 在 bean 属性填充之后、初始化回调之前回调，
    * 例如InitializingBean的InitializingBean.afterPropertiesSet（）方法或自定义init方法
    */
    void setBeanClassLoader(ClassLoader classLoader);

}


public interface BeanFactoryAware extends Aware {
    /**
    * 将 BeanFactory 提供给 bean 实例回调
    * 调用时机和 setBeanClassLoader 一样
    */
    void setBeanFactory(BeanFactory beanFactory) throws BeansException;

}


public interface BeanNameAware extends Aware {

    /**
    * 在创建此 bean 的 bean工厂中设置 beanName
    */
    void setBeanName(String name);

}


public interface ApplicationContextAware extends Aware {

    /**
     * 设置此 bean 对象的 ApplicationContext，通常，该方法用于初始化对象
     */
    void setApplicationContext(ApplicationContext applicationContext)
        throws BeansException;

}
```

### 2.1 示例

下面简单演示下上面四个接口的使用方法：

```
public class MyApplicationAware implements BeanNameAware,BeanFactoryAware,BeanClassLoaderAware,ApplicationContextAware{

    private String beanName;

    private BeanFactory beanFactory;

    private ClassLoader classLoader;

    private ApplicationContext applicationContext;

    @Override
    public void setBeanClassLoader(ClassLoader classLoader) {
        System.out.println("调用了 BeanClassLoaderAware 的 setBeanClassLoader 方法");
        this.classLoader = classLoader;
    }

    @Override
    public void setBeanFactory(BeanFactory beanFactory) throws BeansException {
        System.out.println("调用了 BeanFactoryAware 的 setBeanFactory 方法");
        this.beanFactory = beanFactory;
    }

    @Override
    public void setBeanName(String name) {
        System.out.println("调用了 BeanNameAware 的 setBeanName 方法");
        this.beanName = name;
    }

    @Override
    public void setApplicationContext(ApplicationContext applicationContext) throws BeansException {
        System.out.println("调用了 ApplicationContextAware 的 setApplicationContext 方法");
        this.applicationContext = applicationContext;
    }

    public void display(){
        System.out.println("beanName:" + beanName);
        System.out.println("是否为单例：" + beanFactory.isSingleton(beanName));
        System.out.println("系统环境为：" + applicationContext.getEnvironment());
    }
}
```

测试方法如下:

```
public static void main(String[] args) {
    ClassPathResource resource = new ClassPathResource("spring.xml");
    DefaultListableBeanFactory factory = new DefaultListableBeanFactory();
    XmlBeanDefinitionReader reader = new XmlBeanDefinitionReader(factory);
    reader.loadBeanDefinitions(resource);

    MyApplicationAware applicationAware = (MyApplicationAware) factory.getBean("myApplicationAware");
    applicationAware.display();
}
```

运行结果：

[![201808210002](http://static.iocoder.cn/cf1b7d54fc734e654a88c4611c6a5c34)](http://static.iocoder.cn/cf1b7d54fc734e654a88c4611c6a5c34)201808210002

从该运行结果可以看出，这里只执行了三个 Aware 接口的 set 方法，原因就是通过 `#getBean(...)` 方法调用时，在激活 Aware 接口时只检测了 BeanNameAware、BeanClassLoaderAware、BeanFactoryAware 三个 Aware 接口。如果将测试方法调整为下面：

```
public static void main(String[] args) {
    ApplicationContext applicationContext = new ClassPathXmlApplicationContext("spring.xml");
    MyApplicationAware applicationAware = (MyApplicationAware) applicationContext.getBean("myApplicationAware");
    applicationAware.display();
}
```

则运行结果如下：

[![201808210003](http://static.iocoder.cn/cf1b7d54fc734e654a88c4611c6a5c34)](http://static.iocoder.cn/cf1b7d54fc734e654a88c4611c6a5c34)201808210003

## 3. 小结

从这了我们基本上就可以 Aware 真正的含义是什么了？感知，其实是 Spring 容器在初始化主动检测当前 bean 是否实现了 Aware 接口，如果实现了则回调其 set 方法将相应的参数设置给该 bean ，这个时候该 bean 就从 Spring 容器中取得相应的资源。

最后文章末尾列出部分常用的 Aware 子接口，便于日后查询：

- LoadTimeWeaverAware：加载Spring Bean时织入第三方模块，如AspectJ
- BeanClassLoaderAware：加载Spring Bean的类加载器
- BootstrapContextAware：资源适配器BootstrapContext，如JCA,CCI
- ResourceLoaderAware：底层访问资源的加载器
- BeanFactoryAware：声明BeanFactory
- PortletConfigAware：PortletConfig
- PortletContextAware：PortletContext
- ServletConfigAware：ServletConfig
- ServletContextAware：ServletContext
- MessageSourceAware：国际化
- ApplicationEventPublisherAware：应用事件
- NotificationPublisherAware：JMX通知
- BeanNameAware：声明Spring Bean的名字

# 2、BeanPostProcessor

## 1. BeanPostProcessor 接口

﻿Spring 作为优秀的开源框架，它为我们提供了丰富的可扩展点，除了前面提到的 Aware 接口，还包括其他部分，其中一个很重要的就是 BeanPostProcessor。这篇文章主要介绍 BeanPostProcessor 的使用以及其实现原理。我们先看 BeanPostProcessor 的定位：

> BeanPostProcessor 的作用：在 Bean 完成实例化后，如果我们需要对其进行一些配置、增加一些自己的处理逻辑，那么请使用 BeanPostProcessor。

## 2. BeanPostProcessor 示例

首先定义一个类，该类实现 BeanPostProcessor 接口，代码如下：

```
public class BeanPostProcessorTest implements BeanPostProcessor{

    @Override
    public Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException {
        System.out.println("Bean [" + beanName + "] 开始初始化");
        // 这里一定要返回 bean，不能返回 null
        return bean;
    }

    @Override
    public Object postProcessAfterInitialization(Object bean, String beanName) throws BeansException {
        System.out.println("Bean [" + beanName + "] 完成初始化");
        return bean;
    }

    public void display(){
        System.out.println("hello BeanPostProcessor!!!");
    }
}
```

测试方法如下：

```
ClassPathResource resource = new ClassPathResource("spring.xml");
DefaultListableBeanFactory factory = new DefaultListableBeanFactory();
XmlBeanDefinitionReader reader = new XmlBeanDefinitionReader(factory);
reader.loadBeanDefinitions(resource);

BeanPostProcessorTest test = (BeanPostProcessorTest) factory.getBean("beanPostProcessorTest");
test.display();
```

运行结果：

![201808221005](http://static.iocoder.cn/162fab73d857b39db8e6198d5182c5d1)

运行结果比较奇怪，为什么没有执行 `#postProcessBeforeInitialization(...)` 和 `#postProcessAfterInitialization(...)` 方法呢？

我们 debug 跟踪下代码，这两个方法在 AbstractAutowireCapableBeanFactory 的 `#initializeBean(final String beanName, final Object bean, RootBeanDefinition mbd)` 方法处调用下，如下：

![201808221006](http://static.iocoder.cn/8d2ae41f84bfb1928845428dae1b26c1)

debug，在 `#postProcessBeforeInitialization(...)`方法中，结果如下：

![img](http://static.iocoder.cn/cc20356e408146eba311266ebc20de95)

这段代码是通过迭代 `#getBeanPostProcessors()` 方法返回的结果集来调用 BeanPostProcessor 的 `#postProcessBeforeInitialization(Object bean, String beanName)` 方法，但是在这里我们看到该方法返回的结果集为空，所以肯定不会执行相应的 `#postProcessBeforeInitialization(Object bean, String beanName)` 方法咯。怎么办？答案不言而喻：只需要 `#getBeanPostProcessors()` 方法，返回的结果集中存在至少一个元素即可，该方法定义如下：

```
// AbstractBeanFactory.java

/** BeanPostProcessors to apply in createBean. */
private final List<BeanPostProcessor> beanPostProcessors = new CopyOnWriteArrayList<>();

public List<BeanPostProcessor> getBeanPostProcessors() {
    return this.beanPostProcessors;
}
```

- 返回的 `beanPostProcessors` 是一个 `private` 的 List ，也就是说只要该类中存在 `beanPostProcessors.add(BeanPostProcessor beanPostProcessor)` 的调用，我们就找到了入口，在类 AbstractBeanFactory 中找到了如下代码：

  ```
  // AbstractBeanFactory.java
  
  @Override
  public void addBeanPostProcessor(BeanPostProcessor beanPostProcessor) {
  	Assert.notNull(beanPostProcessor, "BeanPostProcessor must not be null");
  	// Remove from old position, if any
  	this.beanPostProcessors.remove(beanPostProcessor);
  	// Track whether it is instantiation/destruction aware
  	if (beanPostProcessor instanceof InstantiationAwareBeanPostProcessor) {
  		this.hasInstantiationAwareBeanPostProcessors = true;
  	}
  	if (beanPostProcessor instanceof DestructionAwareBeanPostProcessor) {
  		this.hasDestructionAwareBeanPostProcessors = true;
  	}
  	// Add to end of list
  	this.beanPostProcessors.add(beanPostProcessor);
  }
  ```

  - 该方法是由 AbstractBeanFactory 的父类 `org.springframework.beans.factory.config.ConfigurableBeanFactory` 接口定义，它的核心意思就是将指定 BeanPostProcessor 注册到该 BeanFactory 创建的 bean 中，同时它是**按照插入的顺序进行注册的**，完全忽略 Ordered 接口所表达任何排序语义（在 BeanPostProcessor 中我们提供一个 Ordered 顺序，这个后面讲解）。

到这里应该就比较熟悉了，其实只需要显示调用 `#addBeanPostProcessor(BeanPostProcessor beanPostProcessor)` 方法就可以了。加入如下代码：

```
BeanPostProcessorTest beanPostProcessorTest = new BeanPostProcessorTest();
factory.addBeanPostProcessor(beanPostProcessorTest);
```

运行结果：

![img](http://static.iocoder.cn/162fab73d857b39db8e6198d5182c5d1)

其实还有一种更加简单的方法，这个我们后面再说，先看 BeanPostProcessor 的原理。

## 3. BeanPostProcessor 基本原理

`org.springframework.beans.factory.config.BeanPostProcessor` 接口，代码如下：

```
public interface BeanPostProcessor {

	@Nullable
	default Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException {
		return bean;
	}

	@Nullable
	default Object postProcessAfterInitialization(Object bean, String beanName) throws BeansException {
		return bean;
	}

}
```

BeanPostProcessor 可以理解为是 Spring 的一个工厂钩子（其实 Spring 提供一系列的钩子，如 Aware 、InitializingBean、DisposableBean），它是 Spring 提供的对象实例化阶段强有力的扩展点，允许 Spring 在实例化 bean 阶段对其进行定制化修改，比较常见的使用场景是处理标记接口实现类或者为当前对象提供代理实现（例如 AOP）。

一般普通的 BeanFactory 是不支持自动注册 BeanPostProcessor 的，需要我们手动调用 `#addBeanPostProcessor(BeanPostProcessor beanPostProcessor)` 方法进行注册。注册后的 BeanPostProcessor 适用于所有该 BeanFactory 创建的 bean，但是 **ApplicationContext 可以在其 bean 定义中自动检测所有的 BeanPostProcessor 并自动完成注册，同时将他们应用到随后创建的任何 Bean 中**。

`#postProcessBeforeInitialization(Object bean, String beanName)` 和 `#postProcessAfterInitialization(Object bean, String beanName)` 两个方法，都接收一个 Object 类型的 `bean` ，一个 String 类型的 `beanName` ，其中 `bean` 是已经实例化了的 `instanceBean` ，能拿到这个你是不是可以对它为所欲为了？ 这两个方法是初始化 `bean` 的前后置处理器，他们应用 `#invokeInitMethods(String beanName, final Object bean, RootBeanDefinition mbd)` 方法的前后。如下图：

![201808231001](http://static.iocoder.cn/7e25251ee94e4697b5acbaf2acb754d7)

代码层次上面已经贴出来，这里再贴一次：

![201808221006](http://static.iocoder.cn/8d2ae41f84bfb1928845428dae1b26c1)

两者源码如下：

```
// AbstractAutowireCapableBeanFactory.java

@Override
public Object applyBeanPostProcessorsBeforeInitialization(Object existingBean, String beanName)
		throws BeansException {
	Object result = existingBean;
	// 遍历 BeanPostProcessor 数组
	for (BeanPostProcessor processor : getBeanPostProcessors()) {
	    // 处理
		Object current = processor.postProcessBeforeInitialization(result, beanName);
        // 返回空，则返回 result
		if (current == null) {
			return result;
		}
		// 修改 result
		result = current;
	}
	return result;
}

@Override
public Object applyBeanPostProcessorsAfterInitialization(Object existingBean, String beanName)
		throws BeansException {
	Object result = existingBean;
	// 遍历 BeanPostProcessor
	for (BeanPostProcessor processor : getBeanPostProcessors()) {
	    // 处理
		Object current = processor.postProcessAfterInitialization(result, beanName);
		// 返回空，则返回 result
		if (current == null) {
			return result;
		}
		// 修改 result
		result = current;
	}
	return result;
}
```

### 3.1 自动检测并注册

`#getBeanPostProcessors()` 方法，返回的是 `beanPostProcessors` 集合，该集合里面存放就是我们自定义的 BeanPostProcessor ，如果该集合中存在元素则调用相应的方法，否则就直接返回 bean 了。这也是为什么使用 BeanFactory 容器是无法输出自定义 BeanPostProcessor 里面的内容，因为在 `BeanFactory#getBean(...)` 方法的过程中根本就没有将我们自定义的 BeanPostProcessor 注入进来，所以要想 BeanFactory 容器 的 BeanPostProcessor 生效我们必须手动调用 `#addBeanPostProcessor(BeanPostProcessor beanPostProcessor)` 方法，将定义的 BeanPostProcessor 注册到相应的 BeanFactory 中。**但是 ApplicationContext 不需要手动，因为 ApplicationContext 会自动检测并完成注册**。

ApplicationContext 实现自动注册的原因，在于我们构造一个 ApplicationContext 实例对象的时候会调用 `#registerBeanPostProcessors(ConfigurableListableBeanFactory beanFactory)` 方法，将检测到的 BeanPostProcessor 注入到 ApplicationContext 容器中，同时应用到该容器创建的 bean 中。代码如下：

```
// AbstractApplicationContext.java

/**
 * 实例化并调用已经注入的 BeanPostProcessor
 * 必须在应用中 bean 实例化之前调用
 */
protected void registerBeanPostProcessors(ConfigurableListableBeanFactory beanFactory) {
    PostProcessorRegistrationDelegate.registerBeanPostProcessors(beanFactory, this);
}

// PostProcessorRegistrationDelegate.java

public static void registerBeanPostProcessors(
		ConfigurableListableBeanFactory beanFactory, AbstractApplicationContext applicationContext) {

    // 获取所有的 BeanPostProcessor 的 beanName
    // 这些 beanName 都已经全部加载到容器中去，但是没有实例化
	String[] postProcessorNames = beanFactory.getBeanNamesForType(BeanPostProcessor.class, true, false);

	// Register BeanPostProcessorChecker that logs an info message when
	// a bean is created during BeanPostProcessor instantiation, i.e. when
	// a bean is not eligible for getting processed by all BeanPostProcessors.
    // 记录所有的beanProcessor数量
	int beanProcessorTargetCount = beanFactory.getBeanPostProcessorCount() + 1 + postProcessorNames.length;
	// 注册 BeanPostProcessorChecker，它主要是用于在 BeanPostProcessor 实例化期间记录日志
    // 当 Spring 中高配置的后置处理器还没有注册就已经开始了 bean 的实例化过程，这个时候便会打印 BeanPostProcessorChecker 中的内容
	beanFactory.addBeanPostProcessor(new BeanPostProcessorChecker(beanFactory, beanProcessorTargetCount));

	// Separate between BeanPostProcessors that implement PriorityOrdered,
	// Ordered, and the rest.
    // PriorityOrdered 保证顺序
	List<BeanPostProcessor> priorityOrderedPostProcessors = new ArrayList<>();
    // MergedBeanDefinitionPostProcessor
	List<BeanPostProcessor> internalPostProcessors = new ArrayList<>();
    // 使用 Ordered 保证顺序
	List<String> orderedPostProcessorNames = new ArrayList<>();
    // 没有顺序
	List<String> nonOrderedPostProcessorNames = new ArrayList<>();
	for (String ppName : postProcessorNames) {
        // PriorityOrdered
        if (beanFactory.isTypeMatch(ppName, PriorityOrdered.class)) {
            // 调用 getBean 获取 bean 实例对象
			BeanPostProcessor pp = beanFactory.getBean(ppName, BeanPostProcessor.class);
			priorityOrderedPostProcessors.add(pp);
			if (pp instanceof MergedBeanDefinitionPostProcessor) {
				internalPostProcessors.add(pp);
			}
		} else if (beanFactory.isTypeMatch(ppName, Ordered.class)) {
            // 有序 Ordered
			orderedPostProcessorNames.add(ppName);
		} else {
            // 无序
			nonOrderedPostProcessorNames.add(ppName);
		}
	}

	// First, register the BeanPostProcessors that implement PriorityOrdered.
    // 第一步，注册所有实现了 PriorityOrdered 的 BeanPostProcessor
    // 先排序
	sortPostProcessors(priorityOrderedPostProcessors, beanFactory);
    // 后注册
	registerBeanPostProcessors(beanFactory, priorityOrderedPostProcessors);

	// Next, register the BeanPostProcessors that implement Ordered.
    // 第二步，注册所有实现了 Ordered 的 BeanPostProcessor
	List<BeanPostProcessor> orderedPostProcessors = new ArrayList<>();
	for (String ppName : orderedPostProcessorNames) {
		BeanPostProcessor pp = beanFactory.getBean(ppName, BeanPostProcessor.class);
		orderedPostProcessors.add(pp);
		if (pp instanceof MergedBeanDefinitionPostProcessor) {
			internalPostProcessors.add(pp);
		}
	}
    // 先排序
	sortPostProcessors(orderedPostProcessors, beanFactory);
    // 后注册
	registerBeanPostProcessors(beanFactory, orderedPostProcessors);

	// Now, register all regular BeanPostProcessors.
    // 第三步注册所有无序的 BeanPostProcessor
	List<BeanPostProcessor> nonOrderedPostProcessors = new ArrayList<>();
	for (String ppName : nonOrderedPostProcessorNames) {
		BeanPostProcessor pp = beanFactory.getBean(ppName, BeanPostProcessor.class);
		nonOrderedPostProcessors.add(pp);
		if (pp instanceof MergedBeanDefinitionPostProcessor) {
			internalPostProcessors.add(pp);
		}
	}
	// 注册，无需排序
	registerBeanPostProcessors(beanFactory, nonOrderedPostProcessors);

	// Finally, re-register all internal BeanPostProcessors.
    // 最后，注册所有的 MergedBeanDefinitionPostProcessor 类型的 BeanPostProcessor
	sortPostProcessors(internalPostProcessors, beanFactory);
	registerBeanPostProcessors(beanFactory, internalPostProcessors);

	// Re-register post-processor for detecting inner beans as ApplicationListeners,
	// moving it to the end of the processor chain (for picking up proxies etc).
    // 加入ApplicationListenerDetector（探测器）
    // 重新注册 BeanPostProcessor 以检测内部 bean，因为 ApplicationListeners 将其移动到处理器链的末尾
	beanFactory.addBeanPostProcessor(new ApplicationListenerDetector(applicationContext));
}
```

- 方法首先 `beanFactory` 获取注册到该 BeanFactory 中所有 BeanPostProcessor 类型的 `beanName` 数组，其实就是找所有实现了 BeanPostProcessor 接口的 bean ，然后迭代这些 bean ，将其按照 PriorityOrdered、Ordered、无序的顺序，添加至相应的 List 集合中，最后依次调用 `#sortPostProcessors(List<?> postProcessors, ConfigurableListableBeanFactory beanFactory)` 方法来进行排序处理、 `#registerBeanPostProcessors(ConfigurableListableBeanFactory beanFactory, List<BeanPostProcessor> postProcessors)` 方法来完成注册。

- 【**排序**】很简单，如果 `beanFactory` 为 DefaultListableBeanFactory ，则返回 BeanFactory 所依赖的比较器，否则反正默认的比较器(OrderComparator)，然后调用 `List#sort(Comparator<? super E> c)` 方法即可。代码如下：

  ```
  // PostProcessorRegistrationDelegate.java
  private static void sortPostProcessors(List<?> postProcessors, ConfigurableListableBeanFactory beanFactory) {
  	// 获得 Comparator 对象
      Comparator<Object> comparatorToUse = null;
  	if (beanFactory instanceof DefaultListableBeanFactory) { // 依赖的 Comparator 对象
  		comparatorToUse = ((DefaultListableBeanFactory) beanFactory).getDependencyComparator();
  	}
  	if (comparatorToUse == null) { // 默认 Comparator 对象
  		comparatorToUse = OrderComparator.INSTANCE;
  	}
  	// 排序
  	postProcessors.sort(comparatorToUse);
  }
  ```

- 而对于【**注册**】，同样是调用 `AbstractBeanFactory#addBeanPostProcessor(BeanPostProcessor beanPostProcessor)` 方法完成注册。代码如下：

  ```
  // PostProcessorRegistrationDelegate.java
  
  private static void registerBeanPostProcessors(ConfigurableListableBeanFactory beanFactory, List<BeanPostProcessor> postProcessors) {
      // 遍历 BeanPostProcessor 数组，注册
  	for (BeanPostProcessor postProcessor : postProcessors) {
  		beanFactory.addBeanPostProcessor(postProcessor);
  	}
  }
  ```

## 4. 小结

至此，BeanPostProcessor 已经分析完毕了，这里简单总结下：

1. BeanPostProcessor 的作用域是容器级别的，它只和所在的容器相关 ，当 BeanPostProcessor 完成注册后，它会应用于所有跟它在同一个容器内的 bean 。
2. BeanFactory 和 ApplicationContext 对 BeanPostProcessor 的处理不同，ApplicationContext 会自动检测所有实现了 BeanPostProcessor 接口的 bean，并完成注册，但是使用 BeanFactory 容器时则需要手动调用 `AbstractBeanFactory#addBeanPostProcessor(BeanPostProcessor beanPostProcessor)` 方法来完成注册
3. ApplicationContext 的 BeanPostProcessor 支持 Ordered，而 BeanFactory 的 BeanPostProcessor 是不支持的，原因在于ApplicationContext 会对 BeanPostProcessor 进行 Ordered 检测并完成排序，而 BeanFactory 中的 BeanPostProcessor 只跟注册的顺序有关。

# 3、InitializingBean 和 init-method

﻿Spring 在 bean 初始化时进行三个检测扩展，也就是说我们可以对 bean 进行三个不同的定制化处理，前面两篇博客 [《【死磕 Spring】—— IoC 之深入分析 Aware 接口》](http://svip.iocoder.cn/Spring/IoC-Aware-interface) 和 [《【死磕 Spring】—— IoC 之深入分析 ﻿BeanPostProcessor》](http://svip.iocoder.cn/Spring/IoC-BeanPostProcessor) 已经分析了 Aware 接口族和 BeanPostProcessor 接口，这篇分析 InitializingBean 接口和 `init-method` 方法。

## 1. InitializingBean

Spring 的 `org.springframework.beans.factory.InitializingBean` 接口，为 bean 提供了定义初始化方法的方式，它仅包含了一个方法：`#afterPropertiesSet()` 。代码如下：

```
public interface InitializingBean {

    /**
     * 该方法在 BeanFactory 设置完了所有属性之后被调用
     * 该方法允许 bean 实例设置了所有 bean 属性时执行初始化工作，如果该过程出现了错误则需要抛出异常
     *
     * Invoked by the containing {@code BeanFactory} after it has set all bean properties
     * and satisfied {@link BeanFactoryAware}, {@code ApplicationContextAware} etc.
     * <p>This method allows the bean instance to perform validation of its overall
     * configuration and final initialization when all bean properties have been set.
     * @throws Exception in the event of misconfiguration (such as failure to set an
     * essential property) or if initialization fails for any other reason
     */
    void afterPropertiesSet() throws Exception;

}
```

Spring 在完成实例化后，设置完所有属性，进行 “Aware 接口” 和 “BeanPostProcessor 前置处理”之后，会接着检测当前 bean 对象是否实现了 InitializingBean 接口。如果是，则会调用其 `#afterPropertiesSet()` 方法，进一步调整 bean 实例对象的状态。

### 1.1 示例

```
public class InitializingBeanTest implements InitializingBean {

    private String name;

    @Override
    public void afterPropertiesSet() throws Exception {
        System.out.println("InitializingBeanTest initializing...");
        this.name = "chenssy 2 号";
    }

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }
}
```

配置项如下：

```
<bean id="initializingBeanTest" class="org.springframework.core.test.InitializingBeanTest">
    <property name="name" value="chenssy 1 号"/>
</bean>
```

测试代码如下：

```
InitializingBeanTest test = (InitializingBeanTest) factory.getBean("initializingBeanTest");
System.out.println("name ：" + test.getName());
```

执行结果如下：

[![img](http://static.iocoder.cn/4b96bfb7f25ce1550985b7463d232121)](http://static.iocoder.cn/4b96bfb7f25ce1550985b7463d232121)

在这个示例中，改变了 InitializingBeanTest 示例的 `name` 属性，也就是说 在 `#afterPropertiesSet()` 方法中，我们是可以改变 bean 的属性的，这相当于 Spring 容器又给我们提供了一种可以改变 bean 实例对象的方法。

### 1.2 invokeInitMethods

上面提到 bean 初始化阶段（ `#initializeBean(final String beanName, final Object bean, RootBeanDefinition mbd)` 方法）， Spring 容器会主动检查当前 bean 是否已经实现了 InitializingBean 接口，如果实现了，则会掉用其 `#afterPropertiesSet()` 方法。这个主动检查、调用的动作是由 `#invokeInitMethods(String beanName, final Object bean, @Nullable RootBeanDefinition mbd)` 方法来完成的。代码如下：

```
// AbstractAutowireCapableBeanFactory.java

protected void invokeInitMethods(String beanName, final Object bean, @Nullable RootBeanDefinition mbd)
        throws Throwable {
    // 首先会检查是否是 InitializingBean ，如果是的话需要调用 afterPropertiesSet()
    boolean isInitializingBean = (bean instanceof InitializingBean);
    if (isInitializingBean && (mbd == null || !mbd.isExternallyManagedInitMethod("afterPropertiesSet"))) {
        if (logger.isTraceEnabled()) {
            logger.trace("Invoking afterPropertiesSet() on bean with name '" + beanName + "'");
        }
        if (System.getSecurityManager() != null) { // 安全模式
            try {
                AccessController.doPrivileged((PrivilegedExceptionAction<Object>) () -> {
                    // 属性初始化的处理
                    ((InitializingBean) bean).afterPropertiesSet();
                    return null;
                }, getAccessControlContext());
            } catch (PrivilegedActionException pae) {
                throw pae.getException();
            }
        } else {
            // 属性初始化的处理
            ((InitializingBean) bean).afterPropertiesSet();
        }
    }

    if (mbd != null && bean.getClass() != NullBean.class) {
        // 判断是否指定了 init-method()，
        // 如果指定了 init-method()，则再调用制定的init-method
        String initMethodName = mbd.getInitMethodName();
        if (StringUtils.hasLength(initMethodName) &&
                !(isInitializingBean && "afterPropertiesSet".equals(initMethodName)) &&
                !mbd.isExternallyManagedInitMethod(initMethodName)) {
            // 激活用户自定义的初始化方法
            // 利用反射机制执行
            invokeCustomInitMethod(beanName, bean, mbd);
        }
    }
}
```

- 首先，检测当前 bean 是否实现了 InitializingBean 接口，如果实现了则调用其 `#afterPropertiesSet()` 方法。
- 然后，再检查是否也指定了 `init-method`，如果指定了则通过反射机制调用指定的 `init-method` 方法。

虽然该接口为 Spring 容器的扩展性立下了汗马功劳，但是如果真的让我们的业务对象来实现这个接口就显得不是那么的友好了，Spring 的一个核心理念就是无侵入性，但是如果我们业务类实现这个接口就显得 Spring 容器具有侵入性了。所以 Spring 还提供了另外一种实现的方式：`init-method` 方法

## 2. init-method

在分析分析 `<bean>` 标签解析过程中，我们提到了有关于 `init-method` 属性 ([《【死磕 Spring】—— IoC 之解析 标签：BeanDefinition》](http://svip.iocoder.cn/Spring/IoC-parse-BeanDefinitions-for-BeanDefinition))，该属性用于在 bean 初始化时指定执行方法，可以用来替代实现 InitializingBean 接口。

### 2.1 示例

```
public class InitializingBeanTest {

    private String name;

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }

    public void setOtherName(){
        System.out.println("InitializingBeanTest setOtherName...");
        this.name = "chenssy 3 号";
    }
}
```

配置文件如下：

```
<bean id="initializingBeanTest" class="org.springframework.core.test.InitializingBeanTest"
        init-method="setOtherName">
    <property name="name" value="chenssy 1 号"/>
</bean>
```

执行结果:

[![img](http://static.iocoder.cn/b42014ec9266892912cfe7da7eddca73)](http://static.iocoder.cn/b42014ec9266892912cfe7da7eddca73)

完全可以达到和 InitializingBean 一样的效果，而且在代码中我们没有看到丝毫 Spring 侵入的现象。所以通过 `init-method` 我们可以使用业务对象中定义的任何方法来实现 bean 实例对象的初始化定制化，而不再受制于 InitializingBean的 `#afterPropertiesSet()` 方法。同时我们可以使用 `<beans>` 标签的 `default-init-method` 属性来统一指定初始化方法，这样就省了需要在每个 `<bean>` 标签中都设置 `init-method` 这样的繁琐工作了。比如在 `default-init-method` 规定所有初始化操作全部以 `initBean()` 命名。如下：

[![img](http://static.iocoder.cn/69314ca6b0b46ace994636da2ff473a9)](http://static.iocoder.cn/69314ca6b0b46ace994636da2ff473a9)

## 3. 小结

从 `#invokeInitMethods(...)` 方法中，我们知道 `init-method` 指定的方法会在 `#afterPropertiesSet()` 方法之后执行，如果 `#afterPropertiesSet()` 方法的执行的过程中出现了异常，则 `init-method` 是不会执行的，而且由于 `init-method` 采用的是反射执行的方式，所以 `#afterPropertiesSet()` 方法的执行效率一般会高些，但是并不能排除我们要优先使用 `init-method`，主要是因为它消除了 bean 对 Spring 的依赖，Spring 没有侵入到我们业务代码，这样会更加符合 Spring 的理念。诚然，`init-method` 是基于 xml 配置文件的，就目前而言，我们的工程几乎都摒弃了配置，而采用注释的方式，那么 `@PreDestory` 可能适合你，当然这个注解我们后面分析。

至此，InitializingBean 和 init-method 已经分析完毕了，对于DisposableBean 和 `destroy-method` ，他们和 init 相似，这里就不做阐述了。

# 4、Bean 的生命周期

﻿在分析 Spring Bean 实例化过程中提到 Spring 并不是一启动容器就开启 bean 的实例化进程，只有当客户端通过显示或者隐式的方式调用 BeanFactory 的 `#getBean(...)` 方法来请求某个实例对象的时候，它才会触发相应 bean 的实例化进程。当然，也可以选择直接使用 ApplicationContext 容器，因为该容器启动的时候会立刻调用注册到该容器所有 bean 定义的实例化方法。当然，对于 BeanFactory 容器而言，并不是所有的 `#getBean(...)` 方法都会触发实例化进程，比如 singleton 类型的 bean，该类型的 bean 只会在第一次调用 `getBean()` 的时候才会触发，而后续的调用则会直接返回容器缓存中的实例对象。

`#getBean(...)` 方法，只是 bean 实例化进程的入口，真正的实现逻辑其实是在 AbstractAutowireCapableBeanFactory 的 `#doCreateBean(...)` 中实现，实例化过程如下图：

[![img](http://static.iocoder.cn/2a90a57e3bb96cc6ffa2619babe72bc4)](http://static.iocoder.cn/2a90a57e3bb96cc6ffa2619babe72bc4)

原来我们采用 new 的方式创建一个对象，用完该对象在其脱离作用域后就会被回收，对于后续操作我们无权也没法干涉，但是采用 Spring 容器后，我们完全摆脱了这种命运，Spring 容器将会对其所有管理的 Bean 对象全部给予一个**统一的生命周期管理**，同时在这个阶段我们也可以对其进行干涉（比如对 bean 进行增强处理，对 bean 进行篡改），如上图。

## 1. bean 实例化

在 `#doCreateBean(...)` 方法中，首先进行 bean 实例化工作，主要由 `#createBeanInstance(...)` 方法实现，该方法返回一个 BeanWrapper 对象。BeanWrapper 对象是 Spring 的一个低级 Bean 基础结构的核心接口，为什么说是**低级**呢？因为这个时候的 Bean 还不能够被我们使用，连最基本的属性都没有设置。而且在我们实际开发过程中，一般都不会直接使用该类，而是通过 BeanFactory 隐式使用。

BeanWrapper 接口有一个默认实现类 BeanWrapperImpl，其主要作用是对 Bean 进行“包裹”，然后对这个包裹的 bean 进行操作，比如后续注入 bean 属性。

在实例化 bean 过程中，Spring 采用“策略模式”来决定采用哪种方式来实例化 bean，一般有反射和 CGLIB 动态字节码两种方式。

InstantiationStrategy 定义了 Bean 实例化策略的抽象接口，其子类 SimpleInstantiationStrategy 提供了基于反射来实例化对象的功能，但是不支持方法注入方式的对象实例化。CglibSubclassingInstantiationStrategy 继承 SimpleInstantiationStrategy，他除了拥有父类以反射实例化对象的功能外，还提供了通过 CGLIB 的动态字节码的功能进而支持方法注入所需的对象实例化需求。默认情况下，Spring 采用 CglibSubclassingInstantiationStrategy。

关于 Bean 实例化的详细过程，请参考以下几篇文章：

1. [【死磕 Spring】—— IoC 之加载 bean：创建 bean（一）之主流程](http://svip.iocoder.cn/Spring/IoC-get-Bean-createBean-1)
2. [【死磕 Spring】—— IoC 之加载 bean：创建 bean（二）之实例化 Bean 对象(1)](http://svip.iocoder.cn/Spring/IoC-get-Bean-createBean-2)
3. [【死磕 Spring】—— IoC 之加载 bean：创建 bean（三）之实例化 Bean 对象(2)](http://svip.iocoder.cn/Spring/IoC-get-Bean-createBean-3)
4. [【死磕 Spring】—— IoC 之加载 bean：创建 bean（四）之属性填充](http://svip.iocoder.cn/Spring/IoC-get-Bean-createBean-4)
5. [【死磕 Spring】—— IoC 之加载 bean：创建 bean（五）之循环依赖处理](http://svip.iocoder.cn/Spring/IoC-get-Bean-createBean-5)
6. [【死磕 Spring】—— IoC 之加载 bean：创建 bean（六）之初始化 Bean 对象](http://svip.iocoder.cn/Spring/IoC-get-Bean-createBean-6)

对于 BeanWrapper 和 具体的实例化策略，LZ 在后面会专门写文章来进行详细说明。

## 2. 激活 Aware

当 Spring 完成 bean 对象实例化并且设置完相关属性和依赖后，则会开始 bean 的初始化进程（ `#initializeBean(...)` ），初始化第一个阶段是检查当前 bean 对象是否实现了一系列以 Aware 结尾的的接口。

Aware 接口为 Spring 容器的核心接口，是一个具有标识作用的超级接口，实现了该接口的 bean 是具有被 Spring 容器通知的能力，通知的方式是采用回调的方式。

在初始化阶段主要是感知 BeanNameAware、BeanClassLoaderAware、BeanFactoryAware 。代码如下：

```
// AbstractAutowireCapableBeanFactory.java

private void invokeAwareMethods(final String beanName, final Object bean) {
	if (bean instanceof Aware) {
	    // BeanNameAware
		if (bean instanceof BeanNameAware) {
			((BeanNameAware) bean).setBeanName(beanName);
		}
		// BeanClassLoaderAware
		if (bean instanceof BeanClassLoaderAware) {
			ClassLoader bcl = getBeanClassLoader();
			if (bcl != null) {
				((BeanClassLoaderAware) bean).setBeanClassLoader(bcl);
			}
		}
		// BeanFactoryAware
		if (bean instanceof BeanFactoryAware) {
			((BeanFactoryAware) bean).setBeanFactory(AbstractAutowireCapableBeanFactory.this);
		}
	}
}
```

- BeanNameAware：对该 bean 对象定义的 beanName 设置到当前对象实例中
- BeanClassLoaderAware：将当前 bean 对象相应的 ClassLoader 注入到当前对象实例中
- BeanFactoryAware：BeanFactory 容器会将自身注入到当前对象实例中，这样当前对象就会拥有一个 BeanFactory 容器的引用。

当然，Spring 不仅仅只是提供了上面三个 Aware 接口，而是一系列：

- LoadTimeWeaverAware：加载Spring Bean时织入第三方模块，如AspectJ
- BootstrapContextAware：资源适配器BootstrapContext，如JCA,CCI
- ResourceLoaderAware：底层访问资源的加载器
- PortletConfigAware：PortletConfig
- PortletContextAware：PortletContext
- ServletConfigAware：ServletConfig
- ServletContextAware：ServletContext
- MessageSourceAware：国际化
- ApplicationEventPublisherAware：应用事件
- NotificationPublisherAware：JMX通知

更多关于 Aware 的请关注：[《【死磕 Spring】—— IoC 之深入分析 Aware 接口》](http://svip.iocoder.cn/Spring/IoC-Aware-interface) 。

## 3. BeanPostProcessor

初始化第二个阶段则是 BeanPostProcessor 增强处理，在该阶段 BeanPostProcessor 会处理当前容器内所有符合条件的实例化后的 bean 对象。它主要是对 Spring 容器提供的 bean 实例对象进行有效的扩展，允许 Spring 在初始化 bean 阶段对其进行定制化修改，如处理标记接口或者为其提供代理实现。

BeanPostProcessor 接口提供了两个方法，在不同的时机执行，分别对应上图的前置处理和后置处理。代码如下：

```
public interface BeanPostProcessor {

	@Nullable
	default Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException {
		return bean;
	}

	@Nullable
	default Object postProcessAfterInitialization(Object bean, String beanName) throws BeansException {
		return bean;
	}

}
```

更多关于 BeanPostProcessor 的请关注：[《【死磕 Spring】—— IoC 之深入分析 ﻿BeanPostProcessor》](http://svip.iocoder.cn/Spring/IoC-BeanPostProcessor) 。

## 4. InitializingBean 和 init-method

InitializingBean 是一个接口，它为 Spring Bean 的初始化提供了一种方式，它有一个 `#afterPropertiesSet()` 方法，在 bean 的初始化进程中会判断当前 bean 是否实现了 InitializingBean，如果实现了则调用 `#afterPropertiesSet()` 方法，进行初始化工作。然后再检查是否也指定了 `init-method` ，如果指定了则通过反射机制调用指定的 `init-method` 方法。代码如下：

```
// AbstractAutowireCapableBeanFactory.java

protected void invokeInitMethods(String beanName, final Object bean, @Nullable RootBeanDefinition mbd)
		throws Throwable {
    // 首先会检查是否是 InitializingBean ，如果是的话需要调用 afterPropertiesSet()
	boolean isInitializingBean = (bean instanceof InitializingBean);
	if (isInitializingBean && (mbd == null || !mbd.isExternallyManagedInitMethod("afterPropertiesSet"))) {
		if (logger.isTraceEnabled()) {
			logger.trace("Invoking afterPropertiesSet() on bean with name '" + beanName + "'");
		}
		if (System.getSecurityManager() != null) { // 安全模式
			try {
				AccessController.doPrivileged((PrivilegedExceptionAction<Object>) () -> {
                    // 属性初始化的处理
					((InitializingBean) bean).afterPropertiesSet();
					return null;
				}, getAccessControlContext());
			} catch (PrivilegedActionException pae) {
				throw pae.getException();
			}
		} else {
            // 属性初始化的处理
			((InitializingBean) bean).afterPropertiesSet();
		}
	}

	if (mbd != null && bean.getClass() != NullBean.class) {
        // 判断是否指定了 init-method()，
        // 如果指定了 init-method()，则再调用制定的init-method
		String initMethodName = mbd.getInitMethodName();
		if (StringUtils.hasLength(initMethodName) &&
				!(isInitializingBean && "afterPropertiesSet".equals(initMethodName)) &&
				!mbd.isExternallyManagedInitMethod(initMethodName)) {
            // 激活用户自定义的初始化方法
            // 利用反射机制执行
			invokeCustomInitMethod(beanName, bean, mbd);
		}
	}
}
```

对于 Spring 而言，虽然上面两种方式都可以实现初始化定制化，但是更加推崇 `init-method` 方式，因为对于 InitializingBean 接口而言，他需要 bean 去实现接口，这样就会污染我们的应用程序，显得 Spring 具有一定的侵入性。但是由于 `init-method` 是采用反射的方式，所以执行效率上相对于 InitializingBean 接口回调的方式可能会低一些。

更多关于 init 的请关注：[《【死磕 Spring】—— IoC 之 深入分析 InitializingBean 和 init-method》](http://svip.iocoder.cn/Spring/IoC-InitializingBean-and-init-method)

## 5. DisposableBean 和 destroy-method

与 InitializingBean 和 `init-method` 用于对象的自定义初始化工作相似，DisposableBean和 `destroy-method` 则用于对象的自定义销毁工作。

当一个 bean 对象经历了实例化、设置属性、初始化阶段，那么该 bean 对象就可以供容器使用了（调用的过程）。当完成调用后，如果是 singleton 类型的 bean ，则会看当前 bean 是否应实现了 DisposableBean 接口或者配置了 `destroy-method` 属性，如果是的话，则会为该实例注册一个用于对象销毁的回调方法，便于在这些 singleton 类型的 bean 对象销毁之前执行销毁逻辑。

但是，并不是对象完成调用后就会立刻执行销毁方法，因为这个时候 Spring 容器还处于运行阶段，只有当 Spring 容器关闭的时候才会去调用。但是， Spring 容器不会这么聪明会自动去调用这些销毁方法，而是需要我们主动去告知 Spring 容器。

- 对于 BeanFactory 容器而言，我们需要主动调用 `#destroySingletons()` 方法，通知 BeanFactory 容器去执行相应的销毁方法。
- 对于 ApplicationContext 容器而言，调用 `#registerShutdownHook()` 方法。

## 6. 实践验证

下面用一个实例来真实看看看上面执行的逻辑，毕竟理论是不能缺少实践的：

```
public class LifeCycleBean implements BeanNameAware,BeanFactoryAware,BeanClassLoaderAware,BeanPostProcessor,
        InitializingBean,DisposableBean {

    private String test;

    public String getTest() {
        return test;
    }

    public void setTest(String test) {
        System.out.println("属性注入....");
        this.test = test;
    }

    public LifeCycleBean(){ // 构造方法
        System.out.println("构造函数调用...");
    }

    public void display(){
        System.out.println("方法调用...");
    }

    @Override
    public void setBeanFactory(BeanFactory beanFactory) throws BeansException {
        System.out.println("BeanFactoryAware 被调用...");
    }

    @Override
    public void setBeanName(String name) {
        System.out.println("BeanNameAware 被调用...");
    }

    @Override
    public void setBeanClassLoader(ClassLoader classLoader) {
        System.out.println("BeanClassLoaderAware 被调用...");
    }

    @Override
    public Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException {
        System.out.println("BeanPostProcessor postProcessBeforeInitialization 被调用...");
        return bean;
    }

    @Override
    public Object postProcessAfterInitialization(Object bean, String beanName) throws BeansException {
        System.out.println("BeanPostProcessor postProcessAfterInitialization 被调用...");
        return bean;
    }

    @Override
    public void destroy() throws Exception {
        System.out.println("DisposableBean destroy 被调动...");
    }

    @Override
    public void afterPropertiesSet() throws Exception {
        System.out.println("InitializingBean afterPropertiesSet 被调动...");
    }

    public void initMethod(){
        System.out.println("init-method 被调用...");
    }

    public void destroyMethdo(){
        System.out.println("destroy-method 被调用...");
    }

}
```

- LifeCycleBean 继承了 `BeanNameAware` , `BeanFactoryAware` , `BeanClassLoaderAware` , `BeanPostProcessor` , `InitializingBean` , `DisposableBean` 六个接口，同时定义了一个 `test` 属性用于验证属性注入和提供一个 `#display()` 方法用于模拟调用。

配置如下：

```
<bean id="lifeCycle" class="org.springframework.core.test.lifeCycleBean"
        init-method="initMethod" destroy-method="destroyMethdo">
    <property name="test" value="test"/>
</bean>
```

- 配置 `init-method` 和 `destroy-method`。

测试方法如下：

```
// BeanFactory 容器一定要调用该方法进行 BeanPostProcessor 注册
factory.addBeanPostProcessor(new LifeCycleBean()); // <1>

LifeCycleBean lifeCycleBean = (LifeCycleBean) factory.getBean("lifeCycle");
lifeCycleBean.display();

System.out.println("方法调用完成，容器开始关闭....");
// 关闭容器
factory.destroySingletons();
```

运行结果：

```
构造函数调用...
构造函数调用...
属性注入....
BeanNameAware 被调用...
BeanClassLoaderAware 被调用...
BeanFactoryAware 被调用...
BeanPostProcessor postProcessBeforeInitialization 被调用...
InitializingBean afterPropertiesSet 被调动...
init-method 被调用...
BeanPostProcessor postProcessAfterInitialization 被调用...
方法调用...
方法调用完成，容器开始关闭....
DisposableBean destroy 被调动...
destroy-method 被调用...
```

- 有两个构造函数调用，是因为要 `<1>` 处注入一个 BeanPostProcessor（你也可以另外提供一个 BeanPostProcessor 实例）。

根据执行的结果已经上面的分析，我们就可以对 Spring Bean 的声明周期过程如下（方法级别）：

1. Spring 容器根据实例化策略对 Bean 进行实例化。
2. 实例化完成后，如果该 bean 设置了一些属性的话，则利用 set 方法设置一些属性。
3. 如果该 Bean 实现了 BeanNameAware 接口，则调用 `#setBeanName(String beanName)` 方法。
4. 如果该 bean 实现了 BeanClassLoaderAware 接口，则调用 `setBeanClassLoader(ClassLoader classLoader)` 方法。
5. 如果该 bean 实现了 BeanFactoryAware接口，则调用 `setBeanFactory(BeanFactory beanFactory)` 方法。
6. 如果该容器注册了 BeanPostProcessor，则会调用`#postProcessBeforeInitialization(Object bean, String beanName)` 方法,完成 bean 前置处理
7. 如果该 bean 实现了 InitializingBean 接口，则调用`#afterPropertiesSet()` 方法。
8. 如果该 bean 配置了 `init-method` 方法，则调用其指定的方法。
9. 初始化完成后，如果该容器注册了 BeanPostProcessor 则会调用 `#postProcessAfterInitialization(Object bean, String beanName)` 方法,完成 bean 的后置处理。
10. 对象完成初始化，开始方法调用。
11. 在容器进行关闭之前，如果该 bean 实现了 DisposableBean 接口，则调用 `#destroy()` 方法。
12. 在容器进行关闭之前，如果该 bean 配置了 `destroy-method` ，则调用其指定的方法。
13. 到这里一个 bean 也就完成了它的一生。

# 5、BeanFactoryPostProcessor

在博客 [《【死磕 Spring】—— IoC 之 深入分析 BeanPostProcessor》](http://svip.iocoder.cn/Spring/IoC-BeanPostProcessor) 中，深入介绍了 BeanPostProcessor 的实现机制。在这篇文章中提到 BeanPostProcessor 是 Spring 提供一种扩展机制，该机制允许我们在 Bean 实例化之后初始化之际对 Bean 进行增强处理（前、后置处理）。

同样在 Spring 容器启动阶段，Spring 也提供了一种容器扩展机制：**BeanFactoryPostProcessor**，该机制作用于容器启动阶段，允许我们在容器实例化 Bean 之前对注册到该容器的 BeanDefinition 做出修改。

## 1. BeanFactoryPostProcessor

BeanFactoryPostProcessor 的机制，就相当于给了我们在 Bean 实例化之前最后一次修改 BeanDefinition 的机会，我们可以利用这个机会对 BeanDefinition 来进行一些额外的操作，比如更改某些 bean 的一些属性，给某些 Bean 增加一些其他的信息等等操作。

`org.springframework.beans.factory.config.BeanFactoryPostProcessor` 接口，定义如下：

```
public interface BeanFactoryPostProcessor {

 /**
  * 1、Modify the application context's internal bean factory after its standard initialization.
  *
  * 2、All bean definitions will have been loaded, but no beans will have been instantiated yet. This allows for overriding or adding properties even to eager-initializing beans.
  *
  * @param beanFactory the bean factory used by the application context
  * @throws org.springframework.beans.BeansException in case of errors
  */
 void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) throws BeansException;

}
```

BeanFactoryPostProcessor 接口仅有一个 `#postProcessBeanFactory(...)` 方法，该方法接收一个 ConfigurableListableBeanFactory 类型的 `beanFactory` 参数。上面有两行注释：

- 1、表示了该方法的**作用**：在 standard initialization（实在是不知道这个怎么翻译：标准的初始化？） 之后（已经就是已经完成了 BeanDefinition 的加载）对 bean factory 容器进行修改。其中参数 `beanFactory` 应该就是已经完成了 standard initialization 的 BeanFactory 。
- 2、表示作用**时机**：所有的 BeanDefinition 已经完成了加载即加载至 BeanFactory 中，但是还没有完成初始化。

所以这里总结一句话，就是：`#postProcessBeanFactory(...)` 方法，工作于 BeanDefinition 加载完成之后，Bean 实例化之前，其主要作用是对加载 BeanDefinition 进行修改。有一点需要需要**注意**的是在 `#postProcessBeanFactory(...)` 方法中，千万不能进行 Bean 的实例化工作，因为这样会导致 Bean 过早实例化，会产生严重后果，**我们始终需要注意的是 BeanFactoryPostProcessor 是与 BeanDefinition 打交道的，如果想要与 Bean 打交道，请使用 BeanPostProcessor** 。

与 BeanPostProcessor 一样，BeanFactoryPostProcessor 同样支持**排序**，一个容器可以同时拥有多个 BeanFactoryPostProcessor ，这个时候如果我们比较在乎他们的顺序的话，可以实现 Ordered 接口。

如果要自定义 BeanFactoryPostProcessor ，直接实现该接口即可。

## 2. 示例

```
public class BeanFactoryPostProcessor_1 implements BeanFactoryPostProcessor,Ordered {

    @Override
    public void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) throws BeansException {
        System.out.println("调用 BeanFactoryPostProcessor_1 ...");

        System.out.println("容器中有 BeanDefinition 的个数：" + beanFactory.getBeanDefinitionCount());

        // 获取指定的 BeanDefinition
        BeanDefinition bd = beanFactory.getBeanDefinition("studentService");

        MutablePropertyValues pvs = bd.getPropertyValues();

        pvs.addPropertyValue("name","chenssy1");
        pvs.addPropertyValue("age",15);
    }

    @Override
    public int getOrder() {
        return 1;
    }
}

public class BeanFactoryPostProcessor_2 implements BeanFactoryPostProcessor , Ordered {

    @Override
    public void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) throws BeansException {
        System.out.println("调用 BeanFactoryPostProcessor_2 ...");

        // 获取指定的 BeanDefinition
        BeanDefinition bd = beanFactory.getBeanDefinition("studentService");

        MutablePropertyValues pvs = bd.getPropertyValues();

        pvs.addPropertyValue("age",18);
    }

    @Override
    public int getOrder() {
        return 2;
    }
}
```

- 提供了两个自定义的 BeanFactoryPostProcessor ，都继承 BeanFactoryPostProcessor 和 Ordered，其中 BeanFactoryPostProcessor_1 改变 `name` 和 `age` 的值，BeanFactoryPostProcessor_2 该变 `age` 的值。Ordered 分别为 1 和 2 。

XML 配置如下：

```
<bean id="studentService" class="org.springframework.core.service.StudentService">
    <property name="name" value="chenssy"/>
    <property name="age" value="10"/>
</bean>

<bean class="org.springframework.core.test.BeanFactoryPostProcessor_1"/>
<bean class="org.springframework.core.test.BeanFactoryPostProcessor_2"/>
```

- `studentService` 设置 `name` 和 `age` 分别为 `"chenss"` 和 10 。

运行代码：

```
ApplicationContext context = new ClassPathXmlApplicationContext("spring.xml");

StudentService studentService = (StudentService) context.getBean("studentService");
System.out.println("student name:" + studentService.getName() + "-- age:" + studentService.getAge());
```

运行结果：

```
调用 BeanFactoryPostProcessor_1 ...
容器中有 BeanDefinition 的个数：3
调用 BeanFactoryPostProcessor_2 ...
student name:chenssy1-- age:18
```

- 看到运行结果，其实对上面的运行流程就已经一清二楚了。这里就不过多阐述了。

## 3. 原理

在上面测试方法中，我们使用的是 ApplicationContext ，对于 ApplicationContext 来说，使用 BeanFactoryPostProcessor 非常方便，因为他会自动识别配置文件中的 BeanFactoryPostProcessor 并且完成注册和调用，我们只需要简单的配置声明即可。而对于 BeanFactory 容器来说则不行，他和 BeanPostProcessor 一样需要容器主动去进行注册调用，方法如下：

```
BeanFactoryPostProcessor_1 beanFactoryPostProcessor1 = new BeanFactoryPostProcessor_1();
beanFactoryPostProcessor1.postProcessBeanFactory(factory);
```

**至于 ApplicationContext 是如何自动识别和调用，这个我们后续在分析 ApplicationContext 时会做详细说明的，当然，如果有兴趣的同学可以提前看**。

诚然，一般情况下我们是不会主动去自定义 BeanFactoryPostProcessor ，其实 Spring 为我们提供了几个常用的 BeanFactoryPostProcessor，他们是PropertyPlaceholderConfigurer 和 PropertyOverrideConfigurer ，其中 PropertyPlaceholderConfigurer 允许我们在 XML 配置文件中使用占位符并将这些占位符所代表的资源单独配置到简单的 properties 文件中来加载，PropertyOverrideConfigurer 则允许我们使用占位符来明确表明bean 定义中的 property 与 properties 文件中的各配置项之间的对应关系，这两个类在我们大型项目中有非常重要的作用，后续两篇文章将对其进行详细说明分析。

# 6、PropertyPlaceholderConfigurer

在上文 [《【死磕 Spring】—— IoC 之深入分析 BeanFactoryPostProcessor》](http://svip.iocoder.cn/Spring/IoC-BeanFactoryPostProcessor) 中，介绍了 BeanFactoryPostProcessor，知道 BeanFactoryPostProcessor 作用域容器启动阶段，可以对解析好的 BeanDefinition 进行定制化处理，而其中 **PropertyPlaceholderConfigurer** 是其一个非常重要的应用，也是其**子类**，介绍如下：

> PropertyPlaceholderConfigurer 允许我们用 Properties 文件中的属性，来定义应用上下文（配置文件或者注解）。

什么意思，就是说我们在 XML 配置文件（或者其他方式，如注解方式）中使用**占位符**的方式来定义一些资源，并将这些占位符所代表的资源配置到 Properties 中，这样只需要对 Properties 文件进行修改即可，这个特性非常，在后面来介绍一种我们在项目中经常用到场景。

## 1. PropertyResourceConfigurer

[![201809161001](http://static.iocoder.cn/201809161001.png)](http://static.iocoder.cn/201809161001.png)201809161001

从 PropertyPlaceholderConfigurer 的结构图可以看出，它间接实现了 Aware 和 BeanFactoryPostProcessor 两大扩展接口，这里只需要关注 BeanFactoryPostProcessor 即可。我们知道 BeanFactoryPostProcessor 提供了 `#postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory)` 接口方法，在这个体系中该方法的是在 **PropertyResourceConfigurer** 中实现，该类为属性资源的配置类，它实现了 BeanFactoryPostProcessor 接口，代码如下：

```
// PropertyResourceConfigurer.java
// extends PropertiesLoaderSupport
// implements BeanFactoryPostProcessor, PriorityOrdered

@Override
public void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) throws BeansException {
    try {
        // <1> 返回合并的 Properties 实例
        Properties mergedProps = mergeProperties();

        // Convert the merged properties, if necessary.
        // <2> 转换合并属性
        convertProperties(mergedProps);

        // Let the subclass process the properties.
        // <3> 子类处理
        processProperties(beanFactory, mergedProps);
    } catch (IOException ex) {
        throw new BeanInitializationException("Could not load properties", ex);
    }
}
```

- `<1>` 处，调用 `#mergeProperties()` 方法，返回合并的 Properties 实例。Properties 实例维护这**一组** `key-value` ，其实就是 Properties 配置文件中的内容。

- `<2>` 处，调用 `#convertProperties(Properties props)` 方法，转换合并的值，其实就是将原始值替换为真正的值。

- `<3>` 处，调用 `#processProperties(ConfigurableListableBeanFactory beanFactory, Properties props)` 方法，前面两个步骤已经将配置文件中的值进行了处理，那么该方法就是真正的替换过程，该方法**由子类实现**。代码如下：

  ```
  // PropertyResourceConfigurer.java
  
  protected abstract void processProperties(ConfigurableListableBeanFactory beanFactory, Properties props)
  		throws BeansException;
  ```

## 2. PropertyPlaceholderConfigurer

在 PropertyPlaceholderConfigurer 中，重写 `#processProperties(ConfigurableListableBeanFactory beanFactory, Properties props)` 方法，代码如下：

```
// PropertyPlaceholderConfigurer.java

@Override
protected void processProperties(ConfigurableListableBeanFactory beanFactoryToProcess, Properties props)
        throws BeansException {
    // <1> 创建 StringValueResolver 对象
    StringValueResolver valueResolver = new PlaceholderResolvingStringValueResolver(props);
    // <2> 处理
    doProcessProperties(beanFactoryToProcess, valueResolver);
}
```

### 2.1 PlaceholderResolvingStringValueResolver

> 对应 `#processProperties(ConfigurableListableBeanFactory beanFactoryToProcess, Properties props)` 方法的 `<1>` 处。

首先，构造一个 PlaceholderResolvingStringValueResolver 类型的 StringValueResolver 实例。StringValueResolver 为一个解析 String 类型值的策略接口，该接口提供了 `#resolveStringValue(String strVal)` 方法，用于解析 String 值。PlaceholderResolvingStringValueResolver 为其一个解析策略，构造方法如下：

```
// PropertyPlaceholderConfigurer.java
    
private class PlaceholderResolvingStringValueResolver implements StringValueResolver {
    
	private final PropertyPlaceholderHelper helper;
    
	private final PlaceholderResolver resolver;
    
	public PlaceholderResolvingStringValueResolver(Properties props) {
		this.helper = new PropertyPlaceholderHelper(
				placeholderPrefix, placeholderSuffix, valueSeparator, ignoreUnresolvablePlaceholders);
		this.resolver = new PropertyPlaceholderConfigurerResolver(props);
	}
	
	// ... 省略 resolveStringValue 方法
}
```

- 在构造 String 值解析器 StringValueResolver 时，将已经解析的 Properties 实例对象封装在 PlaceholderResolver 实例 `resolver` 中。PlaceholderResolver 是一个用于解析字符串中包含占位符的替换值的策略接口，该接口有一个 `#resolvePlaceholder(String strVa)` 方法，用于返回占位符的替换值。
- 还有一个 PropertyPlaceholderHelper 工具 `helper` ，从名字上面看应该是进行替换的工具类。

### 2.2 doProcessProperties

> 对应 `#processProperties(ConfigurableListableBeanFactory beanFactoryToProcess, Properties props)` 方法的 `<2>` 处。

然后，得到 String 解析器的实例 `valueResolver` 后，则会调用 `#doProcessProperties(ConfigurableListableBeanFactory beanFactoryToProcess, StringValueResolver valueResolver)` 方法，来进行真值的替换操作。该方法在父类 PlaceholderConfigurerSupport 中实现，代码如下：

```
// PlaceholderConfigurerSupport.java
    
protected void doProcessProperties(ConfigurableListableBeanFactory beanFactoryToProcess,
        StringValueResolver valueResolver) {
    // <2.1> 创建 BeanDefinitionVisitor 对象
    BeanDefinitionVisitor visitor = new BeanDefinitionVisitor(valueResolver);
    
    String[] beanNames = beanFactoryToProcess.getBeanDefinitionNames();
    for (String curName : beanNames) {
        // 校验
        // Check that we're not parsing our own bean definition,
        // to avoid failing on unresolvable placeholders in properties file locations.
        if (!(curName.equals(this.beanName) // 1. 当前实例 PlaceholderConfigurerSupport 不在解析范围内
                && beanFactoryToProcess.equals(this.beanFactory))) { // 2. 同一个 Spring 容器
            BeanDefinition bd = beanFactoryToProcess.getBeanDefinition(curName);
            try {
                visitor.visitBeanDefinition(bd);
            } catch (Exception ex) {
                throw new BeanDefinitionStoreException(bd.getResourceDescription(), curName, ex.getMessage(), ex);
            }
        }
    }
    
    // New in Spring 2.5: resolve placeholders in alias target names and aliases as well.
    // <2.3> 别名的占位符
    beanFactoryToProcess.resolveAliases(valueResolver);
    
    // New in Spring 3.0: resolve placeholders in embedded values such as annotation attributes.
    // <2.4> 解析嵌入值的占位符，例如注释属性
    beanFactoryToProcess.addEmbeddedValueResolver(valueResolver);
}
```

- `<2.1`> 处，根据 String 值解析策略 `valueResolver` 得到 BeanDefinitionVisitor 实例。BeanDefinitionVisitor 是 BeanDefinition 的访问者，我们通过它可以实现对 BeanDefinition 内容的进行访问，内容很多，例如 Scope、PropertyValues、FactoryMethodName 等等。
- `<2.2`> 处，得到该容器的所有 BeanName，然后对其进行访问（ `#visitBeanDefinition(BeanDefinition beanDefinition)` 方法）。
- `<2.3`> 处，解析别名的占位符。
- `<2.4`> 处，解析嵌入值的占位符，例如注释属性。

##### 2.2.1 visitBeanDefinition

这个方法的**核心**在于 `#visitBeanDefinition(BeanDefinition beanDefinition)` 方法的调用，代码如下：

```
// BeanDefinitionVisitor.java

public void visitBeanDefinition(BeanDefinition beanDefinition) {
	visitParentName(beanDefinition);
	visitBeanClassName(beanDefinition);
	visitFactoryBeanName(beanDefinition);
	visitFactoryMethodName(beanDefinition);
	visitScope(beanDefinition);
	if (beanDefinition.hasPropertyValues()) {
		visitPropertyValues(beanDefinition.getPropertyValues());
	}
	if (beanDefinition.hasConstructorArgumentValues()) {
		ConstructorArgumentValues cas = beanDefinition.getConstructorArgumentValues();
		visitIndexedArgumentValues(cas.getIndexedArgumentValues());
		visitGenericArgumentValues(cas.getGenericArgumentValues());
	}
}
```

- 我们可以看到该方法基本访问了 BeanDefinition 中所有值得访问的东西了，包括 parent 、class 、factory-bean 、factory-method 、scope 、property 、constructor-arg 。

##### 2.2.2 visitPropertyValues

本篇文章的主题是 **property** ，所以关注 `#visitPropertyValues(MutablePropertyValues pvs)` 方法即可。代码如下：

```
// BeanDefinitionVisitor.java

protected void visitPropertyValues(MutablePropertyValues pvs) {
    PropertyValue[] pvArray = pvs.getPropertyValues();
    // 遍历 PropertyValue 数组
    for (PropertyValue pv : pvArray) {
        // 解析真值
        Object newVal = resolveValue(pv.getValue());
        if (!ObjectUtils.nullSafeEquals(newVal, pv.getValue())) {
            // 设置到 PropertyValue 中
            pvs.add(pv.getName(), newVal);
        }
    }
}
```

- 过程就是对属性数组进行遍历，调用 `#resolveValue(Object value)`方法，对属性进行解析获取最新值，如果新值和旧值不等，则用新值替换旧值。

####### 2.2.2.1 resolveValue

`#resolveValue(Object value)` 方法，代码如下：

```
// BeanDefinitionVisitor.java

@Nullable
protected Object resolveValue(@Nullable Object value) {
	if (value instanceof BeanDefinition) {
		visitBeanDefinition((BeanDefinition) value);
	} else if (value instanceof BeanDefinitionHolder) {
		visitBeanDefinition(((BeanDefinitionHolder) value).getBeanDefinition());
	} else if (value instanceof RuntimeBeanReference) {
		RuntimeBeanReference ref = (RuntimeBeanReference) value;
		String newBeanName = resolveStringValue(ref.getBeanName());
		if (newBeanName == null) {
			return null;
		}
		if (!newBeanName.equals(ref.getBeanName())) {
			return new RuntimeBeanReference(newBeanName);
		}
	} else if (value instanceof RuntimeBeanNameReference) {
		RuntimeBeanNameReference ref = (RuntimeBeanNameReference) value;
		String newBeanName = resolveStringValue(ref.getBeanName());
		if (newBeanName == null) {
			return null;
		}
		if (!newBeanName.equals(ref.getBeanName())) {
			return new RuntimeBeanNameReference(newBeanName);
		}
	} else if (value instanceof Object[]) {
		visitArray((Object[]) value);
	} else if (value instanceof List) {
		visitList((List) value);
	} else if (value instanceof Set) {
		visitSet((Set) value);
	} else if (value instanceof Map) {
		visitMap((Map) value);
	} else if (value instanceof TypedStringValue) {
		TypedStringValue typedStringValue = (TypedStringValue) value;
		String stringValue = typedStringValue.getValue();
		if (stringValue != null) {
			String visitedString = resolveStringValue(stringValue);
			typedStringValue.setValue(visitedString);
		}
	// 由于 Properties 中的是 String，所以重点在此处
	} else if (value instanceof String) {
		return resolveStringValue((String) value);
	}
	return value;
}
```

- 由于配置的是 String 类型，所以只需要看 String 相关的。

####### 2.2.2.2 resolveStringValue

`#resolveStringValue(String strVal)` 方法，代码如下：

```
// BeanDefinitionVisitor.java

@Nullable
protected String resolveStringValue(String strVal) {
	if (this.valueResolver == null) {
		throw new IllegalStateException("No StringValueResolver specified - pass a resolver " +
				"object into the constructor or override the 'resolveStringValue' method");
	}
	// 解析真值
	String resolvedValue = this.valueResolver.resolveStringValue(strVal);
	// Return original String if not modified.
	return (strVal.equals(resolvedValue) ? strVal : resolvedValue);
}
```

- `valueResolver` 是我们在构造 BeanDefinitionVisitor 实例时传入的 String 类型解析器 PlaceholderResolvingStringValueResolver，调用其 `#resolveStringValue(String strVal)` 方法，代码如下：

```
// PropertyPlaceholderConfigurer.java
// 内部类 PlaceholderResolvingStringValueResolver.java

@Override
@Nullable
public String resolveStringValue(String strVal) throws BeansException {
    // 解析真值
	String resolved = this.helper.replacePlaceholders(strVal, this.resolver);
	// trim
	if (trimValues) {
		resolved = resolved.trim();
	}
	// 返回真值
	return (resolved.equals(nullValue) ? null : resolved);
}
```

- `helper` 为 PropertyPlaceholderHelper 实例对象，而 PropertyPlaceholderHelper 则是处理应用程序中包含占位符的字符串工具类。在构造 `helper` 实例对象时需要传入了几个参数：`placeholderPrefix`、`placeholderSuffix`、`valueSeparator`，这些值在 PlaceholderConfigurerSupport 中定义如下：

  ```
  // PlaceholderConfigurerSupport.java
  	
  /** Default placeholder prefix: {@value}. */
  public static final String DEFAULT_PLACEHOLDER_PREFIX = "${";
  /** Default placeholder suffix: {@value}. */
  public static final String DEFAULT_PLACEHOLDER_SUFFIX = "}";
  /** Default value separator: {@value}. */
  public static final String DEFAULT_VALUE_SEPARATOR = ":";
  
  
  /** Defaults to {@value #DEFAULT_PLACEHOLDER_PREFIX}. */
  protected String placeholderPrefix = DEFAULT_PLACEHOLDER_PREFIX;
  /** Defaults to {@value #DEFAULT_PLACEHOLDER_SUFFIX}. */
  protected String placeholderSuffix = DEFAULT_PLACEHOLDER_SUFFIX;
  /** Defaults to {@value #DEFAULT_VALUE_SEPARATOR}. */
  @Nullable
  protected String valueSeparator = DEFAULT_VALUE_SEPARATOR;
  ```

####### 2.2.2.3 replacePlaceholders

调用 PropertyPlaceholderHelper 的 `#replacePlaceholders(String value, PlaceholderResolver placeholderResolver)` 方法，进行占位符替换，代码如下：

```
public String replacePlaceholders(String value, PlaceholderResolver placeholderResolver) {
	Assert.notNull(value, "'value' must not be null");
	return parseStringValue(value, placeholderResolver, new HashSet<>());
}
```

- 调用 `#parseStringValue(String value, PlaceholderResolver placeholderResolver, Set<String> visitedPlaceholders)` 方法，**这个方法是这篇博客最核心的地方**，`${}` 占位符的替换。代码如下：

  ```
  // PropertyPlaceholderHelper.java
  
  protected String parseStringValue(String value, PlaceholderResolver placeholderResolver, Set<String> visitedPlaceholders) {
      StringBuilder result = new StringBuilder(value);
      // 获取前缀 "${" 的索引位置
      int startIndex = value.indexOf(this.placeholderPrefix);
      while (startIndex != -1) {
          // 获取 后缀 "}" 的索引位置
          int endIndex = findPlaceholderEndIndex(result, startIndex);
          if (endIndex != -1) {
              // 截取 "${" 和 "}" 中间的内容，这也就是我们在配置文件中对应的值
              String placeholder = result.substring(startIndex + this.placeholderPrefix.length(), endIndex);
              String originalPlaceholder = placeholder;
              if (!visitedPlaceholders.add(originalPlaceholder)) {
                  throw new IllegalArgumentException(
                          "Circular placeholder reference '" + originalPlaceholder + "' in property definitions");
              }
              // Recursive invocation, parsing placeholders contained in the placeholder key.
              // 解析占位符键中包含的占位符，真正的值
              placeholder = parseStringValue(placeholder, placeholderResolver, visitedPlaceholders);
              // Now obtain the value for the fully resolved key...
              // 从 Properties 中获取 placeHolder 对应的值 propVal
              String propVal = placeholderResolver.resolvePlaceholder(placeholder);
              // 如果不存在
              if (propVal == null && this.valueSeparator != null) {
                  // 查询 : 的位置
                  int separatorIndex = placeholder.indexOf(this.valueSeparator);
                  // 如果存在 :
                  if (separatorIndex != -1) {
                      // 获取 : 前面部分 actualPlaceholder
                      String actualPlaceholder = placeholder.substring(0, separatorIndex);
                      // 获取 : 后面部分 defaultValue
                      String defaultValue = placeholder.substring(separatorIndex + this.valueSeparator.length());
                      // 从 Properties 中获取 actualPlaceholder 对应的值
                      propVal = placeholderResolver.resolvePlaceholder(actualPlaceholder);
                      // 如果不存在 则返回 defaultValue
                      if (propVal == null) {
                          propVal = defaultValue;
                      }
                  }
              }
              if (propVal != null) {
                  // Recursive invocation, parsing placeholders contained in the
                  // previously resolved placeholder value.
                  propVal = parseStringValue(propVal, placeholderResolver, visitedPlaceholders);
                  result.replace(startIndex, endIndex + this.placeholderSuffix.length(), propVal);
                  if (logger.isTraceEnabled()) {
                      logger.trace("Resolved placeholder '" + placeholder + "'");
                  }
                  startIndex = result.indexOf(this.placeholderPrefix, startIndex + propVal.length());
              } else if (this.ignoreUnresolvablePlaceholders) {
                  // Proceed with unprocessed value.
                  // 忽略值
                  startIndex = result.indexOf(this.placeholderPrefix, endIndex + this.placeholderSuffix.length());
              } else {
                  throw new IllegalArgumentException("Could not resolve placeholder '" +
                          placeholder + "'" + " in value \"" + value + "\"");
              }
              visitedPlaceholders.remove(originalPlaceholder);
          } else {
              startIndex = -1;
          }
      }
      // 返回propVal，就是替换之后的值
      return result.toString();
  }
  ```

  1. 获取占位符前缀 `"${"` 的索引位置 `startIndex` 。
  2. 如果前缀 `"${"` 存在，则从 `“{”` 后面开始获取占位符后缀 “}” 的索引位置 `endIndex` 。
  3. 如果前缀 `“${”` 和后缀 `"}"` 都存在，则截取中间部分 `placeholder` 。
  4. 从 Properties 中获取 `placeHolder` 对应的值 `propVal` 。
  5. 如果 `propVal` 为空，则判断占位符中是否存在 `":"`，如果存在则对占位符进行分割处理，全面部分为 `actualPlaceholder`，后面部分 `defaultValue`，尝试从 Properties 中获取 `actualPlaceholder` 对应的值 `propVal`，如果不存在，则将 `defaultValue` 的值赋值给 `propVal`
  6. 返回 `propVal`，也就是 Properties 中对应的值。

## 3.应用

以上PropertyPlaceholderConfigurer 内部实现原理，它**允许我们在 XML 配置文件中使用占位符并将这些占位符所代表的资源单独配置到简单的 properties 文件中来加载**。这个特性非常重要，因为它我们对 Bean 实例属性的配置变得非常容易控制了，主要使用场景有：

1. 动态加载配置文件，多环境切换
2. 属性加解密

### 1. 多环境切换

在我们项目开发过程中，都会存在多个环境，如 dev 、test 、prod 等等，各个环境的配置都会不一样，在传统的开发过程中我们都是在进行打包的时候进行人工干预，或者将配置文件放在系统外部，加载的时候指定加载目录，这种方式容易出错，那么有没有一种比较好的方式来解决这种情况呢？有，**利用 PropertyPlaceholderConfigurer 的特性来动态加载配置文件，实现多环境切换**。

首先我们定义四个 Properties 文件，如下：

[![img](http://static.iocoder.cn/15374242055683.jpg)](http://static.iocoder.cn/15374242055683.jpg)

配置内容如下：

- `application-dev.properties` 文件，如下：

  ```
  student.name=chenssy-dev
  ```

- `application-test.properties` 文件，如下：

  ```
  student.name=chenssy-test
  ```

- `application-prod.properties` 文件，如下：

  ```
  student.name=chenssy-prod
  ```

然后实现一个类，该类继承 PropertyPlaceholderConfigurer，实现 `#loadProperties(Properties props)` 方法，根据环境的不同加载不同的配置文件，代码如下：

```
public class CustomPropertyConfig extends PropertyPlaceholderConfigurer {

    private Resource[] locations;

    private PropertiesPersister propertiesPersister = new DefaultPropertiesPersister();

    @Override
    public void setLocations(Resource[] locations) {
        this.locations = locations;
    }

    @Override
    public void setLocalOverride(boolean localOverride) {
        this.localOverride = localOverride;
    }

    /**
     * 覆盖这个方法，根据启动参数，动态读取配置文件
     * @param props
     * @throws IOException
     */
    @Override
    protected void loadProperties(Properties props) throws IOException {
        if (locations != null) {
            // locations 里面就已经包含了那三个定义的文件
            for (Resource location : this.locations) {
                InputStream is = null;
                try {
                    String filename = location.getFilename();
                    String env = "application-" + System.getProperty("spring.profiles.active", "dev") + ".properties";

                    // 找到我们需要的文件，加载
                    if (filename.contains(env)) {
                        logger.info("Loading properties file from " + location);
                        is = location.getInputStream();
                        this.propertiesPersister.load(props, is);

                    }
                } catch (IOException ex) {
                    logger.info("读取配置文件失败.....");
                    throw ex;
                } finally {
                    if (is != null) {
                        is.close();
                    }
                }
            }
        }
    }
}
```

配置文件：

```
<bean id="PropertyPlaceholderConfigurer" class="org.springframework.core.custom.CustomPropertyConfig">
    <property name="locations">
        <list>
            <value>classpath:config/application-dev.properties</value>
            <value>classpath:config/application-test.properties</value>
            <value>classpath:config/application-prod.properties</value>
        </list>
    </property>
</bean>

<bean id="studentService" class="org.springframework.core.service.StudentService">
    <property name="name" value="${student.name}"/>
</bean>
```

在 idea 的 VM options 里面增加 `-Dspring.profiles.active=dev`，标志当前环境为 dev 环境。测试代码如下：

```
ApplicationContext context = new ClassPathXmlApplicationContext("spring.xml");

StudentService studentService = (StudentService) context.getBean("studentService");
System.out.println("student name:" + studentService.getName());
```

运行结果：

```
student name:chenssy-dev
```

当将 `-Dspring.profiles.active` 调整为 test，则打印结果则是 chenssy-test，这样就完全实现了根据不同的环境加载不同的配置。

如果各位用过 Spring Boot 的话，这个就完全是 Spring Boot 里面的 `spring.profiles.active` ，可参见 `org.springframework.core.envAbstractEnvironment` 类，对应博客为 [《Spring boot源码分析-profiles环境（4）》](https://blog.csdn.net/jamet/article/details/77508182) 。

# 7、PropertyOverrideConfigurer

BeanFactoryPostProcessor 作用与 BeanDefinition 完成加载之后与 Bean 实例化之前，是 Spring 提供的一种强大的扩展机制。它有两个重要的子类，一个是 PropertyPlaceholderConfigurer，另一个是 PropertyOverrideConfigurer ，其中 PropertyPlaceholderConfigurer 允许我们通过配置 Properties 的方式来取代 Bean 中定义的占位符，而 **PropertyOverrideConfigurer** 呢？正是我们这篇博客介绍的。

> PropertyOverrideConfigurer 允许我们对 Spring 容器中配置的任何我们想处理的 bean 定义的 property 信息进行覆盖替换。

这个定义听起来有点儿玄乎，通俗点说，就是我们可以通过 PropertyOverrideConfigurer 来覆盖任何 bean 中的任何属性，只要我们想。

## 1. 使用

PropertyOverrideConfigurer 的使用规则是 `beanName.propertyName=value`，这里需要注意的是 `beanName.propertyName` 则是该 bean 中存在的属性。

### 1.1 示例一

依然使用以前的例子，`Student.class`，我们只需要修改下配置文件，声明下 PropertyOverrideConfigurer 以及其加载的配置文件。如下：

```
<bean class="org.springframework.beans.factory.config.PropertyOverrideConfigurer">
    <property name="locations">
        <list>
            <value>classpath:application.properties</value>
        </list>
    </property>
</bean>

<bean id="student" class="org.springframework.core.service.StudentService">
    <property name="name" value="chenssy"/>
</bean>
```

- 指定 student 的 `name` 属性值为 `"chenssy"` 。

- 声明 PropertyOverrideConfigurer 加载的文件为 `application.properties`，内容如下：

  ```
  student.name = chenssy-PropertyOverrideConfigurer
  ```

  - 指定 beanName 为 `student` 的 bean 的 `name` 属性值为 `"chenssy-PropertyOverrideConfigurer"` 。

测试打印 `student` 中的 `name` 属性值，代码如下：

```
ApplicationContext context = new ClassPathXmlApplicationContext("spring.xml");

StudentService studentService = (StudentService) context.getBean("student");
System.out.println("student name:" + studentService.getName());
```

运行结果为：

[![img](http://static.iocoder.cn/15377119278769.jpg)](http://static.iocoder.cn/15377119278769.jpg)

从中可以看出 PropertyOverrideConfigurer 定义的文件取代了 bean 中默认的值。

### 1.2 示例二

下面我们看一个有趣的例子，如果我们一个 bean 中 PropertyPlaceholderConfigurer 和 PropertyOverrideConfigurer 都使用呢？那是显示谁定义的值呢？这里先简单分析下：如果PropertyOverrideConfigurer 先作用，那么 PropertyPlaceholderConfigurer 在匹配占位符的时候就找不到了，**如果 PropertyOverrideConfigurer 后作用，也会直接取代 PropertyPlaceholderConfigurer 定义的值，所以无论如何都会显示 PropertyOverrideConfigurer 定义的值**。是不是这样呢？看如下例子：

xml 配置文件调整如下：

```
<bean class="org.springframework.beans.factory.config.PropertyOverrideConfigurer">
    <property name="locations">
        <list>
            <value>classpath:application1.properties</value>
        </list>
    </property>
</bean>

<bean class="org.springframework.beans.factory.config.PropertyPlaceholderConfigurer">
    <property name="locations">
        <list>
            <value>classpath:application2.properties</value>
        </list>
    </property>
</bean>

<bean id="student" class="org.springframework.core.service.StudentService">
    <property name="name" value="${studentService.name}"/>
</bean>
```

- 指定 .PropertyOverrideConfigurer 加载文件为 `application1.properties` 。配置文件如下：

  ```
  student.name = chenssy-PropertyOverrideConfigurer
  ```

- PropertyPlaceholderConfigurer 加载文件为 `application2.properties` 。配置文件如下：

  ```
  studentService.name = chenssy-PropertyPlaceholderConfigurer
  ```

- `student` 的 `name` 属性使用占位符 `${studentService.name}`。

测试程序依然是打印 name 属性值，运行结果如下：

[![img](http://static.iocoder.cn/15377127284347.jpg)](http://static.iocoder.cn/15377127284347.jpg)

所以，上面的分析没有错。下面我们来分析 **PropertyOverrideConfigurer 实现原理**。
其实如果了解 PropertyPlaceholderConfigurer 的实现机制的话，那么 PropertyOverrideConfigurer 也不难猜测：加载指定 Properties，迭代其中的属性值，依据 `“.”` 来得到 `beanName`（`split(".")[0]`），从容器中获取指定的 BeanDefinition，然后得到 `name` 属性，进行替换即可。

## 2. 实现原理

UML 结构图如下：

[![spring-201809231001](http://static.iocoder.cn/spring-201809231001.png)](http://static.iocoder.cn/spring-201809231001.png)spring-201809231001

与 PropertyPlaceholderConfigurer 一样，也是继承 PropertyResourceConfigurer，我们知道 PropertyResourceConfigurer 对 BeanFactoryPostProcessor 的 `#postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory)` 方法提供了实现，在该实现中它会去读取指定配置文件中的内容，然后调用 `#processProperties(ConfigurableListableBeanFactory beanFactoryToProcess, Properties props)` 方法。该方法是一个抽象方法，具体的实现由子类来实现，所以这里我们只需要看 PropertyOverrideConfigurer 中 `#processProperties(ConfigurableListableBeanFactory beanFactoryToProcess, Properties props)` 方法的具体实现，代码如下：

```
// PropertyOverrideConfigurer.java

@Override
protected void processProperties(ConfigurableListableBeanFactory beanFactory, Properties props)
        throws BeansException {
    // 迭代配置文件中的内容
    for (Enumeration<?> names = props.propertyNames(); names.hasMoreElements();) {
        String key = (String) names.nextElement();
        try {
            processKey(beanFactory, key, props.getProperty(key));
        } catch (BeansException ex) {
            String msg = "Could not process key '" + key + "' in PropertyOverrideConfigurer";
            if (!this.ignoreInvalidKeys) {
                throw new BeanInitializationException(msg, ex);
            }
            if (logger.isDebugEnabled()) {
                logger.debug(msg, ex);
            }
        }
    }
}
```

- 迭代 `props` 数组，依次调用 `#processKey(ConfigurableListableBeanFactory factory, String key, String value)` 方法，代码如下:

  ```
  // PropertyOverrideConfigurer.java
  
  /**
   * The default bean name separator.
   */
  public static final String DEFAULT_BEAN_NAME_SEPARATOR = ".";
  /**
   * Bean 名字的分隔符
   */
  private String beanNameSeparator = DEFAULT_BEAN_NAME_SEPARATOR;
  /**
   * Contains names of beans that have overrides.
   */
  private final Set<String> beanNames = Collections.newSetFromMap(new ConcurrentHashMap<>(16));
  protected void processKey(ConfigurableListableBeanFactory factory, String key, String value)
          throws BeansException {
      // 判断是否存在 "."，即获取其索引位置
      int separatorIndex = key.indexOf(this.beanNameSeparator);
      if (separatorIndex == -1) {
          throw new BeanInitializationException("Invalid key '" + key +
                  "': expected 'beanName" + this.beanNameSeparator + "property'");
      }
      // 得到 beanName
      String beanName = key.substring(0, separatorIndex);
      // 得到属性值
      String beanProperty = key.substring(separatorIndex+1);
      this.beanNames.add(beanName);
      // 替换
      applyPropertyValue(factory, beanName, beanProperty, value);
      if (logger.isDebugEnabled()) {
          logger.debug("Property '" + key + "' set to value [" + value + "]");
      }
  }
  ```

  - 获取分割符 `“.”` 的索引位置，得到 `beanName` 以及相应的属性，然后调用 `#applyPropertyValue(ConfigurableListableBeanFactory factory, String beanName, String property, String value)` 方法，代码如下：

    ```
    // PropertyOverrideConfigurer.java
    
    protected void applyPropertyValue(ConfigurableListableBeanFactory factory, String beanName, String property, String value) {
        // 获得 BeanDefinition 对象
        BeanDefinition bd = factory.getBeanDefinition(beanName);
        BeanDefinition bdToUse = bd;
        while (bd != null) {
            bdToUse = bd;
            bd = bd.getOriginatingBeanDefinition();
        }
        // 设置 PropertyValue 到 BeanDefinition 中
        PropertyValue pv = new PropertyValue(property, value);
        pv.setOptional(this.ignoreInvalidKeys);
        bdToUse.getPropertyValues().addPropertyValue(pv);
    }
    ```

    - 从容器中获取 BeanDefinition ，然后根据属性 `property` 和其值 `value` 构造成一个 PropertyValue 对象，最后调用 `#addPropertyValue(PropertyValue pv )` 方法。PropertyValue 是用于保存一组bean属性的信息和值的对像。代码如下：

      ```
      // MutablePropertyValues.java
      
      public MutablePropertyValues addPropertyValue(PropertyValue pv) {
          for (int i = 0; i < this.propertyValueList.size(); i++) {
              PropertyValue currentPv = this.propertyValueList.get(i);
              // 匹配
              if (currentPv.getName().equals(pv.getName())) {
                  // 合并属性
                  pv = mergeIfRequired(pv, currentPv);
                  // 覆盖属性
                  setPropertyValueAt(pv, i);
                  return this;
              }
          }
          // 未匹配到，添加到 propertyValueList 中
          this.propertyValueList.add(pv);
          return this;
      }
      ```

      - 添加 PropertyValue 对象，替换或者合并相同的属性值。整个过程其实与上面猜测相差不是很大。

## 3. 小结

至此，PropertyOverrideConfigurer 到这里也就分析完毕了。最后看下 PropertyPlaceholderConfigurer 和 PropertyOverrideConfigurer 整体的结构图：

[![spring-201809231002](http://static.iocoder.cn/spring-201809231002.png)](http://static.iocoder.cn/spring-201809231002.png)spring-201809231002

# 8、PropertyOverrideConfigurer

BeanFactoryPostProcessor 作用与 BeanDefinition 完成加载之后与 Bean 实例化之前，是 Spring 提供的一种强大的扩展机制。它有两个重要的子类，一个是 PropertyPlaceholderConfigurer，另一个是 PropertyOverrideConfigurer ，其中 PropertyPlaceholderConfigurer 允许我们通过配置 Properties 的方式来取代 Bean 中定义的占位符，而 **PropertyOverrideConfigurer** 呢？正是我们这篇博客介绍的。

> PropertyOverrideConfigurer 允许我们对 Spring 容器中配置的任何我们想处理的 bean 定义的 property 信息进行覆盖替换。

这个定义听起来有点儿玄乎，通俗点说，就是我们可以通过 PropertyOverrideConfigurer 来覆盖任何 bean 中的任何属性，只要我们想。

## 1. 使用

PropertyOverrideConfigurer 的使用规则是 `beanName.propertyName=value`，这里需要注意的是 `beanName.propertyName` 则是该 bean 中存在的属性。

### 1.1 示例一

依然使用以前的例子，`Student.class`，我们只需要修改下配置文件，声明下 PropertyOverrideConfigurer 以及其加载的配置文件。如下：

```
<bean class="org.springframework.beans.factory.config.PropertyOverrideConfigurer">
    <property name="locations">
        <list>
            <value>classpath:application.properties</value>
        </list>
    </property>
</bean>

<bean id="student" class="org.springframework.core.service.StudentService">
    <property name="name" value="chenssy"/>
</bean>
```

- 指定 student 的 `name` 属性值为 `"chenssy"` 。

- 声明 PropertyOverrideConfigurer 加载的文件为 `application.properties`，内容如下：

  ```
  student.name = chenssy-PropertyOverrideConfigurer
  ```

  - 指定 beanName 为 `student` 的 bean 的 `name` 属性值为 `"chenssy-PropertyOverrideConfigurer"` 。

测试打印 `student` 中的 `name` 属性值，代码如下：

```
ApplicationContext context = new ClassPathXmlApplicationContext("spring.xml");

StudentService studentService = (StudentService) context.getBean("student");
System.out.println("student name:" + studentService.getName());
```

运行结果为：

![img](http://static.iocoder.cn/15377119278769.jpg)

从中可以看出 PropertyOverrideConfigurer 定义的文件取代了 bean 中默认的值。

### 1.2 示例二

下面我们看一个有趣的例子，如果我们一个 bean 中 PropertyPlaceholderConfigurer 和 PropertyOverrideConfigurer 都使用呢？那是显示谁定义的值呢？这里先简单分析下：如果PropertyOverrideConfigurer 先作用，那么 PropertyPlaceholderConfigurer 在匹配占位符的时候就找不到了，**如果 PropertyOverrideConfigurer 后作用，也会直接取代 PropertyPlaceholderConfigurer 定义的值，所以无论如何都会显示 PropertyOverrideConfigurer 定义的值**。是不是这样呢？看如下例子：

xml 配置文件调整如下：

```
<bean class="org.springframework.beans.factory.config.PropertyOverrideConfigurer">
    <property name="locations">
        <list>
            <value>classpath:application1.properties</value>
        </list>
    </property>
</bean>

<bean class="org.springframework.beans.factory.config.PropertyPlaceholderConfigurer">
    <property name="locations">
        <list>
            <value>classpath:application2.properties</value>
        </list>
    </property>
</bean>

<bean id="student" class="org.springframework.core.service.StudentService">
    <property name="name" value="${studentService.name}"/>
</bean>
```

- 指定 .PropertyOverrideConfigurer 加载文件为 `application1.properties` 。配置文件如下：

  ```
  student.name = chenssy-PropertyOverrideConfigurer
  ```

- PropertyPlaceholderConfigurer 加载文件为 `application2.properties` 。配置文件如下：

  ```
  studentService.name = chenssy-PropertyPlaceholderConfigurer
  ```

- `student` 的 `name` 属性使用占位符 `${studentService.name}`。

测试程序依然是打印 name 属性值，运行结果如下：

![img](http://static.iocoder.cn/15377127284347.jpg)

所以，上面的分析没有错。下面我们来分析 **PropertyOverrideConfigurer 实现原理**。
其实如果了解 PropertyPlaceholderConfigurer 的实现机制的话，那么 PropertyOverrideConfigurer 也不难猜测：加载指定 Properties，迭代其中的属性值，依据 `“.”` 来得到 `beanName`（`split(".")[0]`），从容器中获取指定的 BeanDefinition，然后得到 `name` 属性，进行替换即可。

## 2. 实现原理

UML 结构图如下：

![spring-201809231001](http://static.iocoder.cn/spring-201809231001.png)

与 PropertyPlaceholderConfigurer 一样，也是继承 PropertyResourceConfigurer，我们知道 PropertyResourceConfigurer 对 BeanFactoryPostProcessor 的 `#postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory)` 方法提供了实现，在该实现中它会去读取指定配置文件中的内容，然后调用 `#processProperties(ConfigurableListableBeanFactory beanFactoryToProcess, Properties props)` 方法。该方法是一个抽象方法，具体的实现由子类来实现，所以这里我们只需要看 PropertyOverrideConfigurer 中 `#processProperties(ConfigurableListableBeanFactory beanFactoryToProcess, Properties props)` 方法的具体实现，代码如下：

```
// PropertyOverrideConfigurer.java

@Override
protected void processProperties(ConfigurableListableBeanFactory beanFactory, Properties props)
        throws BeansException {
    // 迭代配置文件中的内容
    for (Enumeration<?> names = props.propertyNames(); names.hasMoreElements();) {
        String key = (String) names.nextElement();
        try {
            processKey(beanFactory, key, props.getProperty(key));
        } catch (BeansException ex) {
            String msg = "Could not process key '" + key + "' in PropertyOverrideConfigurer";
            if (!this.ignoreInvalidKeys) {
                throw new BeanInitializationException(msg, ex);
            }
            if (logger.isDebugEnabled()) {
                logger.debug(msg, ex);
            }
        }
    }
}
```

- 迭代 `props` 数组，依次调用 `#processKey(ConfigurableListableBeanFactory factory, String key, String value)` 方法，代码如下:

  ```
  // PropertyOverrideConfigurer.java
  
  /**
   * The default bean name separator.
   */
  public static final String DEFAULT_BEAN_NAME_SEPARATOR = ".";
  /**
   * Bean 名字的分隔符
   */
  private String beanNameSeparator = DEFAULT_BEAN_NAME_SEPARATOR;
  /**
   * Contains names of beans that have overrides.
   */
  private final Set<String> beanNames = Collections.newSetFromMap(new ConcurrentHashMap<>(16));
  protected void processKey(ConfigurableListableBeanFactory factory, String key, String value)
          throws BeansException {
      // 判断是否存在 "."，即获取其索引位置
      int separatorIndex = key.indexOf(this.beanNameSeparator);
      if (separatorIndex == -1) {
          throw new BeanInitializationException("Invalid key '" + key +
                  "': expected 'beanName" + this.beanNameSeparator + "property'");
      }
      // 得到 beanName
      String beanName = key.substring(0, separatorIndex);
      // 得到属性值
      String beanProperty = key.substring(separatorIndex+1);
      this.beanNames.add(beanName);
      // 替换
      applyPropertyValue(factory, beanName, beanProperty, value);
      if (logger.isDebugEnabled()) {
          logger.debug("Property '" + key + "' set to value [" + value + "]");
      }
  }
  ```

  - 获取分割符 `“.”` 的索引位置，得到 `beanName` 以及相应的属性，然后调用 `#applyPropertyValue(ConfigurableListableBeanFactory factory, String beanName, String property, String value)` 方法，代码如下：

    ```
    // PropertyOverrideConfigurer.java
    
    protected void applyPropertyValue(ConfigurableListableBeanFactory factory, String beanName, String property, String value) {
        // 获得 BeanDefinition 对象
        BeanDefinition bd = factory.getBeanDefinition(beanName);
        BeanDefinition bdToUse = bd;
        while (bd != null) {
            bdToUse = bd;
            bd = bd.getOriginatingBeanDefinition();
        }
        // 设置 PropertyValue 到 BeanDefinition 中
        PropertyValue pv = new PropertyValue(property, value);
        pv.setOptional(this.ignoreInvalidKeys);
        bdToUse.getPropertyValues().addPropertyValue(pv);
    }
    ```

    - 从容器中获取 BeanDefinition ，然后根据属性 `property` 和其值 `value` 构造成一个 PropertyValue 对象，最后调用 `#addPropertyValue(PropertyValue pv )` 方法。PropertyValue 是用于保存一组bean属性的信息和值的对像。代码如下：

      ```
      // MutablePropertyValues.java
      
      public MutablePropertyValues addPropertyValue(PropertyValue pv) {
          for (int i = 0; i < this.propertyValueList.size(); i++) {
              PropertyValue currentPv = this.propertyValueList.get(i);
              // 匹配
              if (currentPv.getName().equals(pv.getName())) {
                  // 合并属性
                  pv = mergeIfRequired(pv, currentPv);
                  // 覆盖属性
                  setPropertyValueAt(pv, i);
                  return this;
              }
          }
          // 未匹配到，添加到 propertyValueList 中
          this.propertyValueList.add(pv);
          return this;
      }
      ```

      - 添加 PropertyValue 对象，替换或者合并相同的属性值。整个过程其实与上面猜测相差不是很大。

## 3. 小结

至此，PropertyOverrideConfigurer 到这里也就分析完毕了。最后看下 PropertyPlaceholderConfigurer 和 PropertyOverrideConfigurer 整体的结构图：

![spring-201809231002](http://static.iocoder.cn/spring-201809231002.png)

# 9、Bean 的类型转换体系

我们知道不管 Bean 对象里面的属性时什么类型，他们都是通过 XML 、Properties 或者其他方式来配置这些**属性**对象类型的。在 Spring 容器加载过程中，这些属性都是以 String 类型加载进容器的，但是最终都需要将这些 String 类型的属性转换 Bean 对象属性所对应真正的类型，要想完成这种由字符串到具体对象的转换，就需要这种转换规则相关的信息，而这些信息以及转换过程由 **Spring 类型转换体系**来完成。

------

我们依然以 xml 为例，在 Spring 容器加载阶段，容器将 xml 文件中定义的 `<bean>` 解析为 BeanDefinition，BeanDefinition 中存储着我们定义一个 bean 需要的所有信息，包括属性，这些属性是以 String 类型的存储的。当用户触发 Bean 实例化阶段时，Spring 容器会将这些属性转换为这些属性真正对应的类型。我们知道在 Bean 实例化阶段，属性的注入是在实例化 Bean 阶段的属性注入阶段，即 AbstractAutowireCapableBeanFactory 的 `#populateBean(String beanName, RootBeanDefinition mbd, BeanWrapper bw)` 方法。

在 `#populateBean(String beanName, RootBeanDefinition mbd, BeanWrapper bw)` 方法中，会将 BeanDefinition 中定义的属性值翻译为 PropertyValue ，然后调用 `#applyPropertyValues(String beanName, BeanDefinition mbd, BeanWrapper bw, PropertyValues pvs)` 方法，进行属性应用。其中 PropertyValue 用于保存单个 bean 属性的信息和值的对象。

------

在 `#applyPropertyValues(String beanName, BeanDefinition mbd, BeanWrapper bw, PropertyValues pvs)` 方法中，会调用 `#convertForProperty(Object value, String propertyName, BeanWrapper bw, TypeConverter converter)` 进行属性转换，代码如下：

```
// AbstractAutowireCapableBeanFactory。java

@Nullable
private Object convertForProperty(
        @Nullable Object value, String propertyName, BeanWrapper bw, TypeConverter converter) {
    // 若 TypeConverter 为 BeanWrapperImpl 类型，则使用 BeanWrapperImpl 来进行类型转换
    // 这里主要是因为 BeanWrapperImpl 实现了 PropertyEditorRegistry 接口
    if (converter instanceof BeanWrapperImpl) {
        return ((BeanWrapperImpl) converter).convertForProperty(value, propertyName);
    } else {
        // 获得属性对应的 PropertyDescriptor 对象
        PropertyDescriptor pd = bw.getPropertyDescriptor(propertyName);
        // 获得属性对应的 setting MethodParameter 对象
        MethodParameter methodParam = BeanUtils.getWriteMethodParameter(pd);
        // 执行转换
        return converter.convertIfNecessary(value, pd.getPropertyType(), methodParam);
    }
}
```

- 若 TypeConverter 为 BeanWrapperImpl 类型，则使用 BeanWrapperImpl 来进行类型转换，这里主要是因为 BeanWrapperImpl 实现了 PropertyEditorRegistry 接口。
- 否则，调用 TypeConverter 的 `#convertIfNecessary(Object value, Class<T> requiredType, MethodParameter methodParam)` 方法，进行类型转换。TypeConverter 是定义类型转换方法的接口，通常情况下与 PropertyEditorRegistry 配合使用实现类型转换。
- 关于 BeanWrapperImpl 小编后续专门出文分析它。

------

`#convertIfNecessary(Object value, Class<T> requiredType, MethodParameter methodParam)` 方法的实现者有两个：DataBinder 和 TypeConverterSupport 类。

- DataBinder 主要用于参数绑定（熟悉 Spring MVC 的都应该知道这个类）
- TypeConverterSupport 则是 TypeConverter 的基本实现，使用的是 `typeConverterDelegate` 委托者。

所以这里我们只需要关注 TypeConverterSupport 的 `#convertIfNecessary(Object value, Class<T> requiredType, MethodParameter methodParam)` 方法，代码如下：

```
// TypeConverterSupport.java

@Override
@Nullable
public <T> T convertIfNecessary(@Nullable Object value, @Nullable Class<T> requiredType, @Nullable Field field)
        throws TypeMismatchException {
    return doConvert(value, requiredType, null, field);
}

@Nullable
private <T> T doConvert(@Nullable Object value,@Nullable Class<T> requiredType,
        @Nullable MethodParameter methodParam, @Nullable Field field) throws TypeMismatchException {
    Assert.state(this.typeConverterDelegate != null, "No TypeConverterDelegate");
    try {
        if (field != null) { // field
            return this.typeConverterDelegate.convertIfNecessary(value, requiredType, field);
        } else { // methodParam
            return this.typeConverterDelegate.convertIfNecessary(value, requiredType, methodParam);
        }
    } catch (ConverterNotFoundException | IllegalStateException ex) {
        throw new ConversionNotSupportedException(value, requiredType, ex);
    } catch (ConversionException | IllegalArgumentException ex) {
        throw new TypeMismatchException(value, requiredType, ex);
    }
}
```

我们一直往下跟会跟踪到 TypeConverterDelegate 的 `#convertIfNecessary(Object newValue, @Nullable Class<T> requiredType, ...)` 方法，会发现如下代码段：

```
// TypeConverterDelegate.java

@Nullable
public <T> T convertIfNecessary(@Nullable String propertyName, @Nullable Object oldValue, @Nullable Object newValue,
        @Nullable Class<T> requiredType, @Nullable TypeDescriptor typeDescriptor) throws IllegalArgumentException {

        // ... 省略暂时非关键的代码

        // No custom editor but custom ConversionService specified?
        ConversionService conversionService = this.propertyEditorRegistry.getConversionService();
        if (editor == null && conversionService != null && newValue != null && typeDescriptor != null) {
            TypeDescriptor sourceTypeDesc = TypeDescriptor.forObject(newValue);
            if (conversionService.canConvert(sourceTypeDesc, typeDescriptor)) {
                try {
                    return (T) conversionService.convert(newValue, sourceTypeDesc, typeDescriptor);
                } catch (ConversionFailedException ex) {
                    // fallback to default conversion logic below
                    conversionAttemptEx = ex;
                }
            }
        }

        // ... 省略暂时非关键的代码
}
```

- 如果没有自定义的编辑器则使用 ConversionService 。

------

ConversionService 是字 Spring 3 后推出来用来替代 PropertyEditor 转换模式的转换体系，接口定义如下：

```
// ConversionService.java

public interface ConversionService {

    boolean canConvert(@Nullable Class<?> sourceType, Class<?> targetType);

    boolean canConvert(@Nullable TypeDescriptor sourceType, TypeDescriptor targetType);

    @Nullable
    <T> T convert(@Nullable Object source, Class<T> targetType);

    @Nullable
    Object convert(@Nullable Object source, @Nullable TypeDescriptor sourceType, TypeDescriptor targetType);

}
```

其 UML 类图如下：

[![ConversionService 类图](http://static.iocoder.cn/5dae2df55357fdf3f3e5a3c7470e5c6b)](http://static.iocoder.cn/5dae2df55357fdf3f3e5a3c7470e5c6b)ConversionService 类图

- **ConfigurableConversionService**：ConversionService 的配置接口，继承 ConversionService 和 ConverterRegistry 两个接口，用于合并他们两者的操作，以便于通过 add 和 remove 的方式添加和删除转换器。
- **GenericConversionService**：ConversionService 接口的基础实现，适用于大部分条件下的转换工作，通过 ConfigurableConversionService 接口间接地将 ConverterRegistry 实现为注册 API 。
- **DefaultConversionService**：ConversionService 接口的默认实现，适用于大部分条件下的转换工作。

------

回归到 TypeConverterDelegate 的 `#convertIfNecessary(String propertyName, Object oldValue, @Nullable Object newValue, Class<T> requiredType, TypeDescriptor typeDescriptor)` 方法，在该方法中，如果没有自定义的属性编辑器，则调用 ConversionService 接口的 `#convert(...)`，方法定义如下：

```
// ConversionService.java

Object convert(@Nullable Object source, @Nullable TypeDescriptor sourceType, TypeDescriptor targetType);
```

- `source` ：要转换的源对象，可以为 `null` 。
- `sourceType`：`source` 的类型的上下文，如果 `source` 为 `null` ，则可以为 `null` 。
- `targetType` ：`source` 要转换的类型的上下文。

`#convert(...)` 方法，将给定的源对象 `source` 转换为指定的 `targetType` 。TypeDescriptors 提供有关发生转换的源位置和目标位置的附加上下文，通常是对象字段或属性位置。该方法由子类 GenericConversionService 实现：

```
// GenericConversionService.java

@Override
@Nullable
public Object convert(@Nullable Object source, @Nullable TypeDescriptor sourceType, TypeDescriptor targetType) {
    Assert.notNull(targetType, "Target type to convert to cannot be null");
    // <1> 如果 sourceType 为空，则直接处理结果
    if (sourceType == null) {
        Assert.isTrue(source == null, "Source must be [null] if source type == [null]");
        return handleResult(null, targetType, convertNullSource(null, targetType));
    }
    // <2> 如果类型不对，抛出 IllegalArgumentException 异常
    if (source != null && !sourceType.getObjectType().isInstance(source)) {
        throw new IllegalArgumentException("Source to convert from must be an instance of [" +
                sourceType + "]; instead it was a [" + source.getClass().getName() + "]");
    }
    // <3> 获得对应的 GenericConverter 对象
    GenericConverter converter = getConverter(sourceType, targetType);
    // <4> 如果 converter 非空，则进行转换，然后再处理结果
    if (converter != null) {
        // <4.1> 执行转换
        Object result = ConversionUtils.invokeConverter(converter, source, sourceType, targetType);
        // <4.2> 处理器结果
        return handleResult(sourceType, targetType, result);
    }
    // <5> 处理 converter 为空的情况
    return handleConverterNotFound(source, sourceType, targetType);
}
```

- `<1>` 处，如果 `sourceType` 为空，则直接处理结果。

- `<2>` 处，如果类型不对，抛出 IllegalArgumentException 异常。

- `<3>` 处，调用 `#getConverter(TypeDescriptor sourceType, TypeDescriptor targetType)` 方法，获取 GenericConverter 对象 `converter` 。

- `<4>` 处，如果 `converter` 非空，则进行转换，然后再处理结果。

  - `<4.1>` 处，调用 `ConversionUtils#invokeConverter(GenericConverter converter, Object source, TypeDescriptor sourceType, TypeDescriptor targetType)` 方法，执行转换。代码如下：

    ```
    // ConversionUtils.java
    
    @Nullable
    public static Object invokeConverter(GenericConverter converter, @Nullable Object source,
            TypeDescriptor sourceType, TypeDescriptor targetType) {
        try {
            // 执行转换
            return converter.convert(source, sourceType, targetType);
        } catch (ConversionFailedException ex) {
            throw ex;
        } catch (Throwable ex) {
            throw new ConversionFailedException(sourceType, targetType, source, ex);
        }
    }
    ```

    - 【重要】在这里，我们看到执行转换。

  - `<4.2>` 处，调用 `#handleResult(TypeDescriptor sourceType, TypeDescriptor targetType, Object result)` 方法，处理结果。代码如下：

    ```
    // GenericConversionService.java
    
    @Nullable
    private Object handleResult(@Nullable TypeDescriptor sourceType, TypeDescriptor targetType, @Nullable Object result) {
    	if (result == null) {
    		assertNotPrimitiveTargetType(sourceType, targetType);
    	}
    	return result;
    }
    
    private void assertNotPrimitiveTargetType(@Nullable TypeDescriptor sourceType, TypeDescriptor targetType) {
    	if (targetType.isPrimitive()) {
    		throw new ConversionFailedException(sourceType, targetType, null,
    				new IllegalArgumentException("A null value cannot be assigned to a primitive type"));
    	}
    }
    ```

    - 实际上，是**校验**结果。

- `<5>` 处，调用 `#handleConverterNotFound(Object source, TypeDescriptor sourceType, TypeDescriptor targetType)` 方法，处理 `converter` 为空的情况。代码如下：

  ```
  // GenericConversionService.java
  
  @Nullable
  private Object handleConverterNotFound(
          @Nullable Object source, @Nullable TypeDescriptor sourceType, TypeDescriptor targetType) {
      // 情况一，如果 source 为空，则返回空
      if (source == null) {
          assertNotPrimitiveTargetType(sourceType, targetType);
          return null;
      }
      // 情况二，如果 sourceType 为空，或者 targetType 是 sourceType 的子类，则返回 source
      if ((sourceType == null || sourceType.isAssignableTo(targetType)) &&
              targetType.getObjectType().isInstance(source)) {
          return source;
      }
      // 抛出 ConverterNotFoundException 异常
      throw new ConverterNotFoundException(sourceType, targetType);
  }
  ```

------

😈 逻辑有点点长，我们先从 `#getConverter(TypeDescriptor sourceType, TypeDescriptor targetType)` 方法，获取 GenericConverter 对象 `converter` 。代码如下：

```
// GenericConversionService.java

@Nullable
protected GenericConverter getConverter(TypeDescriptor sourceType, TypeDescriptor targetType) {
    // 创建 ConverterCacheKey 对象
    ConverterCacheKey key = new ConverterCacheKey(sourceType, targetType);
    // 从 converterCache 缓存中，获得 GenericConverter 对象 converter
    GenericConverter converter = this.converterCache.get(key);
    // 如果获得到，则返回 converter
    if (converter != null) {
        return (converter != NO_MATCH ? converter : null);
    }

    // 如果获取不到，则从 converters 中查找
    converter = this.converters.find(sourceType, targetType);
    // 如果查找不到，则获得默认的 Converter 对象
    if (converter == null) {
        converter = getDefaultConverter(sourceType, targetType);
    }

    // 如果找到 converter ，则添加 converter 到 converterCache 中，并返回 converter
    if (converter != null) {
        this.converterCache.put(key, converter);
        return converter;
    }

    // 如果找不到 converter ，则添加 NO_MATCH 占位符到 converterCache 中，并返回 null
    this.converterCache.put(key, NO_MATCH);
    return null;
}
```

- 这段代码意图非常明确，从 `converterCache` 缓存中获取，如果存在返回，否则从 `converters` 中获取，然后加入到 `converterCache` 缓存中。

- `converterCache` 和 `converters` 是 GenericConversionService 维护的两个很重要的对象，代码如下：

  ```
  // GenericConversionService.java
  
  /**
   * 所有 Converter 集合的封装对象
   */
  private final Converters converters = new Converters();
  
  /**
   * GenericConverter 缓存
   */
  private final Map<ConverterCacheKey, GenericConverter> converterCache = new ConcurrentReferenceHashMap<>(64);
  ```

------

Converters 是 GenericConversionService 内部类，用于管理所有注册的转换器，其内部维护一个 Set 和 Map 的数据结构用于管理转换器，代码如下：

```
// GenericConversionService.java#Converters

private final Set<GenericConverter> globalConverters = new LinkedHashSet<>();

private final Map<ConvertiblePair, ConvertersForPair> converters = new LinkedHashMap<>(36);
```

- 同时提供了相应的方法（如 add、remove）操作这两个集合。

在 `#getConverter(TypeDescriptor sourceType, TypeDescriptor targetType)` 方法中，如果缓存 `converterCache` 中不存在，则调用 Converters 对象的 `#find(TypeDescriptor sourceType, TypeDescriptor targetType)` 方法，查找相应的 GenericConverter，如下：

```
// GenericConversionService.java#Converters

@Nullable
public GenericConverter find(TypeDescriptor sourceType, TypeDescriptor targetType) {
    // Search the full type hierarchy
    List<Class<?>> sourceCandidates = getClassHierarchy(sourceType.getType());
    List<Class<?>> targetCandidates = getClassHierarchy(targetType.getType());
    // 遍历 sourceCandidates 数组
    for (Class<?> sourceCandidate : sourceCandidates) {
        // 遍历 targetCandidates 数组
        for (Class<?> targetCandidate : targetCandidates) {
            // 创建 ConvertiblePair 对象
            ConvertiblePair convertiblePair = new ConvertiblePair(sourceCandidate, targetCandidate);
            // 获得 GenericConverter 对象
            GenericConverter converter = getRegisteredConverter(sourceType, targetType, convertiblePair);
            if (converter != null) {
                return converter;
            }
        }
    }
    return null;
}

@Nullable
private GenericConverter getRegisteredConverter(TypeDescriptor sourceType,
        TypeDescriptor targetType, ConvertiblePair convertiblePair) {
    // Check specifically registered converters
    // 从 converters 中，获得 converter
    ConvertersForPair convertersForPair = this.converters.get(convertiblePair);
    if (convertersForPair != null) {
        GenericConverter converter = convertersForPair.getConverter(sourceType, targetType);
        if (converter != null) {
            return converter;
        }
    }
    // Check ConditionalConverters for a dynamic match
    // 从 globalConverters 中，获得 globalConverter
    for (GenericConverter globalConverter : this.globalConverters) {
        if (((ConditionalConverter) globalConverter).matches(sourceType, targetType)) {
            return globalConverter;
        }
    }
    return null;
}
```

- 在 `#find(TypeDescriptor sourceType, TypeDescriptor targetT)` 方法中，会根据 `sourceType` 和 `targetType` 去查询 Converters 中维护的 Map 中是否包括支持的注册类型。如果存在返回 GenericConverter ，如果没有存在返回 `null` 。

------

当得到 GenericConverter 后，则调用其 `#convert(Object source, TypeDescriptor sourceType, TypeDescriptor targetType)` 方法，进行类型转换。代码如下：

```
// GenericConverter.java

Object convert(@Nullable Object source, TypeDescriptor sourceType, TypeDescriptor targetType);
```

到这里我们就可以得到 Bean 属性定义的真正类型了。

**GenericConverter 接口**

GenericConverter 是一个转换接口，一个用于在两种或更多种类型之间转换的通用型转换器接口。它是 Converter SPI 体系中最灵活的，也是最复杂的接口，灵活性在于 GenericConverter 可以支持在多个源/目标类型对之间进行转换，同时也可以在类型转换过程中访问源/目标字段上下文。由于该接口足够复杂，所有当更简单的 Converter 或 ConverterFactory 接口足够使用时，通常不应使用此接口。其定义如下：

```
// GenericConverter.java

public interface GenericConverter {

    @Nullable
    Set<ConvertiblePair> getConvertibleTypes();

    @Nullable
    Object convert(@Nullable Object source, TypeDescriptor sourceType, TypeDescriptor targetType);

}
```

GenericConverter 的子类有这么多（看类名就知道是干嘛的了）：

[![GenericConverter 子类](http://static.iocoder.cn/ae851c788e7011fd6e325056cab2f797)](http://static.iocoder.cn/ae851c788e7011fd6e325056cab2f797)GenericConverter 子类

我们看一个子类的实现 StringToArrayConverter，该子类将逗号分隔的 String 转换为 Array 。代码如下：

```
// StringToArrayConverter.java

final class StringToArrayConverter implements ConditionalGenericConverter {

	private final ConversionService conversionService;

	public StringToArrayConverter(ConversionService conversionService) {
		this.conversionService = conversionService;
	}

	@Override
	public Set<ConvertiblePair> getConvertibleTypes() {
		return Collections.singleton(new ConvertiblePair(String.class, Object[].class));
	}

	@Override
	public boolean matches(TypeDescriptor sourceType, TypeDescriptor targetType) {
		return ConversionUtils.canConvertElements(sourceType, targetType.getElementTypeDescriptor(),
				this.conversionService);
	}

	@Override
	@Nullable
	public Object convert(@Nullable Object source, TypeDescriptor sourceType, TypeDescriptor targetType) {
		if (source == null) {
			return null;
		}
		// 按照 , 分隔成字符串数组
		String string = (String) source;
		String[] fields = StringUtils.commaDelimitedListToStringArray(string);
		// 获得 TypeDescriptor 对象
		TypeDescriptor targetElementType = targetType.getElementTypeDescriptor();
		Assert.state(targetElementType != null, "No target element type");
		// 创建目标数组
		Object target = Array.newInstance(targetElementType.getType(), fields.length);
		// 遍历 fields 数组，逐个转换
		for (int i = 0; i < fields.length; i++) {
			String sourceElement = fields[i];
			// 执行转换
			Object targetElement = this.conversionService.convert(sourceElement.trim(), sourceType, targetElementType);
			// 设置到 target 中
			Array.set(target, i, targetElement);
		}
		return target;
	}

}
```

在类型转换体系中，Spring 提供了非常多的类型转换器，除了上面的 GenericConverter，还有 Converter、ConditionalConverter、ConverterFactory。

------

**Converter**

Converter 是一个将 `<S>` 类型的源对象转换为 `<T>` 类型的目标对象的转换器。该接口是线程安全的，所以可以共享。代码如下:

```
// Converter.java

public interface Converter<S, T> {

    @Nullable
    T convert(S source);

}
```

子类如下：

[![Converter 类图](http://static.iocoder.cn/7846fde6784243ff50c21366a438dab6)](http://static.iocoder.cn/7846fde6784243ff50c21366a438dab6)Converter 类图

------

**ConditionalConverter**

ConditionalConverter 接口用于表示有条件的类型转换，通过转入的`sourceType` 与 `targetType` 判断转换能否匹配，只有可匹配的转换才会调用convert 方法进行转换。代码如下：

```
// ConditionalConverter.java

public interface ConditionalConverter {

    boolean matches(TypeDescriptor sourceType, TypeDescriptor targetType);

}
```

ConditionalConverter 的子类如下：

[![ConditionalConverter 类图](http://static.iocoder.cn/8a7276c5d8bcdd7a724e69147655a710)](http://static.iocoder.cn/8a7276c5d8bcdd7a724e69147655a710)ConditionalConverter 类图

------

**ConverterFactory**

一个用于“远程”转换的转换工厂，可以将对象从 `<S>` 转换为 `<R>` 的子类型。代码如下：

```
// ConverterFactory.java

public interface ConverterFactory<S, R> {

    <T extends R> Converter<S, T> getConverter(Class<T> targetType);

}
```

子类如下：

[![ConverterFactory 类图](http://static.iocoder.cn/63de40b54f1bc3a032275065d43916a1)](http://static.iocoder.cn/63de40b54f1bc3a032275065d43916a1)ConverterFactory 类图

------

四种不同的转换器承载着不同的转换过程：

- Converter：用于 `1:1` 的 `source -> target` 类型转换。
- ConverterFactory：用于 `1:N` 的 `source -> target` 类型转换。
- GenericConverter用于 `N:N` 的 `source -> target` 类型转换。
- ConditionalConverter：有条件的 `source -> target` 类型转换。

------

**GenericConversionService**

转换器介绍完了，我们再次回归到 ConversionService 接口中去，该接口定义了两类方法：

- `canConvert(sourceType, targetType)` 方法，用于判 `sourceType` 能否转成 `targetType` 。
- `convert(source, targetType)` 方法，用于将 `source` 转成转入的 TargetType 类型实例。

这两类方法都是在 GenericConversionService 中实现。
类 GenericConversionService 实现 ConfigurableConversionService 接口，而 ConfigurableConversionService 接口继承 ConversionService 和 ConverterRegistry。
ConverterRegistry 提供了类型转换器的管理功能，他提供了四个 add 和一个 remove 方法，支持注册/删除相应的类型转换器。

GenericConversionService 作为一个基础实现类，它即支持了不同类型之间的转换，也对各类型转换器进行管理，主要是通过一个 Map 类型的 `converterCache` 和一个内部类 Converters 。在上面已经分析了 GenericConversionService 执行类型转换的过程 `#cover(...)` 方法。下面我们就一个 `addConverter(Converter<?, ?> converter)` 方法，来看看它是如何完成转换器的注入的工作的。代码如下：

```
// GenericConversionService.java

@Override
public void addConverter(Converter<?, ?> converter) {
    // <1> 获取 ResolvableType 对象，基于 converter.getClass() 类
    ResolvableType[] typeInfo = getRequiredTypeInfo(converter.getClass(), Converter.class);
    // <1> 如果获取不到，并且 converter 是 DecoratingProxy 类型，则基于 ((DecoratingProxy) converter).getDecoratedClass() 类
    if (typeInfo == null && converter instanceof DecoratingProxy) {
        typeInfo = getRequiredTypeInfo(((DecoratingProxy) converter).getDecoratedClass(), Converter.class);
    }
    // 如果获取不到，抛出 IllegalArgumentException 异常
    if (typeInfo == null) {
        throw new IllegalArgumentException("Unable to determine source type <S> and target type <T> for your " +
                "Converter [" + converter.getClass().getName() + "]; does the class parameterize those types?");
    }
    // <2> 封装成 ConverterAdapter 对象，添加到 converters 中
    addConverter(new ConverterAdapter(converter, typeInfo[0], typeInfo[1]));
}
```

- `<1>` 首先，根据 `converter` 获取 ResolvableType 数组。
- `<2>` 然后，将其与 `converter` 封装成一个 ConverterAdapter 实例。
- `<2>` 最后，调用 `#addConverter(GenericConverter converter)` 方法，添加到 `converters` 中。
- ResolvableType 用于封装 Java 的 [Type](https://juejin.im/post/5adefaba518825670e5cb44d) 类型。
- ConverterAdapter 则是 Converter 的一个适配器， 它实现了 GenericConverter 和 ConditionalConverter 两个类型转换器。

其中，`#addConverter(GenericConverter converter)` 方法，代码如下：

```
// GenericConversionService.java

@Override
public void addConverter(GenericConverter converter) {
    // 添加到 converters 中
    this.converters.add(converter);
    // 过期缓存
    invalidateCache();
}
```

直接调用内部类 Converters 的 `#add(GenericConverter converter)` 方法，代码如下：

```
// GenericConversionService.java

public void add(GenericConverter converter) {
    // 获得 ConvertiblePair 集合
    Set<ConvertiblePair> convertibleTypes = converter.getConvertibleTypes();
    // 如果为空，并且 converter 是 ConditionalConverter 类型，则添加到 【globalConverters】 中
    if (convertibleTypes == null) {
        Assert.state(converter instanceof ConditionalConverter,
                "Only conditional converters may return null convertible types");
        this.globalConverters.add(converter);
    } else {
        // 通过迭代的方式依次添加【converters】中
        for (ConvertiblePair convertiblePair : convertibleTypes) {
            // 从 converters 中，获得 ConvertersForPair 对象
            ConvertersForPair convertersForPair = getMatchableConverters(convertiblePair);
            // 添加 converter 到 ConvertersForPair 中
            convertersForPair.add(converter);
        }
    }
}
```

- 首先调用 GenericConverter 的 `#getConvertibleTypes()` 方法，获取 ConvertiblePair 集合。如果为空，则加入到 `globalConverters` 集合中，否则通过迭代的方式依次添加 `converters` 中。

- ConvertiblePair 为 source-to-target 的持有者，它持有 `source` 和 `target` 的 class 类型，代码如下：

  ```
  // GenericConverter.java#ConvertiblePair
  
  final class ConvertiblePair {
  
      private final Class<?> sourceType;
      private final Class<?> targetType;
  
      // 省略其他代码
  }
  ```

在迭代过程中会根据 ConvertiblePair 获取相应的 ConvertersForPair 对象，然后添加 `converter` 转换器加入其中。ConvertiblePair 用于管理使用特定GenericConverter.ConvertiblePair 注册的转换器。代码如下：

```
// GenericConversionService.java#ConvertersForPair

private static class ConvertersForPair {

   private final LinkedList<GenericConverter> converters = new LinkedList<>();

    public void add(GenericConverter converter) {
        this.converters.addFirst(converter);
    }

    @Nullable
    public GenericConverter getConverter(TypeDescriptor sourceType, TypeDescriptor targetType) {
        for (GenericConverter converter : this.converters) {
            if (!(converter instanceof ConditionalGenericConverter) ||
                    ((ConditionalGenericConverter) converter).matches(sourceType, targetType)) {
                return converter;
            }
        }
        return null;
    }

}
```

- 其实内部就是维护一个 LinkedList 集合。他内部有两个方法：`#add(GenericConverter converter)` 和 `getConverter(TypeDescriptor sourceType, TypeDescriptor targetType)`，实现较为简单，这里就不多介绍了。

------

**DefaultConversionService**

DefaultConversionService 是 ConversionService 的默认实现，它继承 GenericConversionService，GenericConversionService 主要用于转换器的注册和调用，DefaultConversionService 则是为 ConversionService 体系提供一些默认的转换器。

在 DefaultConversionService 构造方法中就会添加默认的 Converter ，代码如下：

```
// DefaultConversionService.java

public DefaultConversionService() {
    addDefaultConverters(this);
}

public static void addDefaultConverters(ConverterRegistry converterRegistry) {
    addScalarConverters(converterRegistry);
    addCollectionConverters(converterRegistry);

    converterRegistry.addConverter(new ByteBufferConverter((ConversionService) converterRegistry));
    converterRegistry.addConverter(new StringToTimeZoneConverter());
    converterRegistry.addConverter(new ZoneIdToTimeZoneConverter());
    converterRegistry.addConverter(new ZonedDateTimeToCalendarConverter());

    converterRegistry.addConverter(new ObjectToObjectConverter());
    converterRegistry.addConverter(new IdToEntityConverter((ConversionService) converterRegistry));
    converterRegistry.addConverter(new FallbackObjectToStringConverter());
    converterRegistry.addConverter(new ObjectToOptionalConverter((ConversionService) converterRegistry));
}
```

当然它还提供了一些其他的方法如 `#addCollectionConverters(ConverterRegistry converterRegistry)`、`addScalarConverters(ConverterRegistry converterRegistry)` 方法，用于注册其他类型的转换器。

------

至此，从 Bean 属性的转换，到 Spring ConversionService 体系的转换器 Converter 以及转换器的管理都介绍完毕了，下篇我们将分析如何利用 ConversionService 实现**自定义类型**转换器。

> 艿艿：因为本文是基于调用的过程，进行解析。所以胖友可以自己在总结整理下。
>
> 实际上，大体的调用流是如下：
>
> TypeConverterSupport => ConversionService => Converter

# 10、BeanWrapper

在实例化 Bean 阶段，我们从 BeanDefinition 得到的并不是我们最终想要的 Bean 实例，而是 BeanWrapper 实例，如下：

[![img](http://static.iocoder.cn/dfd73f60540dd579297a1f9df9f95fe8)](http://static.iocoder.cn/dfd73f60540dd579297a1f9df9f95fe8)

所以这里 BeanWrapper 是一个从 BeanDefinition 到 Bean 直接的**中间产物**，我们可以称它为”低级 bean“。在一般情况下，我们不会在实际项目中用到它。BeanWrapper 是 Spring 框架中重要的组件类，它就相当于一个代理类，Spring 委托 BeanWrapper 完成 Bean 属性的填充工作。在 Bean 实例被 InstantiationStrategy 创建出来后，Spring 容器会将 Bean 实例通过 BeanWrapper 包裹起来，是通过如如下代码实现：

[![img](http://static.iocoder.cn/3a5c719e69c1113dcc8cdc7ff124929d)](http://static.iocoder.cn/3a5c719e69c1113dcc8cdc7ff124929d)

- `beanInstance` 就是我们实例出来的 bean 实例，通过构造一个 BeanWrapper 实例对象进行包裹，如下：

  ```
  // BeanWrapperImpl.java
  
  public BeanWrapperImpl(Object object) {
      super(object);
  }
  
  protected AbstractNestablePropertyAccessor(Object object) {
      registerDefaultEditors();
      setWrappedInstance(object);
  }
  ```

------

下面小编就 BeanWrapper 来进行分析说明，先看整体的结构：

[![2018101210001](http://static.iocoder.cn/fea787ac555caf4dab31fb3bc889dc8d)](http://static.iocoder.cn/fea787ac555caf4dab31fb3bc889dc8d)2018101210001

从上图可以看出 BeanWrapper 主要继承三个核心接口：PropertyAccessor、PropertyEditorRegistry、TypeConverter。

**PropertyAccessor**

> 可以访问属性的通用型接口（例如对象的 bean 属性或者对象中的字段），作为 BeanWrapper 的基础接口。

```
// PropertyAccessor.java

public interface PropertyAccessor {

    String NESTED_PROPERTY_SEPARATOR = ".";
    char NESTED_PROPERTY_SEPARATOR_CHAR = '.';

    String PROPERTY_KEY_PREFIX = "[";
    char PROPERTY_KEY_PREFIX_CHAR = '[';

    String PROPERTY_KEY_SUFFIX = "]";
    char PROPERTY_KEY_SUFFIX_CHAR = ']';

    boolean isReadableProperty(String propertyName);

    boolean isWritableProperty(String propertyName);

    Class<?> getPropertyType(String propertyName) throws BeansException;
    TypeDescriptor getPropertyTypeDescriptor(String propertyName) throws BeansException;
    Object getPropertyValue(String propertyName) throws BeansException;

    void setPropertyValue(String propertyName, @Nullable Object value) throws BeansException;
    void setPropertyValue(PropertyValue pv) throws BeansException;
    void setPropertyValues(Map<?, ?> map) throws BeansException;
    void setPropertyValues(PropertyValues pvs) throws BeansException;
    void setPropertyValues(PropertyValues pvs, boolean ignoreUnknown)
    throws BeansException;
    void setPropertyValues(PropertyValues pvs, boolean ignoreUnknown, boolean ignoreInvalid)
    throws BeansException;

}
```

就上面的源码我们可以分解为四类方法：

- `#isReadableProperty(String propertyName)` 方法：判断指定 property 是否可读，是否包含 getter 方法。
- `#isWritableProperty(String propertyName)` 方法：判断指定 property 是否可写,是否包含 setter 方法。
- `#getPropertyType(...)` 方法：获取指定 propertyName 的类型
- `#setPropertyValue(...)` 方法：设置指定 propertyValue 。

------

**PropertyEditorRegistry**

> 用于注册 JavaBean 的 PropertyEditors，对 PropertyEditorRegistrar 起核心作用的中心接口。由 BeanWrapper 扩展，BeanWrapperImpl 和 DataBinder 实现。

```
// PropertyEditorRegistry.java

public interface PropertyEditorRegistry {

    void registerCustomEditor(Class<?> requiredType, PropertyEditor propertyEditor);

    void registerCustomEditor(@Nullable Class<?> requiredType, @Nullable String propertyPath, PropertyEditor propertyEditor);

    @Nullable
    PropertyEditor findCustomEditor(@Nullable Class<?> requiredType, @Nullable String propertyPath);

}
```

根据接口提供的方法，PropertyEditorRegistry 就是用于 PropertyEditor 的注册和发现，而 PropertyEditor 是 Java 内省里面的接口，用于改变指定 property 属性的类型。

------

**TypeConverter**

> 定义类型转换的接口，通常与 PropertyEditorRegistry 接口一起实现（但不是必须），但由于 TypeConverter 是基于线程不安全的 PropertyEditors ，因此 TypeConverters 本身也不被视为线程安全。
> 这里小编解释下，在 Spring 3 后，不在采用 PropertyEditors 类作为 Spring 默认的类型转换接口，而是采用 ConversionService 体系，但 ConversionService 是线程安全的，所以在 Spring 3 后，如果你所选择的类型转换器是 ConversionService 而不是 PropertyEditors 那么 TypeConverters 则是线程安全的。

```
public interface TypeConverter {

    <T> T convertIfNecessary(Object value, Class<T> requiredType) throws TypeMismatchException;
    <T> T convertIfNecessary(Object value, Class<T> requiredType, MethodParameter methodParam)
            throws TypeMismatchException;
    <T> T convertIfNecessary(Object value, Class<T> requiredType, Field field)
            throws TypeMismatchException;

}
```

------

BeanWrapper 继承上述三个接口，那么它就具有三重身份：

- 属性编辑器
- 属性编辑器注册表
- 类型转换器

BeanWrapper 继承 ConfigurablePropertyAccessor 接口，该接口除了继承上面介绍的三个接口外还集成了 Spring 的 ConversionService 类型转换体系。

```
// ConfigurablePropertyAccessor.java

public interface ConfigurablePropertyAccessor extends PropertyAccessor, PropertyEditorRegistry, TypeConverter {

    void setConversionService(@Nullable ConversionService conversionService);

    @Nullable
    ConversionService getConversionService();

    void setExtractOldValueForEditor(boolean extractOldValueForEditor);

    boolean isExtractOldValueForEditor();

    void setAutoGrowNestedPaths(boolean autoGrowNestedPaths);

    boolean isAutoGrowNestedPaths();

}
```

`#setConversionService(ConversionService conversionService)` 和 `#getConversionService()` 方法，则是用于集成 Spring 的 ConversionService 类型转换体系。

**BeanWrapper**

> Spring 的 低级 JavaBean 基础结构的接口，一般不会直接使用，而是通过 BeanFactory 或者 DataBinder 隐式使用。它提供分析和操作标准 JavaBeans 的操作：获取和设置属性值、获取属性描述符以及查询属性的可读性/可写性的能力。

```
// BeanWrapper.java
public interface BeanWrapper extends ConfigurablePropertyAccessor {

    void setAutoGrowCollectionLimit(int autoGrowCollectionLimit);
    int getAutoGrowCollectionLimit();

    Object getWrappedInstance();
    Class<?> getWrappedClass();

    PropertyDescriptor[] getPropertyDescriptors();
    PropertyDescriptor getPropertyDescriptor(String propertyName) throws InvalidPropertyException;

}
```

下面几个方法比较重要：

- `#getWrappedInstance()` 方法：获取包装对象的实例。
- `#getWrappedClass()` 方法：获取包装对象的类型。
- `#getPropertyDescriptors()` 方法：获取包装对象所有属性的 PropertyDescriptor 就是这个属性的上下文。
- `#getPropertyDescriptor(String propertyName)` 方法：获取包装对象指定属性的上下文。

------

**BeanWrapperImpl**

> BeanWrapper 接口的默认实现，用于对Bean的包装，实现上面接口所定义的功能很简单包括设置获取被包装的对象，获取被包装bean的属性描述器

------

**小结**

BeanWrapper 体系相比于 Spring 中其他体系是比较简单的，它作为 BeanDefinition 向 Bean 转换过程中的中间产物，承载了 Bean 实例的包装、类型转换、属性的设置以及访问等重要作用。