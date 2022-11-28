# 1、IoC 之开启 Bean 的加载

**本文主要基于 Spring 5.0.6.RELEASE**

摘要: 原创出处 http://cmsblogs.com/?p=2806 「小明哥」，谢谢！

作为「小明哥」的忠实读者，「老艿艿」略作修改，记录在理解过程中，参考的资料。

------

﻿[![Spring IoC 作用](http://static.iocoder.cn/181705b2eb48364c7626b59397e2af1c)](http://static.iocoder.cn/181705b2eb48364c7626b59397e2af1c)Spring IoC 作用

(此图参考《Spring 揭秘》)

Spring IoC 容器所起的作用如上图所示，它会以某种方式加载 Configuration Metadata，将其解析注册到容器内部，然后回根据这些信息绑定整个系统的对象，最终组装成一个可用的基于轻量级容器的应用系统。

Spring 在实现上述功能中，将整个流程分为两个阶段：容器初始化阶段和加载bean 阶段。分别如下：

1. 容器初始化阶段

   ：

   - 首先，通过某种方式加载 Configuration Metadata (主要是依据 Resource、ResourceLoader 两个体系) 。
   - 然后，容器会对加载的 Configuration MetaData 进行解析和分析，并将分析的信息组装成 BeanDefinition 。
   - 最后，将 BeanDefinition 保存注册到相应的 BeanDefinitionRegistry 中。
   - 至此，Spring IoC 的初始化工作完成。

2. 加载 Bean 阶段

   ：

   - 经过容器初始化阶段后，应用程序中定义的 bean 信息已经全部加载到系统中了，当我们显示或者隐式地调用 `BeanFactory#getBean(...)` 方法时，则会触发加载 Bean 阶段。
   - 在这阶段，容器会首先检查所请求的对象是否已经初始化完成了，如果没有，则会根据注册的 Bean 信息实例化请求的对象，并为其注册依赖，然后将其返回给请求方。
   - 至此第二个阶段也已经完成。

第一个阶段前面已经用了 10 多篇博客深入分析了（总结参考 [《【死磕 Spring】—— IoC 之装载 BeanDefinitions 总结》](http://svip.iocoder.cn/Spring/IoC-load-BeanDefinitions-summary)）。所以从这篇开始分析第二个阶段：加载 Bean 阶段。

## 1. getBean

当我们显示或者隐式地调用 `BeanFactory#getBean(String name)` 方法时，则会触发加载 Bean 阶段。代码如下：

```
// AbstractBeanFactory.java

@Override
public Object getBean(String name) throws BeansException {
	return doGetBean(name, null, null, false);
}
```

- 内部调用 `doGetBean(String name, final Class<T> requiredType, Object[] args, boolean typeCheckOnly)` 方法，其接受四个方法参数：
  - `name` ：要获取 Bean 的名字
  - `requiredType` ：要获取 bean 的类型
  - `args` ：创建 Bean 时传递的参数。这个参数仅限于创建 Bean 时使用。
  - `typeCheckOnly` ：是否为类型检查。

## 2. doGetBean

`#doGetBean(String name, final Class<T> requiredType, Object[] args, boolean typeCheckOnly)` 方法，代码比较长，各位耐心看下。代码如下：

```
// AbstractBeanFactory.java

protected <T> T doGetBean(final String name, @Nullable final Class<T> requiredType,
        @Nullable final Object[] args, boolean typeCheckOnly) throws BeansException {
    // <1> 返回 bean 名称，剥离工厂引用前缀。
    // 如果 name 是 alias ，则获取对应映射的 beanName 。
    final String beanName = transformedBeanName(name);
    Object bean;

    // 从缓存中或者实例工厂中获取 Bean 对象
    // Eagerly check singleton cache for manually registered singletons.
    Object sharedInstance = getSingleton(beanName);
    if (sharedInstance != null && args == null) {
        if (logger.isTraceEnabled()) {
            if (isSingletonCurrentlyInCreation(beanName)) {
                logger.trace("Returning eagerly cached instance of singleton bean '" + beanName +
                        "' that is not fully initialized yet - a consequence of a circular reference");
            } else {
                logger.trace("Returning cached instance of singleton bean '" + beanName + "'");
            }
        }
        // <2> 完成 FactoryBean 的相关处理，并用来获取 FactoryBean 的处理结果
        bean = getObjectForBeanInstance(sharedInstance, name, beanName, null);
    } else {
        // Fail if we're already creating this bean instance:
        // We're assumably within a circular reference.
        // <3> 因为 Spring 只解决单例模式下得循环依赖，在原型模式下如果存在循环依赖则会抛出异常。
        if (isPrototypeCurrentlyInCreation(beanName)) {
            throw new BeanCurrentlyInCreationException(beanName);
        }

        // <4> 如果容器中没有找到，则从父类容器中加载
        // Check if bean definition exists in this factory.
        BeanFactory parentBeanFactory = getParentBeanFactory();
        if (parentBeanFactory != null && !containsBeanDefinition(beanName)) {
            // Not found -> check parent.
            String nameToLookup = originalBeanName(name);
            if (parentBeanFactory instanceof AbstractBeanFactory) {
                return ((AbstractBeanFactory) parentBeanFactory).doGetBean(
                        nameToLookup, requiredType, args, typeCheckOnly);
            } else if (args != null) {
                // Delegation to parent with explicit args.
                return (T) parentBeanFactory.getBean(nameToLookup, args);
            } else if (requiredType != null) {
                // No args -> delegate to standard getBean method.
                return parentBeanFactory.getBean(nameToLookup, requiredType);
            } else {
                return (T) parentBeanFactory.getBean(nameToLookup);
            }
        }

        // <5> 如果不是仅仅做类型检查则是创建bean，这里需要记录
        if (!typeCheckOnly) {
            markBeanAsCreated(beanName);
        }

        try {
            // <6> 从容器中获取 beanName 相应的 GenericBeanDefinition 对象，并将其转换为 RootBeanDefinition 对象
            final RootBeanDefinition mbd = getMergedLocalBeanDefinition(beanName);
            // 检查给定的合并的 BeanDefinition
            checkMergedBeanDefinition(mbd, beanName, args);

            // Guarantee initialization of beans that the current bean depends on.
            // <7> 处理所依赖的 bean
            String[] dependsOn = mbd.getDependsOn();
            if (dependsOn != null) {
                for (String dep : dependsOn) {
                    // 若给定的依赖 bean 已经注册为依赖给定的 bean
                    // 循环依赖的情况
                    if (isDependent(beanName, dep)) {
                        throw new BeanCreationException(mbd.getResourceDescription(), beanName,
                                "Circular depends-on relationship between '" + beanName + "' and '" + dep + "'");
                    }
                    // 缓存依赖调用 TODO 芋艿
                    registerDependentBean(dep, beanName);
                    try {
                        getBean(dep);
                    } catch (NoSuchBeanDefinitionException ex) {
                        throw new BeanCreationException(mbd.getResourceDescription(), beanName,
                                "'" + beanName + "' depends on missing bean '" + dep + "'", ex);
                    }
                }
            }

            // <8> bean 实例化
            // Create bean instance.
            if (mbd.isSingleton()) { // 单例模式
                sharedInstance = getSingleton(beanName, () -> {
                    try {
                        return createBean(beanName, mbd, args);
                    }
                    catch (BeansException ex) {
                        // Explicitly remove instance from singleton cache: It might have been put there
                        // eagerly by the creation process, to allow for circular reference resolution.
                        // Also remove any beans that received a temporary reference to the bean.
                        // 显式从单例缓存中删除 Bean 实例
                        // 因为单例模式下为了解决循环依赖，可能他已经存在了，所以销毁它。 TODO 芋艿
                        destroySingleton(beanName);
                        throw ex;
                    }
                });
                bean = getObjectForBeanInstance(sharedInstance, name, beanName, mbd);
            } else if (mbd.isPrototype()) { // 原型模式
                // It's a prototype -> create a new instance.
                Object prototypeInstance;
                try {
                    beforePrototypeCreation(beanName);
                    prototypeInstance = createBean(beanName, mbd, args);
                } finally {
                    afterPrototypeCreation(beanName);
                }
                bean = getObjectForBeanInstance(prototypeInstance, name, beanName, mbd);
            } else {
                // 从指定的 scope 下创建 bean
                String scopeName = mbd.getScope();
                final Scope scope = this.scopes.get(scopeName);
                if (scope == null) {
                    throw new IllegalStateException("No Scope registered for scope name '" + scopeName + "'");
                }try {
                    Object scopedInstance = scope.get(beanName, () -> {
                        beforePrototypeCreation(beanName);
                        try {
                            return createBean(beanName, mbd, args);
                        } finally {
                            afterPrototypeCreation(beanName);
                        }
                    });
                    bean = getObjectForBeanInstance(scopedInstance, name, beanName, mbd);
                } catch (IllegalStateException ex) {
                    throw new BeanCreationException(beanName,
                            "Scope '" + scopeName + "' is not active for the current thread; consider " +
                            "defining a scoped proxy for this bean if you intend to refer to it from a singleton",
                            ex);
                }
            }
        } catch (BeansException ex) {
            cleanupAfterBeanCreationFailure(beanName);
            throw ex;
        }
    }

    // <9> 检查需要的类型是否符合 bean 的实际类型
    // Check if required type matches the type of the actual bean instance.
    if (requiredType != null && !requiredType.isInstance(bean)) {
        try {
            T convertedBean = getTypeConverter().convertIfNecessary(bean, requiredType);
            if (convertedBean == null) {
                throw new BeanNotOfRequiredTypeException(name, requiredType, bean.getClass());
            }
            return convertedBean;
        } catch (TypeMismatchException ex) {
            if (logger.isTraceEnabled()) {
                logger.trace("Failed to convert bean '" + name + "' to required type '" +
                        ClassUtils.getQualifiedName(requiredType) + "'", ex);
            }
            throw new BeanNotOfRequiredTypeException(name, requiredType, bean.getClass());
        }
    }
    return (T) bean;
}
```

代码是相当长，处理逻辑也是相当复杂，下面将其进行拆分阐述。

- `<1>` 处，详细解析，见 [「2.1 获取 beanName」](http://svip.iocoder.cn/Spring/IoC-get-Bean-begin/#) 。
- `<2>` 处，详细解析，见 [「2.2 从单例 Bean 缓存中获取 Bean」](http://svip.iocoder.cn/Spring/IoC-get-Bean-begin/#) 。
- `<3>` 处，详细解析，见 [「2.3 原型模式依赖检查」](http://svip.iocoder.cn/Spring/IoC-get-Bean-begin/#) 。
- `<4>` 处，详细解析，见 [「2.4 从 parentBeanFactory 获取 Bean」](http://svip.iocoder.cn/Spring/IoC-get-Bean-begin/#) 。
- `<5>` 处，详细解析，见 [「2.5 指定的 Bean 标记为已经创建或即将创建」](http://svip.iocoder.cn/Spring/IoC-get-Bean-begin/#) 。
- `<6>` 处，详细解析，见 [「2.6 获取 BeanDefinition」](http://svip.iocoder.cn/Spring/IoC-get-Bean-begin/#) 。
- `<7>` 处，详细解析，见 [「2.7 依赖 Bean 处理」](http://svip.iocoder.cn/Spring/IoC-get-Bean-begin/#) 。
- `<8>` 处，详细解析，见 [「2.8 不同作用域的 Bean 实例化」](http://svip.iocoder.cn/Spring/IoC-get-Bean-begin/#) 。
- `<9>` 处，详细解析，见 [「2.9 类型转换」](http://svip.iocoder.cn/Spring/IoC-get-Bean-begin/#) 。

#### 2.1 获取 beanName

对应代码段如下：

```
// AbstractBeanFactory.java

final String beanName = transformedBeanName(name);
```

- 这里传递的是 `name` 方法，不一定就是 beanName，可能是 aliasName ，也有可能是 FactoryBean ，所以这里需要调用 `#transformedBeanName(String name)` 方法，对 `name` 进行一番转换。代码如下：

  ```
  // AbstractBeanFactory.java
  
  protected String transformedBeanName(String name) {
  	return canonicalName(BeanFactoryUtils.transformedBeanName(name));
  }
  ```

  - 调用 `BeanFactoryUtils#transformedBeanName(String name)` 方法，去除 FactoryBean 的修饰符。代码如下：

    ```
    // BeanFactoryUtils.java
    
    /**
     * Cache from name with factory bean prefix to stripped name without dereference.
     *
     * 缓存 {@link #transformedBeanName(String)} 已经转换好的结果。
     *
     * @since 5.1
     * @see BeanFactory#FACTORY_BEAN_PREFIX
     */
    private static final Map<String, String> transformedBeanNameCache = new ConcurrentHashMap<>();
    /**
     * 去除 FactoryBean 的修饰符 &
     *
     * 如果 name 以 “&” 为前缀，那么会去掉该 "&" 。
     * 例如，name = "&studentService" ，则会是 name = "studentService"。
     *
     * Return the actual bean name, stripping out the factory dereference
     * prefix (if any, also stripping repeated factory prefixes if found).
     * @param name the name of the bean
     * @return the transformed name
     * @see BeanFactory#FACTORY_BEAN_PREFIX
     */
    public static String transformedBeanName(String name) {
        Assert.notNull(name, "'name' must not be null");
        if (!name.startsWith(BeanFactory.FACTORY_BEAN_PREFIX)) { // BeanFactory.FACTORY_BEAN_PREFIX = $
            return name;
        }
        // computeIfAbsent 方法，分成两种情况：
        //      1. 未存在，则进行计算执行，并将结果添加到缓存、
        //      2. 已存在，则直接返回，无需计算。
        return transformedBeanNameCache.computeIfAbsent(name, beanName -> {
            do {
                beanName = beanName.substring(BeanFactory.FACTORY_BEAN_PREFIX.length());
            } while (beanName.startsWith(BeanFactory.FACTORY_BEAN_PREFIX));
            return beanName;
        });
    }
    ```

    - 实际上，逻辑比较简单，就是去除传入 `name` 参数的 `"&"` 的前缀。

      > 小知识补充。假设配置了一个 FactoryBean 的名字为 `"abc"` ，那么获取 FactoryBean 创建的 Bean 时，使用 `"abc"` ，如果获取 FactoryBean 本身，使用 `"$abc"` 。另外，`&`定义在 `BeanFactory.FACTORY_BEAN_PREFIX = "&"` 上。

    - `transformedBeanNameCache` 集合的存在，是为了缓存转换后的结果。下次再获取相同的 `name` 时，直接返回缓存中的结果即可。

  - 调用 `#canonicalName(String name)` 方法，取指定的 `alias` 所表示的最终 beanName 。

    ```
    // SimpleAliasRegistry.java
    
    /** Map from alias to canonical name. */
    // key: alias
    // value: beanName
    private final Map<String, String> aliasMap = new ConcurrentHashMap<>(16);
    
    public String canonicalName(String name) {
    	String canonicalName = name;
    	// Handle aliasing...
    	String resolvedName;
    	// 循环，从 aliasMap 中，获取到最终的 beanName
    	do {
    		resolvedName = this.aliasMap.get(canonicalName);
    		if (resolvedName != null) {
    			canonicalName = resolvedName;
    		}
    	} while (resolvedName != null);
    	return canonicalName;
    }
    ```

    - 主要是一个循环获取 beanName 的过程，例如，别名 A 指向名称为 B 的 bean 则返回 B，若 别名 A 指向别名 B，别名 B 指向名称为 C 的 bean，则返回 C。

#### 2.2 从单例 Bean 缓存中获取 Bean

对应代码段如下：

```
// AbstractBeanFactory.java

// 从缓存中或者实例工厂中获取 Bean 对象
// Eagerly check singleton cache for manually registered singletons.
Object sharedInstance = getSingleton(beanName);
if (sharedInstance != null && args == null) {
	if (logger.isTraceEnabled()) {
		if (isSingletonCurrentlyInCreation(beanName)) {
			logger.trace("Returning eagerly cached instance of singleton bean '" + beanName +
					"' that is not fully initialized yet - a consequence of a circular reference");
		} else {
			logger.trace("Returning cached instance of singleton bean '" + beanName + "'");
		}
	}
	// <x> 完成 FactoryBean 的相关处理，并用来获取 FactoryBean 的处理结果
	bean = getObjectForBeanInstance(sharedInstance, name, beanName, null);
}
```

- 我们知道单例模式的 Bean 在整个过程中只会被创建一次。第一次创建后会将该 Bean 加载到缓存中。后面，在获取 Bean 就会直接从单例缓存中获取。

- `<x>` 处，如果从缓存中得到了 Bean 对象，则需要调用 `#getObjectForBeanInstance(Object beanInstance, String name, String beanName, RootBeanDefinition mbd)` 方法，对 Bean 进行实例化处理。因为，缓存中记录的是最原始的 Bean 状态，我们得到的不一定是我们**最终**想要的 Bean 。另外，FactoryBean 的用途如下：

  > From 《Spring 源码深度解析》P83 页
  >
  > 一般情况下，Spring 通过反射机制利用 bean 的 class 属性指定实现类来实例化 bean 。某些情况下，实例化 bean 过程比较复杂，如果按照传统的方式，则需要在 中提供大量的配置信息，配置方式的灵活性是受限的，这时采用编码的方式可能会得到一个简单的方案。Spring 为此提供了一个 FactoryBean 的工厂类接口，用户可以通过实现该接口定制实例化 bean 的逻辑。
  >
  > FactoryBean 接口对于 Spring 框架来说战友重要的地址，Spring 自身就提供了 70 多个 FactoryBean 的实现。它们隐藏了实例化一些复杂 bean 的细节，给上层应用带来了便利。

详细解析，见 [《【死磕 Spring】—— ﻿IoC 之加载 Bean：从单例缓存中获取单》](http://svip.iocoder.cn/Spring/IoC-get-Bean-getSingleton) 。

#### 2.3 原型模式依赖检查

对应代码段如下：

```
// AbstractBeanFactory.java

// Fail if we're already creating this bean instance:
// We're assumably within a circular reference.
// 因为 Spring 只解决单例模式下得循环依赖，在原型模式下如果存在循环依赖则会抛出异常。
if (isPrototypeCurrentlyInCreation(beanName)) {
	throw new BeanCurrentlyInCreationException(beanName);
}
```

Spring 只处理单例模式下得循环依赖，对于原型模式的循环依赖直接抛出异常。主要原因还是在于，**和 Spring 解决循环依赖的策略有关**。

- 对于**单例( Singleton )模式**， Spring 在创建 Bean 的时候并不是等 Bean 完全创建完成后才会将 Bean 添加至缓存中，而是不等 Bean 创建完成就会将创建 Bean 的 ObjectFactory 提早加入到缓存中，这样一旦下一个 Bean 创建的时候需要依赖 bean 时则直接使用 ObjectFactroy 。
- 但是**原型( Prototype )模式**，我们知道是没法使用缓存的，所以 Spring 对原型模式的循环依赖处理策略则是不处理。

详细解析，见 [《【死磕 Spring】—— ﻿IoC 之加载 Bean：parentBeanFactory 与依赖处理》](http://svip.iocoder.cn/Spring/IoC-get-Bean-parentBeanFactory-and-depend-on) 。

#### 2.4 从 parentBeanFactory 获取 Bean

对应代码段如下：

```
// AbstractBeanFactory.java

// 如果当前容器中没有找到，则从父类容器中加载
// Check if bean definition exists in this factory.
BeanFactory parentBeanFactory = getParentBeanFactory();
if (parentBeanFactory != null && !containsBeanDefinition(beanName)) {
    // Not found -> check parent.
    String nameToLookup = originalBeanName(name);
    // 如果，父类容器为 AbstractBeanFactory ，直接递归查找
    if (parentBeanFactory instanceof AbstractBeanFactory) {
        return ((AbstractBeanFactory) parentBeanFactory).doGetBean(
                nameToLookup, requiredType, args, typeCheckOnly);
    // 用明确的 args 从 parentBeanFactory 中，获取 Bean 对象
    } else if (args != null) {
        // Delegation to parent with explicit args.
        return (T) parentBeanFactory.getBean(nameToLookup, args);
    // 用明确的 requiredType 从 parentBeanFactory 中，获取 Bean 对象
    } else if (requiredType != null) {
        // No args -> delegate to standard getBean method.
        return parentBeanFactory.getBean(nameToLookup, requiredType);
    // 直接使用 nameToLookup 从 parentBeanFactory 获取 Bean 对象
    } else {
        return (T) parentBeanFactory.getBean(nameToLookup);
    }
}
```

- 如果**当前**容器缓存中没有相对应的 BeanDefinition 对象，则会尝试从父类工厂（`parentBeanFactory`）中加载，然后再去递归调用 `#getBean(...)` 方法。

详细解析，见 [《【死磕 Spring】—— ﻿IoC 之加载 Bean：parentBeanFactory 与依赖处理》](http://svip.iocoder.cn/Spring/IoC-get-Bean-parentBeanFactory-and-depend-on) 。

#### 2.5 指定的 Bean 标记为已经创建或即将创建

对应代码段如下：

```
// AbstractBeanFactory.java

// 如果不是仅仅做类型检查则是创建bean，这里需要记录
if (!typeCheckOnly) {
	markBeanAsCreated(beanName);
}
```

详细解析，见 [《【死磕 Spring】—— ﻿IoC 之加载 Bean：parentBeanFactory 与依赖处理》](http://svip.iocoder.cn/Spring/IoC-get-Bean-parentBeanFactory-and-depend-on) 。

#### 2.6 获取 BeanDefinition

对应代码段如下：

```
// AbstractBeanFactory.java

// 从容器中获取 beanName 相应的 GenericBeanDefinition 对象，并将其转换为 RootBeanDefinition 对象
final RootBeanDefinition mbd = getMergedLocalBeanDefinition(beanName);
// 检查给定的合并的 BeanDefinition
checkMergedBeanDefinition(mbd, beanName, args);
```

> 因为从 XML 配置文件中读取到的 Bean 信息是存储在GenericBeanDefinition 中的。但是，所有的 Bean 后续处理都是针对于 RootBeanDefinition 的，所以这里需要进行一个转换。
>
> 转换的同时，如果父类 bean 不为空的话，则会一并合并父类的属性。

详细解析，见 [《【死磕 Spring】—— ﻿IoC 之加载 Bean：parentBeanFactory 与依赖处理》](http://svip.iocoder.cn/Spring/IoC-get-Bean-parentBeanFactory-and-depend-on) 。

#### 2.7 依赖 Bean 处理

对应代码段如下：

```
// AbstractBeanFactory.java

// Guarantee initialization of beans that the current bean depends on.
// 处理所依赖的 bean
String[] dependsOn = mbd.getDependsOn();
if (dependsOn != null) {
    for (String dep : dependsOn) {
        // 若给定的依赖 bean 已经注册为依赖给定的 bean
        // 即循环依赖的情况，抛出 BeanCreationException 异常
        if (isDependent(beanName, dep)) {
            throw new BeanCreationException(mbd.getResourceDescription(), beanName,
                    "Circular depends-on relationship between '" + beanName + "' and '" + dep + "'");
        }
        // 缓存依赖调用 TODO 芋艿
        registerDependentBean(dep, beanName);
        try {
            // 递归处理依赖 Bean
            getBean(dep);
        } catch (NoSuchBeanDefinitionException ex) {
            throw new BeanCreationException(mbd.getResourceDescription(), beanName,
                    "'" + beanName + "' depends on missing bean '" + dep + "'", ex);
        }
    }
}
```

- 每个 Bean 都不是单独工作的，它会依赖其他 Bean，其他 Bean 也会依赖它。
- 对于依赖的 Bean ，它会优先加载，所以，在 Spring 的加载顺序中，在初始化某一个 Bean 的时候，首先会初始化这个 Bean 的依赖。

详细解析，见 [《【死磕 Spring】—— ﻿IoC 之加载 Bean：parentBeanFactory 与依赖处理》](http://svip.iocoder.cn/Spring/IoC-get-Bean-parentBeanFactory-and-depend-on) 。

#### 2.8 不同作用域的 Bean 实例化

对应代码段如下：

```
// AbstractBeanFactory.java

// bean 实例化
// Create bean instance.
if (mbd.isSingleton()) { // 单例模式
    sharedInstance = getSingleton(beanName, () -> {
        try {
            return createBean(beanName, mbd, args);
        }
        catch (BeansException ex) {
            // Explicitly remove instance from singleton cache: It might have been put there
            // eagerly by the creation process, to allow for circular reference resolution.
            // Also remove any beans that received a temporary reference to the bean.
            // 显式从单例缓存中删除 Bean 实例
            // 因为单例模式下为了解决循环依赖，可能他已经存在了，所以销毁它。 TODO 芋艿
            destroySingleton(beanName);
            throw ex;
        }
    });
    bean = getObjectForBeanInstance(sharedInstance, name, beanName, mbd);
} else if (mbd.isPrototype()) { // 原型模式
    // It's a prototype -> create a new instance.
    Object prototypeInstance;
    try {
        beforePrototypeCreation(beanName);
        prototypeInstance = createBean(beanName, mbd, args);
    } finally {
        afterPrototypeCreation(beanName);
    }
    bean = getObjectForBeanInstance(prototypeInstance, name, beanName, mbd);
} else {
    // 从指定的 scope 下创建 bean
    String scopeName = mbd.getScope();
    final Scope scope = this.scopes.get(scopeName);
    if (scope == null) {
        throw new IllegalStateException("No Scope registered for scope name '" + scopeName + "'");
    }try {
        Object scopedInstance = scope.get(beanName, () -> {
            beforePrototypeCreation(beanName);
            try {
                return createBean(beanName, mbd, args);
            } finally {
                afterPrototypeCreation(beanName);
            }
        });
        bean = getObjectForBeanInstance(scopedInstance, name, beanName, mbd);
    } catch (IllegalStateException ex) {
        throw new BeanCreationException(beanName,
                "Scope '" + scopeName + "' is not active for the current thread; consider " +
                "defining a scoped proxy for this bean if you intend to refer to it from a singleton",
                ex);
    }
}
```

- Spring Bean 的作用域默认为 singleton 。当然，还有其他作用域，如 prototype、request、session 等。
- 不同的作用域会有不同的初始化策略。

详细解析，见 [《【死磕 Spring】—— IoC 之加载 Bean：分析各 scope 的 Bean 创建》](http://svip.iocoder.cn/Spring/IoC-get-Bean-different-scope) 。

#### 2.9 类型转换

对应代码段如下：

```
// AbstractBeanFactory.java

// 检查需要的类型是否符合 bean 的实际类型
// Check if required type matches the type of the actual bean instance.
if (requiredType != null && !requiredType.isInstance(bean)) {
    try {
        // 执行转换
        T convertedBean = getTypeConverter().convertIfNecessary(bean, requiredType);
        // 转换失败，抛出 BeanNotOfRequiredTypeException 异常
        if (convertedBean == null) {
            throw new BeanNotOfRequiredTypeException(name, requiredType, bean.getClass());
        }
        return convertedBean;
    } catch (TypeMismatchException ex) {
        if (logger.isTraceEnabled()) {
            logger.trace("Failed to convert bean '" + name + "' to required type '" +
                    ClassUtils.getQualifiedName(requiredType) + "'", ex);
        }
        throw new BeanNotOfRequiredTypeException(name, requiredType, bean.getClass());
    }
}
```

- 在调用 `#doGetBean(...)` 方法时，`有一个 requiredTyp`e 参数。该参数的功能就是将返回的 Bean 转换为 `requiredType` 类型。
- 当然就一般而言，我们是不需要进行类型转换的，也就是 `requiredType` 为空（比如 `#getBean(String name)` 方法）。但有，可能会存在这种情况，比如我们返回的 Bean 类型为 String ，我们在使用的时候需要将其转换为 Integer，那么这个时候 `requiredType` 就有用武之地了。**当然我们一般是不需要这样做的**。

详细解析，见 TODO

## 3. 小结

至此 `BeanFactory#getBean(...)` 方法的过程讲解完了。后续将会对该过程进行拆分，更加详细的说明，弄清楚其中的来龙去脉，所以这篇博客只能算是 Spring Bean 加载过程的一个概览。拆分主要是分为三个部分：

1. 分析从缓存中获取单例 Bean ，以及对 Bean 的实例中获取对象。
2. 如果从单例缓存中获取 Bean ，Spring 是怎么加载的呢？所以第二部分是分析 Bean 加载，以及 Bean 的依赖处理。
3. Bean 已经加载了，依赖也处理完毕了，第三部分则分析各个作用域的 Bean 初始化过程。

> 老艿艿：再推荐几篇不错的 Spring Bean 加载的过程，写的不错的文章：
>
> - zhanglbjames [《Spring-IoC-getBean源码解析》](https://www.jianshu.com/p/a6acfceb42fb)
> - glmapper [《Spring源码系列：依赖注入（一）getBean》](https://juejin.im/post/5a7532746fb9a0633a70d57f)
> - 是Guava不是瓜娃 [《Spring原理与源码分析系列（五）- Spring IoC源码分析（下）》](https://blog.csdn.net/noaman_wgs/article/details/79138883)

# 2、从单例缓存中获取单例 Bean

加载 Bean 的第一个步骤，从缓存中获取 Bean 。代码片段如下：

```
// AbstractBeanFactory.java

// 从缓存中或者实例工厂中获取 Bean 对象
// Eagerly check singleton cache for manually registered singletons.
Object sharedInstance = getSingleton(beanName);
if (sharedInstance != null && args == null) {
	if (logger.isTraceEnabled()) {
		if (isSingletonCurrentlyInCreation(beanName)) {
			logger.trace("Returning eagerly cached instance of singleton bean '" + beanName +
					"' that is not fully initialized yet - a consequence of a circular reference");
		} else {
			logger.trace("Returning cached instance of singleton bean '" + beanName + "'");
		}
	}
	// 完成 FactoryBean 的相关处理，并用来获取 FactoryBean 的处理结果
	bean = getObjectForBeanInstance(sharedInstance, name, beanName, null);
}
```

调用 `#getSingleton(String beanName)` 方法，从缓存中获取 Bean 。

## 1. getSingleton

在上篇博客 [《【死磕 Spring】—— IoC 之开启 Bean 的加载》](http://svip.iocoder.cn/Spring/IoC-get-Bean-begin) 中提到过，Spring 对单例模式的 bean 只会创建一次。后续，如果再获取该 Bean ，则是直接从单例缓存中获取，该过程就体现在 `#getSingleton(String beanName)` 方法中。代码如下：

```
// DefaultSingletonBeanRegistry.java

@Override
@Nullable
public Object getSingleton(String beanName) {
    return getSingleton(beanName, true);
}

@Nullable
protected Object getSingleton(String beanName, boolean allowEarlyReference) {
    // 从单例缓冲中加载 bean
    Object singletonObject = this.singletonObjects.get(beanName);
    // 缓存中的 bean 为空，且当前 bean 正在创建
    if (singletonObject == null && isSingletonCurrentlyInCreation(beanName)) {
        // 加锁
        synchronized (this.singletonObjects) {
            // 从 earlySingletonObjects 获取
            singletonObject = this.earlySingletonObjects.get(beanName);
            // earlySingletonObjects 中没有，且允许提前创建
            if (singletonObject == null && allowEarlyReference) {
                // 从 singletonFactories 中获取对应的 ObjectFactory
                ObjectFactory<?> singletonFactory = this.singletonFactories.get(beanName);
                if (singletonFactory != null) {
                    // 获得 bean
                    singletonObject = singletonFactory.getObject();
                    // 添加 bean 到 earlySingletonObjects 中
                    this.earlySingletonObjects.put(beanName, singletonObject);
                    // 从 singletonFactories 中移除对应的 ObjectFactory
                    this.singletonFactories.remove(beanName);
                }
            }
        }
    }
    return singletonObject;
}
```

这段代码非常简单，保持淡定，过程如下：

- 第一步，从 `singletonObjects` 中，获取 Bean 对象。

- 第二步，若为空且当前 bean 正在创建中，则从 `earlySingletonObjects` 中获取 Bean 对象。

- 第三步，若为空且允许提前创建，则从 `singletonFactories` 中获取相应的 ObjectFactory 对象。若不为空，则调用其 `ObjectFactory#getObject(String name)` 方法，创建 Bean 对象，然后将其加入到 `earlySingletonObjects` ，然后从 `singletonFactories` 删除。

- 总体逻辑，就是根据 `beanName` 依次检测这三个 Map，若为空，从下一个，否则返回。这三个 Map 存放的都有各自的功能，代码如下：

  ```
  // DefaultSingletonBeanRegistry.java
  
  /**
   * Cache of singleton objects: bean name to bean instance.
   *
   * 存放的是单例 bean 的映射。
   *
   * 对应关系为 bean name --> bean instance
   */
  private final Map<String, Object> singletonObjects = new ConcurrentHashMap<>(256);
  
  /**
   * Cache of singleton factories: bean name to ObjectFactory.
   *
   * 存放的是 ObjectFactory，可以理解为创建单例 bean 的 factory 。
   *
   * 对应关系是 bean name --> ObjectFactory
   **/
  private final Map<String, ObjectFactory<?>> singletonFactories = new HashMap<>(16);
  
  /**
   * Cache of early singleton objects: bean name to bean instance.
   *
   * 存放的是早期的 bean，对应关系也是 bean name --> bean instance。
   *
   * 它与 {@link #singletonFactories} 区别在于 earlySingletonObjects 中存放的 bean 不一定是完整。
   *
   * 从 {@link #getSingleton(String)} 方法中，我们可以了解，bean 在创建过程中就已经加入到 earlySingletonObjects 中了。
   * 所以当在 bean 的创建过程中，就可以通过 getBean() 方法获取。
   *
   * 这个 Map 也是【循环依赖】的关键所在。
   */
  private final Map<String, Object> earlySingletonObjects = new HashMap<>(16);
  ```

#### 1.1 isSingletonCurrentlyInCreation

在上面代码中，还有一个非常重要的检测方法 `#isSingletonCurrentlyInCreation(String beanName)` 方法，该方法用于判断该 `beanName` 对应的 Bean **是否在创建过程**中，注意这个过程讲的是整个工厂中。代码如下：

```
// DefaultSingletonBeanRegistry.java

/**
 * Names of beans that are currently in creation.
 *
 * 正在创建中的单例 Bean 的名字的集合
 */
private final Set<String> singletonsCurrentlyInCreation =
		Collections.newSetFromMap(new ConcurrentHashMap<>(16));
```

从这段代码中，我们可以预测，在 Bean 创建过程中都会将其加入到 `singletonsCurrentlyInCreation` 集合中。具体是在什么时候加的，我们后面分析。

## 2. getObjectForBeanInstance

到这里从缓存中获取 bean 的过程已经分析完毕了，我们再看开篇的代码段，从缓存中获取 Bean 后，若其不为 `null` 且 `args` 为空，则会调用 `#getObjectForBeanInstance(Object beanInstance, String name, String beanName,RootBeanDefinition mbd)` 方法，进行处理。

- 为什么会有这么一段呢？因为我们从缓存中获取的 bean 是**最原始**的 Bean ，并不一定使我们最终想要的 Bean 。
- 怎么办呢？调用 `#getObjectForBeanInstance(...)` 方法，进行处理，该方法的定义为获取给定 Bean 实例的对象，该对象要么是 bean 实例本身，要么就是 FactoryBean 创建的 Bean 对象。

代码如下：

```
// AbstractBeanFactory.java

protected Object getObjectForBeanInstance(
        Object beanInstance, String name, String beanName, @Nullable RootBeanDefinition mbd) {
    // <1> 若为工厂类引用（name 以 & 开头）
    // Don't let calling code try to dereference the factory if the bean isn't a factory.
    if (BeanFactoryUtils.isFactoryDereference(name)) {
        // 如果是 NullBean，则直接返回
        if (beanInstance instanceof NullBean) {
            return beanInstance;
        }
        // 如果 beanInstance 不是 FactoryBean 类型，则抛出异常
        if (!(beanInstance instanceof FactoryBean)) {
            throw new BeanIsNotAFactoryException(transformedBeanName(name), beanInstance.getClass());
        }
    }

    // 到这里我们就有了一个 Bean 实例，当然该实例可能是会是是一个正常的 bean 又或者是一个 FactoryBean
    // 如果是 FactoryBean，我我们则创建该 Bean
    // Now we have the bean instance, which may be a normal bean or a FactoryBean.
    // If it's a FactoryBean, we use it to create a bean instance, unless the
    // caller actually wants a reference to the factory.
    if (!(beanInstance instanceof FactoryBean) || BeanFactoryUtils.isFactoryDereference(name)) {
        return beanInstance;
    }

    Object object = null;
    // <3> 若 BeanDefinition 为 null，则从缓存中加载 Bean 对象
    if (mbd == null) {
        object = getCachedObjectForFactoryBean(beanName);
    }
    // 若 object 依然为空，则可以确认，beanInstance 一定是 FactoryBean 。从而，使用 FactoryBean 获得 Bean 对象
    if (object == null) {
        // Return bean instance from factory.
        FactoryBean<?> factory = (FactoryBean<?>) beanInstance;
        // containsBeanDefinition 检测 beanDefinitionMap 中也就是在所有已经加载的类中
        // 检测是否定义 beanName
        // Caches object obtained from FactoryBean if it is a singleton.
        if (mbd == null && containsBeanDefinition(beanName)) {
            // 将存储 XML 配置文件的 GenericBeanDefinition 转换为 RootBeanDefinition，
            // 如果指定 BeanName 是子 Bean 的话同时会合并父类的相关属性
            mbd = getMergedLocalBeanDefinition(beanName);
        }
        // 是否是用户定义的，而不是应用程序本身定义的
        boolean synthetic = (mbd != null && mbd.isSynthetic());
        // 核心处理方法，使用 FactoryBean 获得 Bean 对象
        object = getObjectFromFactoryBean(factory, beanName, !synthetic);
    }
    return object;
}
```

该方法主要是进行检测工作的，主要如下：

- `<1>` 处，若 `name` 为工厂相关的（以 & 开头），且 `beanInstance` 为 NullBean 类型则直接返回，如果 `beanInstance` 不为 FactoryBean 类型则抛出 BeanIsNotAFactoryException 异常。这里主要是**校验** `beanInstance` 的**正确性**。

- `<2>` 处，如果 `beanInstance` 不为 FactoryBean 类型或者 `name` 也不是与工厂相关的，则直接返回 `beanInstance` 这个 Bean 对象。**这里主要是对非 FactoryBean 类型处理**。

- `<3>` 处，如果 BeanDefinition 为空，则从 `factoryBeanObjectCache` 中加载 Bean 对象。如果还是空，则可以断定 `beanInstance` 一定是 FactoryBean 类型，则委托 `#getObjectFromFactoryBean(FactoryBean<?> factory, String beanName, boolean shouldPostProcess)` 方法，进行处理，**使用 FactoryBean 获得 Bean 对象**。

  > 老艿艿：所以实际上，`#getObjectForBeanInstance(...)` 方法的**重心**，就是使用 FactoryBean 对象，获得( 或者创建 )其 Bean 对象，即调用 `#getObjectFromFactoryBean(...)` 方法。

#### 2.1 getObjectFromFactoryBean

从上面可以看出， `#getObjectForBeanInstance(Object beanInstance, String name, String beanName,RootBeanDefinition mbd)` 方法，分成两种情况：

- 第一种，当该实例对象为**非 FactoryBean 类型**，**直接**返回给定的 Bean 实例对象 `beanInstance` 。
- 第二种，当该实例对象为**FactoryBean 类型**，从 FactoryBean ( `beanInstance` ) 中，获取 Bean 实例对象。

第二种，通过 `#getObjectFromFactoryBean(FactoryBean<?> factory, String beanName, boolean shouldPostProcess)` 方法来实现。代码如下：

```
// FactoryBeanRegistrySupport.java

/**
 * Cache of singleton objects created by FactoryBeans: FactoryBean name to object.
 *
 * 缓存 FactoryBean 创建的单例 Bean 对象的映射
 * beanName ===> Bean 对象
 */
private final Map<String, Object> factoryBeanObjectCache = new ConcurrentHashMap<>(16);

protected Object getObjectFromFactoryBean(FactoryBean<?> factory, String beanName, boolean shouldPostProcess) {
    // <1> 为单例模式且缓存中存在
    if (factory.isSingleton() && containsSingleton(beanName)) {
        synchronized (getSingletonMutex()) { // <1.1> 单例锁
            // <1.2> 从缓存中获取指定的 factoryBean
            Object object = this.factoryBeanObjectCache.get(beanName);
            if (object == null) {
                // 为空，则从 FactoryBean 中获取对象
                object = doGetObjectFromFactoryBean(factory, beanName);
                // 从缓存中获取
                // TODO 芋艿，具体原因
                // Only post-process and store if not put there already during getObject() call above
                // (e.g. because of circular reference processing triggered by custom getBean calls)
                Object alreadyThere = this.factoryBeanObjectCache.get(beanName);
                if (alreadyThere != null) {
                    object = alreadyThere;
                } else {
                    // <1.3> 需要后续处理
                    if (shouldPostProcess) {
                        // 若该 Bean 处于创建中，则返回非处理对象，而不是存储它
                        if (isSingletonCurrentlyInCreation(beanName)) {
                            // Temporarily return non-post-processed object, not storing it yet..
                            return object;
                        }
                        // 单例 Bean 的前置处理
                        beforeSingletonCreation(beanName);
                        try {
                            // 对从 FactoryBean 获取的对象进行后处理
                            // 生成的对象将暴露给 bean 引用
                            object = postProcessObjectFromFactoryBean(object, beanName);
                        } catch (Throwable ex) {
                            throw new BeanCreationException(beanName,
                                    "Post-processing of FactoryBean's singleton object failed", ex);
                        } finally {
                            // 单例 Bean 的后置处理
                            afterSingletonCreation(beanName);
                        }
                    }
                    // <1.4> 添加到 factoryBeanObjectCache 中，进行缓存
                    if (containsSingleton(beanName)) {
                        this.factoryBeanObjectCache.put(beanName, object);
                    }
                }
            }
            return object;
        }
    // <2>
    } else {
        // 为空，则从 FactoryBean 中获取对象
        Object object = doGetObjectFromFactoryBean(factory, beanName);
        // 需要后续处理
        if (shouldPostProcess) {
            try {
                // 对从 FactoryBean 获取的对象进行后处理
                // 生成的对象将暴露给 bean 引用
                object = postProcessObjectFromFactoryBean(object, beanName);
            }
            catch (Throwable ex) {
                throw new BeanCreationException(beanName, "Post-processing of FactoryBean's object failed", ex);
            }
        }
        return object;
    }
}
```

主要流程如下：

- 若为单例且单例 Bean 缓存中存在 `beanName` ，则 `<1>` 进行后续处理（跳转到下一步），否则，则 `<2>` 从 FactoryBean 中获取 Bean 实例对象。

- `<1.1>` 首先，获取锁。其实我们在前面篇幅中发现了大量的同步锁，锁住的对象都是 `this.singletonObjects`，主要是因为在单例模式中必须要**保证全局唯一**。代码如下：

  ```
  // DefaultSingletonBeanRegistry.java
  
  /**
   * Cache of singleton objects: bean name to bean instance.
   *
   * 存放的是单例 bean 的映射。
   *
   * 对应关系为 bean name --> bean instance
   */
  private final Map<String, Object> singletonObjects = new ConcurrentHashMap<>(256);
  
  public final Object getSingletonMutex() {
  	return this.singletonObjects;
  }
  ```

- `<1.2>` 然后，从 `factoryBeanObjectCache` 缓存中获取实例对象 `object` 。若 `object` 为空，则调用 `#doGetObjectFromFactoryBean(FactoryBean<?> factory, String beanName)` 方法，从 FactoryBean 获取对象，其实内部就是调用 `FactoryBean#getObject()` 方法。代码如下：

  ```
  private Object doGetObjectFromFactoryBean(final FactoryBean<?> factory, final String beanName)
      throws BeanCreationException {
      Object object;
      try {
          // 需要权限验证
          if (System.getSecurityManager() != null) {
              AccessControlContext acc = getAccessControlContext();
              try {
                  // <x> 从 FactoryBean 中，获得 Bean 对象
                  object = AccessController.doPrivileged((PrivilegedExceptionAction<Object>) factory::getObject, acc);
              } catch (PrivilegedActionException pae) {
                  throw pae.getException();
              }
          } else {
              // <x> 从 FactoryBean 中，获得 Bean 对象
              object = factory.getObject();
          }
      } catch (FactoryBeanNotInitializedException ex) {
          throw new BeanCurrentlyInCreationException(beanName, ex.toString());
      } catch (Throwable ex) {
          throw new BeanCreationException(beanName, "FactoryBean threw exception on object creation", ex);
      }
      // Do not accept a null value for a FactoryBean that's not fully
      // initialized yet: Many FactoryBeans just return null then.
      if (object == null) {
          if (isSingletonCurrentlyInCreation(beanName)) {
              throw new BeanCurrentlyInCreationException(
                      beanName, "FactoryBean which is currently in creation returned null from getObject");
          }
          object = new NullBean();
      }
      return object;
  }
  ```

  - 在 `<x>` 处，可以看到，调用 `FactoryBean#getObject()` 方法，获取 Bean 对象。

- `<1.3>` 如果需要后续处理( `shouldPostProcess = true` )，则进行进一步处理，步骤如下：

  - 若该 Bean 处于创建中（`#isSingletonCurrentlyInCreation(String beanName)` 方法返回 `true` ），则返回**非处理的 Bean 对象**，而不是存储它。
  - 调用 `#beforeSingletonCreation(String beanName)` 方法，进行创建之前的处理。默认实现将该 Bean 标志为当前创建的。
  - 调用 `#postProcessObjectFromFactoryBean(Object object, String beanName)` 方法，对从 FactoryBean 获取的 Bean 实例对象进行后置处理。详细解析，见 [「2.3 postProcessObjectFromFactoryBean」](http://svip.iocoder.cn/Spring/IoC-get-Bean-getSingleton/#) 。
  - 调用 `#afterSingletonCreation(String beanName)` 方法，进行创建 Bean 之后的处理，默认实现是将该 bean 标记为不再在创建中。

- `<1.4>` 最后，加入到 `factoryBeanObjectCache` 缓存中。

该方法应该就是创建 Bean 实例对象中的核心方法之一了。这里我们关注三个方法：

- `#beforeSingletonCreation(String beanName)`
- `#afterSingletonCreation(String beanName)`
- `#postProcessObjectFromFactoryBean(Object object, String beanName)`

#### 2.2 isSingletonCurrentlyInCreation

可能有小伙伴觉得前面两个方法不是很重要，LZ 可以肯定告诉你，这两方法是非常重要的操作，因为**他们记录着 Bean 的加载状态，是检测当前 Bean 是否处于创建中的关键之处，对解决 Bean 循环依赖起着关键作用**。

- `#beforeSingletonCreation(String beanName)` 方法，用于添加标志，当前 bean **正处于**创建中
- `#afterSingletonCreation(String beanName)` 方法，用于移除标记，当前 Bean **不处于**创建中。

其实在这篇博客刚刚开始就已经提到了， `#isSingletonCurrentlyInCreation(String beanName)` 方法，是用于检测当前 Bean 是否处于创建之中。代码如下：

```
// DefaultSingletonBeanRegistry.java

/**
 * Names of beans that are currently in creation.
 *
 * 正在创建中的单例 Bean 的名字的集合
 */
private final Set<String> singletonsCurrentlyInCreation =
        Collections.newSetFromMap(new ConcurrentHashMap<>(16));
```

- 是根据 `singletonsCurrentlyInCreation` 集合中是否包含了 `beanName` 。

###### 2.2.1 beforeSingletonCreation

集合的元素，则一定是在 `#beforeSingletonCreation(String beanName)` 方法中添加的。代码如下：

```
// DefaultSingletonBeanRegistry.java

protected void beforeSingletonCreation(String beanName) {
	if (!this.inCreationCheckExclusions.contains(beanName)
            && !this.singletonsCurrentlyInCreation.add(beanName)) { // 添加
		throw new BeanCurrentlyInCreationException(beanName); // 如果添加失败，则抛出 BeanCurrentlyInCreationException 异常。
	}
}
```

###### 2.2.2 afterSingletonCreation

`#afterSingletonCreation(String beanName)` 方法，为移除，则一定就是对 `singletonsCurrentlyInCreation` 集合 remove 了。代码如下：

```
// DefaultSingletonBeanRegistry.java

protected void afterSingletonCreation(String beanName) {
	if (!this.inCreationCheckExclusions.contains(beanName) &&
            !this.singletonsCurrentlyInCreation.remove(beanName)) { // 移除
	    // 如果移除失败，则抛出 IllegalStateException 异常
		throw new IllegalStateException("Singleton '" + beanName + "' isn't currently in creation");
	}
}
```

#### 2.3 postProcessObjectFromFactoryBean

`postProcessObjectFromFactoryBean(Object object, String beanName)` 方法，对从 FactoryBean 处获取的 Bean 实例对象进行后置处理。其默认实现是直接返回 object 对象，不做任何处理。代码如下：

```
// DefaultSingletonBeanRegistry.java

protected Object postProcessObjectFromFactoryBean(Object object, String beanName) throws BeansException {
	return object;
}
```

###### 2.3.1

当然，子类可以重写，例如应用后处理器。`org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory` 抽象类，对其提供了实现，代码如下：

```
// AbstractAutowireCapableBeanFactory.java

protected Object postProcessObjectFromFactoryBean(Object object, String beanName) {
	return applyBeanPostProcessorsAfterInitialization(object, beanName);
}
```

- 该方法的定义为：对所有的 `{@code postProcessAfterInitialization}` 进行回调注册 BeanPostProcessors ，让他们能够后期处理从 FactoryBean 中获取的对象。下面是具体实现：

  ```
  // AbstractAutowireCapableBeanFactory.java
  
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

  - 对于后置处理器，这里我们不做过多阐述，后面会专门的博文进行详细介绍，这里我们只需要记住一点：尽可能保证所有 bean 初始化后都会调用注册的 `BeanPostProcessor#postProcessAfterInitialization(Object bean, String beanName)` 方法进行处理，在实际开发过程中大可以针对此特性设计自己的业务逻辑。

## 3. 小结

至此，从缓存中获取 Bean 对象过程已经分析完毕了。

下面两篇博客分析，如果从单例缓存中没有获取到单例 Bean ，则 Spring 是如何处理的。

# 3、parentBeanFactory 与依赖处理

如果从单例缓存中没有获取到单例 Bean 对象，则说明两种两种情况：

1. 该 Bean 的 scope 不是 singleton
2. 该 Bean 的 scope 是 singleton ，但是没有初始化完成。

针对这两种情况，Spring 是如何处理的呢？统一加载并完成初始化！这部分内容的篇幅较长，拆分为两部分：

- 第一部分，主要是一些检测、parentBeanFactory 以及依赖处理。
- 第二部分则是各个 scope 的初始化。

代码如下：

```
// AbstractBeanFactory.java
//protected <T> T doGetBean(final String name, final Class<T> requiredType, final Object[] args, boolean typeCheckOnly)

// ... 省略很多代码

// Fail if we're already creating this bean instance:
// We're assumably within a circular reference.
// <3> 因为 Spring 只解决单例模式下得循环依赖，在原型模式下如果存在循环依赖则会抛出异常。
if (isPrototypeCurrentlyInCreation(beanName)) {
    throw new BeanCurrentlyInCreationException(beanName);
}

// <4> 如果容器中没有找到，则从父类容器中加载
// Check if bean definition exists in this factory.
BeanFactory parentBeanFactory = getParentBeanFactory();
if (parentBeanFactory != null && !containsBeanDefinition(beanName)) {
    // Not found -> check parent.
    String nameToLookup = originalBeanName(name);
    if (parentBeanFactory instanceof AbstractBeanFactory) {
        return ((AbstractBeanFactory) parentBeanFactory).doGetBean(
                nameToLookup, requiredType, args, typeCheckOnly);
    } else if (args != null) {
        // Delegation to parent with explicit args.
        return (T) parentBeanFactory.getBean(nameToLookup, args);
    } else if (requiredType != null) {
        // No args -> delegate to standard getBean method.
        return parentBeanFactory.getBean(nameToLookup, requiredType);
    } else {
        return (T) parentBeanFactory.getBean(nameToLookup);
    }
}

// <5> 如果不是仅仅做类型检查则是创建bean，这里需要记录
if (!typeCheckOnly) {
    markBeanAsCreated(beanName);
}

try {
    // <6> 从容器中获取 beanName 相应的 GenericBeanDefinition 对象，并将其转换为 RootBeanDefinition 对象
    final RootBeanDefinition mbd = getMergedLocalBeanDefinition(beanName);
    // 检查给定的合并的 BeanDefinition
    checkMergedBeanDefinition(mbd, beanName, args);

    // Guarantee initialization of beans that the current bean depends on.
    // <7> 处理所依赖的 bean
    String[] dependsOn = mbd.getDependsOn();
    if (dependsOn != null) {
        for (String dep : dependsOn) {
            // 若给定的依赖 bean 已经注册为依赖给定的 bean
            // 循环依赖的情况
            if (isDependent(beanName, dep)) {
                throw new BeanCreationException(mbd.getResourceDescription(), beanName,
                        "Circular depends-on relationship between '" + beanName + "' and '" + dep + "'");
            }
            // 缓存依赖调用
            registerDependentBean(dep, beanName);
            try {
                getBean(dep);
            } catch (NoSuchBeanDefinitionException ex) {
                throw new BeanCreationException(mbd.getResourceDescription(), beanName,
                        "'" + beanName + "' depends on missing bean '" + dep + "'", ex);
            }
        }
    }
}

// ... 省略很多代码
```

这段代码主要处理如下几个部分：

- ```
  <3>
  ```

   

  处，检测。若当前 Bean 在创建，则抛出 BeanCurrentlyInCreationException 异常。

  - 详细解析，见 [「1. 检测」](http://svip.iocoder.cn/Spring/IoC-get-Bean-parentBeanFactory-and-depend-on/#) 。

- ```
  <4>
  ```

   

  处，如果 beanDefinitionMap 中不存在 beanName 的 BeanDefinition（即在 Spring bean 初始化过程中没有加载），则尝试从 parentBeanFactory 中加载。

  - 详细解析，见 [「2. 检查父类 BeanFactory」](http://svip.iocoder.cn/Spring/IoC-get-Bean-parentBeanFactory-and-depend-on/#) 。

- ```
  <5>
  ```

   

  处，判断是否为类型检查。

  - 详细解析，见 [「3. 类型检查」](http://svip.iocoder.cn/Spring/IoC-get-Bean-parentBeanFactory-and-depend-on/#) 。

- ```
  <6>
  ```

   

  处，从

   

  ```
  mergedBeanDefinitions
  ```

   

  中获取

   

  ```
  beanName
  ```

   

  对应的 RootBeanDefinition 对象。如果这个 BeanDefinition 是子 Bean 的话，则会合并父类的相关属性。

  - 详细解析，见 [「4. 获取 RootBeanDefinition」](http://svip.iocoder.cn/Spring/IoC-get-Bean-parentBeanFactory-and-depend-on/#) 。

- ```
  <7>
  ```

   

  处，依赖处理。

  - 详细解析，见 [「5. 处理依赖」](http://svip.iocoder.cn/Spring/IoC-get-Bean-parentBeanFactory-and-depend-on/#) 。

## 1. 检测

在前面就提过，Spring 只解决单例模式下的循环依赖，对于原型模式的循环依赖则是抛出 BeanCurrentlyInCreationException 异常，所以首先检查该 `beanName` 是否处于原型模式下的循环依赖。如下：

```
// AbstractBeanFactory.java

if (isPrototypeCurrentlyInCreation(beanName)) {
    throw new BeanCurrentlyInCreationException(beanName);
}
```

- 调用 `#isPrototypeCurrentlyInCreation(String beanName)` 方法，判断当前 Bean 是否正在创建。代码如下：

  ```
  // AbstractBeanFactory.java
  
  protected boolean isPrototypeCurrentlyInCreation(String beanName) {
  	Object curVal = this.prototypesCurrentlyInCreation.get();
  	return (curVal != null &&
  			(curVal.equals(beanName)  // 相等
                      || (curVal instanceof Set && ((Set<?>) curVal).contains(beanName)))); // 包含
  }
  ```

  - 其实检测逻辑和单例模式一样，一个“集合”存放着正在创建的 Bean ，从该集合中进行判断即可，只不过单例模式的“集合”为 Set ，而原型模式的则是 ThreadLocal 。`prototypesCurrentlyInCreation` 定义如下：

    ```
    // AbstractBeanFactory.java
    
    /** Names of beans that are currently in creation. */
    private final ThreadLocal<Object> prototypesCurrentlyInCreation =
    		new NamedThreadLocal<>("Prototype beans currently in creation");
    ```

## 2. 检查父类 BeanFactory

若 `#containsBeanDefinition(String beanName)` 方法中不存在 `beanName` 相对应的 BeanDefinition 对象时，则从 `parentBeanFactory` 中获取。代码如下：

```
// AbstractBeanFactory.java

// 获取 parentBeanFactory
BeanFactory parentBeanFactory = getParentBeanFactory();
// parentBeanFactory 不为空且 beanDefinitionMap 中不存该 name 的 BeanDefinition
if (parentBeanFactory != null && !containsBeanDefinition(beanName)) {
    // 确定原始 beanName
    String nameToLookup = originalBeanName(name);
    // 若为 AbstractBeanFactory 类型，委托父类处理
    if (parentBeanFactory instanceof AbstractBeanFactory) {
        return ((AbstractBeanFactory) parentBeanFactory).doGetBean(
                nameToLookup, requiredType, args, typeCheckOnly);
    } else if (args != null) {
        // 委托给构造函数 getBean() 处理
        return (T) parentBeanFactory.getBean(nameToLookup, args);
    } else {
        // 没有 args，委托给标准的 getBean() 处理
        return parentBeanFactory.getBean(nameToLookup, requiredType);
    }
}
```

- 整个过程较为简单，都是委托 `parentBeanFactory` 的 `#getBean(...)` 方法来进行处理，只不过在获取之前对 `breanName` 进行简单的处理，主要是想获取原始的 `beanName` 。代码如下：

  ```
  // AbstractBeanFactory.java
  
  protected String originalBeanName(String name) {
  	String beanName = transformedBeanName(name); // <1>
  	if (name.startsWith(FACTORY_BEAN_PREFIX)) { // <2>
  		beanName = FACTORY_BEAN_PREFIX + beanName;
  	}
  	return beanName;
  }
  ```

  - `<1>` 处，`#transformedBeanName(String name)` 方法，是对 `name` 进行转换，获取真正的 beanName 。在 [《【死磕 Spring】—— IoC 之开启 Bean 的加载》](http://svip.iocoder.cn/Spring/IoC-get-Bean-begin) 中，已经有详细解析。
  - `<2>` 处，如果 `name` 是以 `“&”` 开头的，则加上 `“&”` ，因为在 `#transformedBeanName(String name)` 方法，将 `“&”` 去掉了，这里**补上**。

## 3. 类型检查

方法参数 `typeCheckOnly` ，是用来判断调用 `#getBean(...)` 方法时，表示是否为**仅仅**进行类型检查获取 Bean 对象。如果不是仅仅做类型检查，而是创建 Bean 对象，则需要调用 `#markBeanAsCreated(String beanName)` 方法，进行记录。代码如下：

```
// AbstractBeanFactory.java

/**
 *  Names of beans that have already been created at least once.
 *
 *  已创建 Bean 的名字集合
 */
private final Set<String> alreadyCreated = Collections.newSetFromMap(new ConcurrentHashMap<>(256));

protected void markBeanAsCreated(String beanName) {
    // 没有创建
    if (!this.alreadyCreated.contains(beanName)) {
        // 加上全局锁
        synchronized (this.mergedBeanDefinitions) {
            // 再次检查一次：DCL 双检查模式
            if (!this.alreadyCreated.contains(beanName)) {
                // Let the bean definition get re-merged now that we're actually creating
                // the bean... just in case some of its metadata changed in the meantime.
                // 从 mergedBeanDefinitions 中删除 beanName，并在下次访问时重新创建它。
                clearMergedBeanDefinition(beanName);
                // 添加到已创建 bean 集合中
                this.alreadyCreated.add(beanName);
            }
        }
    }
}

protected void clearMergedBeanDefinition(String beanName) {
    this.mergedBeanDefinitions.remove(beanName);
}
```

## 4. 获取 RootBeanDefinition

```
// AbstractBeanFactory.java

// 从容器中获取 beanName 相应的 GenericBeanDefinition 对象，并将其转换为 RootBeanDefinition 对象
final RootBeanDefinition mbd = getMergedLocalBeanDefinition(beanName);
// 检查给定的合并的 BeanDefinition
checkMergedBeanDefinition(mbd, beanName, args);
```

- 调用 `#getMergedLocalBeanDefinition(String beanName)` 方法，获取相对应的 BeanDefinition 对象。代码如下：

  ```
  // AbstractBeanFactory.java
  
  /** Map from bean name to merged RootBeanDefinition. */
  private final Map<String, RootBeanDefinition> mergedBeanDefinitions = new ConcurrentHashMap<>(256);
  
  protected RootBeanDefinition getMergedLocalBeanDefinition(String beanName) throws BeansException {
      // Quick check on the concurrent map first, with minimal locking.
      // 快速从缓存中获取，如果不为空，则直接返回
      RootBeanDefinition mbd = this.mergedBeanDefinitions.get(beanName);
      if (mbd != null) {
          return mbd;
      }
      // 获取 RootBeanDefinition，
      // 如果返回的 BeanDefinition 是子类 bean 的话，则合并父类相关属性
      return getMergedBeanDefinition(beanName, getBeanDefinition(beanName));
  }
  ```

  - 首先，直接从 `mergedBeanDefinitions` 缓存中获取相应的 RootBeanDefinition 对象，如果存在则直接返回。
  - 否则，调用 `#getMergedBeanDefinition(String beanName, BeanDefinition bd)` 方法，获取 RootBeanDefinition 对象。若获取的 BeanDefinition 为**子** BeanDefinition，则需要合并**父类**的相关属性。关于该方法的源码，本文不做详细解析。感兴趣的胖友，可以自己研究。

- 调用 `#checkMergedBeanDefinition()` 方法，检查给定的合并的 BeanDefinition 对象。代码如下：

  ```
  // AbstractBeanFactory.java
  
  protected void checkMergedBeanDefinition(RootBeanDefinition mbd, String beanName, @Nullable Object[] args)
  		throws BeanDefinitionStoreException {
  	if (mbd.isAbstract()) {
  		throw new BeanIsAbstractException(beanName);
  	}
  }
  ```

## 5. 处理依赖

如果一个 Bean 有依赖 Bean 的话，那么在初始化该 Bean 时是需要先初始化它所依赖的 Bean 。代码如下：

```
// AbstractBeanFactory.java

// Guarantee initialization of beans that the current bean depends on.
// 处理所依赖的 bean
String[] dependsOn = mbd.getDependsOn();
if (dependsOn != null) {
    for (String dep : dependsOn) {
        // <1> 若给定的依赖 bean 已经注册为依赖给定的 bean
        // 即循环依赖的情况，抛出 BeanCreationException 异常
        if (isDependent(beanName, dep)) {
            throw new BeanCreationException(mbd.getResourceDescription(), beanName,
                    "Circular depends-on relationship between '" + beanName + "' and '" + dep + "'");
        }
        // <2> 缓存依赖调用 TODO 芋艿
        registerDependentBean(dep, beanName);
        try {
            // <3> 递归处理依赖 Bean
            getBean(dep);
        } catch (NoSuchBeanDefinitionException ex) {
            throw new BeanCreationException(mbd.getResourceDescription(), beanName,
                    "'" + beanName + "' depends on missing bean '" + dep + "'", ex);
        }
    }
}
```

- 这段代码逻辑是：通过迭代的方式依次对依赖 bean 进行检测、校验。如果通过，则调用 `#getBean(String beanName)` 方法，实例化**依赖**的 Bean 对象。

#### 5.1 isDependent

`<1>` 处，调用 `#isDependent(String beanName, String dependentBeanName)` 方法，是校验该依赖是否已经注册给当前 Bean 。代码如下：

```
// DefaultSingletonBeanRegistry.java

/**
 * Map between dependent bean names: bean name to Set of dependent bean names.
 *
 * 保存的是依赖 beanName 之间的映射关系：beanName - > 依赖 beanName 的集合
 */
private final Map<String, Set<String>> dependentBeanMap = new ConcurrentHashMap<>(64);

protected boolean isDependent(String beanName, String dependentBeanName) {
	synchronized (this.dependentBeanMap) {
		return isDependent(beanName, dependentBeanName, null);
	}
}
```

- `dependentBeanMap` 对象保存的是依赖 `beanName` 之间的映射关系：`beanName` - > 依赖 `beanName` 的集合。

- 同步加锁给 `dependentBeanMap` 对象，然后调用 `#isDependent(String beanName, String dependentBeanName, Set<String> alreadySeen)` 方法，进行校验。代码如下：

  ```
  // DefaultSingletonBeanRegistry.java
  
  private boolean isDependent(String beanName, String dependentBeanName, @Nullable Set<String> alreadySeen) {
      // alreadySeen 已经检测的依赖 bean
      if (alreadySeen != null && alreadySeen.contains(beanName)) {
          return false;
      }
      // 获取原始 beanName
      String canonicalName = canonicalName(beanName);
      // 获取当前 beanName 的依赖集合
      Set<String> dependentBeans = this.dependentBeanMap.get(canonicalName);
      if (dependentBeans == null) {
          return false;
      }
      // 存在，则证明存在已经注册的依赖
      if (dependentBeans.contains(dependentBeanName)) {
          return true;
      }
      // 递归检测依赖
      for (String transitiveDependency : dependentBeans) {
          if (alreadySeen == null) {
              alreadySeen = new HashSet<>();
          }
          // 添加到 alreadySeen 中
          alreadySeen.add(beanName);
          // 递推
          if (isDependent(transitiveDependency, dependentBeanName, alreadySeen)) {
              return true;
          }
      }
      return false;
  }
  ```

  - 代码比较长，当然也有点绕。感兴趣的胖友，可以调试下。

#### 5.2 registerDependentBean

`<2>` 处，如果校验成功，则调用 `#registerDependentBean(String beanName, String dependentBeanName)` 方法，将该依赖进行注册，便于在销毁 Bean 之前对其进行销毁。代码如下：

```
// DefaultSingletonBeanRegistry.java

/**
 * Map between dependent bean names: bean name to Set of dependent bean names.
 *
 * 保存的是依赖 beanName 之间的映射关系：beanName - > 依赖 beanName 的集合
 */
private final Map<String, Set<String>> dependentBeanMap = new ConcurrentHashMap<>(64);

/**
 * Map between depending bean names: bean name to Set of bean names for the bean's dependencies.
 *
 * 保存的是依赖 beanName 之间的映射关系：依赖 beanName - > beanName 的集合
 */
private final Map<String, Set<String>> dependenciesForBeanMap = new ConcurrentHashMap<>(64);

public void registerDependentBean(String beanName, String dependentBeanName) {
    // 获取 beanName
    String canonicalName = canonicalName(beanName);

    // 添加 <canonicalName, <dependentBeanName>> 到 dependentBeanMap 中
    synchronized (this.dependentBeanMap) {
        Set<String> dependentBeans =
                this.dependentBeanMap.computeIfAbsent(canonicalName, k -> new LinkedHashSet<>(8));
        if (!dependentBeans.add(dependentBeanName)) {
            return;
        }
    }

    // 添加 <dependentBeanName, <canonicalName>> 到 dependenciesForBeanMap 中
    synchronized (this.dependenciesForBeanMap) {
        Set<String> dependenciesForBean =
                this.dependenciesForBeanMap.computeIfAbsent(dependentBeanName, k -> new LinkedHashSet<>(8));
        dependenciesForBean.add(canonicalName);
    }
}
```

- 其实将就是该映射关系保存到两个集合中：`dependentBeanMap`、`dependenciesForBeanMap` 。

#### 5.3 getBean

`<3>` 处，最后调用 `#getBean(String beanName)` 方法，实例化依赖 Bean 对象。

## 6. 小结

至此，加载 bean 的第二个部分也分析完毕了，下篇开始分析第三个部分：各大作用域 bean 的处理。

# 4、分析各 scope 的 Bean 创建

﻿在 Spring 中存在着不同的 scope，默认是 singleton ，还有 prototype、request 等等其他的 scope 。他们的初始化步骤是怎样的呢？这个答案在这篇博客中给出。

## 1. singleton

Spring 的 scope 默认为 singleton ，其初始化的代码如下：

```
// AbstractBeanFactory.java

if (mbd.isSingleton()) { // 单例模式
    sharedInstance = getSingleton(beanName, () -> {
        try {
            return createBean(beanName, mbd, args);
        }
        catch (BeansException ex) {
            // Explicitly remove instance from singleton cache: It might have been put there
            // eagerly by the creation process, to allow for circular reference resolution.
            // Also remove any beans that received a temporary reference to the bean.
            // 显式从单例缓存中删除 Bean 实例
            // 因为单例模式下为了解决循环依赖，可能他已经存在了，所以销毁它。 TODO 芋艿
            destroySingleton(beanName);
            throw ex;
        }
    });
    // <x>
    bean = getObjectForBeanInstance(sharedInstance, name, beanName, mbd);
}
```

- 在 [《【死磕 Spring】—— ﻿IoC 之加载 Bean：从单例缓存中获取单》](http://svip.iocoder.cn/Spring/IoC-get-Bean-getSingleton) 中，已经分析了从缓存中获取单例模式的 bean 。但是如果缓存中**不存在**呢？则需要从头开始加载 Bean ，这个过程由 `#getSingleton(String beanName, ObjectFactory<?> singletonFactory)` 方法来实现。代码如下：

  ```
  // DefaultSingletonBeanRegistry.java
  
  public Object getSingleton(String beanName, ObjectFactory<?> singletonFactory) {
      Assert.notNull(beanName, "Bean name must not be null");
      // 全局加锁
      synchronized (this.singletonObjects) {
          // <1> 从缓存中检查一遍
          // 因为 singleton 模式其实就是复用已经创建的 bean 所以这步骤必须检查
          Object singletonObject = this.singletonObjects.get(beanName);
          //  为空，开始加载过程
          if (singletonObject == null) {
              if (this.singletonsCurrentlyInDestruction) {
                  throw new BeanCreationNotAllowedException(beanName,
                          "Singleton bean creation not allowed while singletons of this factory are in destruction " +
                          "(Do not request a bean from a BeanFactory in a destroy method implementation!)");
              }
              if (logger.isDebugEnabled()) {
                  logger.debug("Creating shared instance of singleton bean '" + beanName + "'");
              }
              // <2> 加载前置处理
              beforeSingletonCreation(beanName);
              boolean newSingleton = false;
              boolean recordSuppressedExceptions = (this.suppressedExceptions == null);
              if (recordSuppressedExceptions) {
                  this.suppressedExceptions = new LinkedHashSet<>();
              }
              try {
                  // <3> 初始化 bean
                  // 这个过程其实是调用 createBean() 方法
                  singletonObject = singletonFactory.getObject();
                  newSingleton = true;
              } catch (IllegalStateException ex) {
                  // Has the singleton object implicitly appeared in the meantime ->
                  // if yes, proceed with it since the exception indicates that state.
                  singletonObject = this.singletonObjects.get(beanName);
                  if (singletonObject == null) {
                      throw ex;
                  }
              } catch (BeanCreationException ex) {
                  if (recordSuppressedExceptions) {
                      for (Exception suppressedException : this.suppressedExceptions) {
                          ex.addRelatedCause(suppressedException);
                      }
                  }
                  throw ex;
              } finally {
                  if (recordSuppressedExceptions) {
                      this.suppressedExceptions = null;
                  }
                  // <4> 后置处理
                  afterSingletonCreation(beanName);
              }
              // <5> 加入缓存中
              if (newSingleton) {
                  addSingleton(beanName, singletonObject);
              }
          }
          return singletonObject;
      }
  }
  ```

  - 其实，这个过程并没有真正创建 Bean 对象，仅仅只是做了一部分准备和预处理步骤。真正获取单例 bean 的方法，其实是由 `<3>` 处的 `singletonFactory.getObject()` 这部分代码块来实现，而 `singletonFactory` 由回调方法产生。
  - 那么这个方法做了哪些准备呢？
    - `<1>` 处，再次检查缓存是否已经加载过，如果已经加载了则直接返回，否则开始加载过程。
    - `<2>` 处，调用 `#beforeSingletonCreation(String beanName)` 方法，记录加载单例 bean 之前的加载状态，即前置处理。在 [《【死磕 Spring】—— ﻿IoC 之加载 Bean：从单例缓存中获取单例 Bean》](http://svip.iocoder.cn/Spring/IoC-get-Bean-getSingleton) 中，已经详细解析。
    - `<3>` 处，调用参数传递的 ObjectFactory 的 `#getObject()` 方法，实例化 bean 。【重要】后续文章，详细解析。
    - `<4>` 处，调用 `#afterSingletonCreation(String beanName)` 方法，进行加载单例后的后置处理。在 [《【死磕 Spring】—— ﻿IoC 之加载 Bean：从单例缓存中获取单例 Bean》](http://svip.iocoder.cn/Spring/IoC-get-Bean-getSingleton) 中，已经详细解析。
    - `<5>` 处，调用 `#addSingleton(String beanName, Object singletonObject)` 方法，将结果记录并加入值缓存中，同时删除加载 bean 过程中所记录的一些辅助状态。详细解析，见 [「1.1 addSingleton」](http://svip.iocoder.cn/Spring/IoC-get-Bean-different-scope/#) 。

- 在 `<x>` 处，加载了单例 bean 后，调用 `#getObjectForBeanInstance(Object beanInstance, String name, String beanName, RootBeanDefinition mbd)` 方法，从 bean 实例中获取对象。该方法已经在 [《【死磕 Spring】—— ﻿IoC 之加载 Bean：从单例缓存中获取单例 Bean》](http://svip.iocoder.cn/Spring/IoC-get-Bean-getSingleton) 中，详细分析了。

#### 1.1 addSingleton

```
// DefaultSingletonBeanRegistry.java

/**
 * Cache of singleton objects: bean name to bean instance.
 *
 * 存放的是单例 bean 的映射。
 *
 * 对应关系为 bean name --> bean instance
 */
private final Map<String, Object> singletonObjects = new ConcurrentHashMap<>(256);

/**
 * Cache of singleton factories: bean name to ObjectFactory.
 *
 * 存放的是【早期】的单例 bean 的映射。
 *
 * 对应关系也是 bean name --> bean instance。
 *
 * 它与 {@link #singletonObjects} 的区别区别在，于 earlySingletonObjects 中存放的 bean 不一定是完整的。
 *
 * 从 {@link #getSingleton(String)} 方法中，中我们可以了解，bean 在创建过程中就已经加入到 earlySingletonObjects 中了，
 * 所以当在 bean 的创建过程中就可以通过 getBean() 方法获取。
 * 这个 Map 也是解决【循环依赖】的关键所在。
 **/
private final Map<String, ObjectFactory<?>> singletonFactories = new HashMap<>(16);

/**
 * Cache of early singleton objects: bean name to bean instance.
 *
 * 存放的是 ObjectFactory 的映射，可以理解为创建单例 bean 的 factory 。
 *
 * 对应关系是 bean name --> ObjectFactory
 */
private final Map<String, Object> earlySingletonObjects = new HashMap<>(16);

/** Set of registered singletons, containing the bean names in registration order. */
private final Set<String> registeredSingletons = new LinkedHashSet<>(256);

protected void addSingleton(String beanName, Object singletonObject) {
	synchronized (this.singletonObjects) {
		this.singletonObjects.put(beanName, singletonObject);
		this.singletonFactories.remove(beanName);
		this.earlySingletonObjects.remove(beanName);
		this.registeredSingletons.add(beanName);
	}
}
```

- 一个 put、一个 add、两个 remove 操作。
- 【put】`singletonObjects` 属性，单例 bean 的缓存。
- 【remove】`singletonFactories` 属性，单例 bean Factory 的缓存。
- 【remove】`earlySingletonObjects` 属性，“早期”创建的单例 bean 的缓存。
- 【add】`registeredSingletons` 属性，已经注册的单例缓存。

## 2. 原型模式

```
// AbstractBeanFactory.java

else if (mbd.isPrototype()) {
    Object prototypeInstance = null;
    try {
       // <1> 加载前置处理
        beforePrototypeCreation(beanName);
        // <2> 创建 Bean 对象
        prototypeInstance = createBean(beanName, mbd, args);
    } finally {
       // <3> 加载后缀处理
        afterPrototypeCreation(beanName);
    }
    // <4> 从 Bean 实例中获取对象
    bean = getObjectForBeanInstance(prototypeInstance, name, beanName, mbd);
}
```

原型模式的初始化过程很简单：直接创建一个**新**的 Bean 的实例就可以了。过程如下：

- 在 `<1>` 处，调用 `#beforePrototypeCreation(String beanName)` 方法，记录加载原型模式 bean 之前的加载状态，即前置处理。详细解析，见 [「2.1 beforePrototypeCreation」](http://svip.iocoder.cn/Spring/IoC-get-Bean-different-scope/#) 。
- 在 `<2>` 处，调用 `#createBean(String beanName)` 方法，创建一个 bean 实例对象。【重要】后续文章，详细解析。
- 在 `<3>` 处，调用 `#afterSingletonCreation(String beanName)` 方法，进行加载原型模式 bean 后的后置处理。详细解析，见 [「2.3 afterSingletonCreation」](http://svip.iocoder.cn/Spring/IoC-get-Bean-different-scope/#) 。
- 在 `<4>` 处，加载了单例 bean 后，调用 `#getObjectForBeanInstance(Object beanInstance, String name, String beanName, RootBeanDefinition mbd)` 方法，从 bean 实例中获取对象。该方法已经在 [《【死磕 Spring】—— ﻿IoC 之加载 Bean：从单例缓存中获取单例 Bean》](http://svip.iocoder.cn/Spring/IoC-get-Bean-getSingleton) 中，详细分析了。

#### 2.1 beforePrototypeCreation

```
// AbstractBeanFactory.java

/** Names of beans that are currently in creation. */
private final ThreadLocal<Object> prototypesCurrentlyInCreation =
		new NamedThreadLocal<>("Prototype beans currently in creation");
		
protected void beforePrototypeCreation(String beanName) {
	Object curVal = this.prototypesCurrentlyInCreation.get();
	if (curVal == null) { // String
		this.prototypesCurrentlyInCreation.set(beanName);
	} else if (curVal instanceof String) { // String => Set
		Set<String> beanNameSet = new HashSet<>(2);
		beanNameSet.add((String) curVal);
		beanNameSet.add(beanName);
		this.prototypesCurrentlyInCreation.set(beanNameSet);
	} else { // Set
		Set<String> beanNameSet = (Set<String>) curVal;
		beanNameSet.add(beanName);
	}
}
```

#### 2.2 afterSingletonCreation

```
// AbstractBeanFactory.java

protected void afterPrototypeCreation(String beanName) {
	Object curVal = this.prototypesCurrentlyInCreation.get();
	if (curVal instanceof String) { // String => null
		this.prototypesCurrentlyInCreation.remove();
	} else if (curVal instanceof Set) { // Set
		Set<String> beanNameSet = (Set<String>) curVal;
		beanNameSet.remove(beanName);
		if (beanNameSet.isEmpty()) { // Set => null
			this.prototypesCurrentlyInCreation.remove();
		}
	}
}
```

## 3. 其它作用域

```
// AbstractBeanFactory.java

else {
    // 获得 scopeName 对应的 Scope 对象
    String scopeName = mbd.getScope();
    final Scope scope = this.scopes.get(scopeName);
    if (scope == null) {
        throw new IllegalStateException("No Scope registered for scope name '" + scopeName + "'");
    }
    try {
        // 从指定的 scope 下创建 bean
        Object scopedInstance = scope.get(beanName, () -> {
            // 加载前置处理
            beforePrototypeCreation(beanName);
            try {
                // 创建 Bean 对象
                return createBean(beanName, mbd, args);
            } finally {
                // 加载后缀处理
                afterPrototypeCreation(beanName);
            }
        });
        // 从 Bean 实例中获取对象
        bean = getObjectForBeanInstance(scopedInstance, name, beanName, mbd);
    } catch (IllegalStateException ex) {
        throw new BeanCreationException(beanName,
                "Scope '" + scopeName + "' is not active for the current thread; consider " +
                "defining a scoped proxy for this bean if you intend to refer to it from a singleton",
                ex);
    }
}
```

- **核心流程和原型模式一样**，只不过获取 bean 实例是由 `Scope#get(String name, ObjectFactory<?> objectFactory)` 方法来实现。代码如下：

  ```
  // SimpleThreadScope.java
  
  private final ThreadLocal<Map<String, Object>> threadScope =
      new NamedThreadLocal<Map<String, Object>>("SimpleThreadScope") {
          @Override
          protected Map<String, Object> initialValue() {
              return new HashMap<>();
          }
      };
  
  @Override
  public Object get(String name, ObjectFactory<?> objectFactory) {
      // 获取 scope 缓存
      Map<String, Object> scope = this.threadScope.get();
      Object scopedObject = scope.get(name);
      if (scopedObject == null) {
          scopedObject = objectFactory.getObject();
          // 加入缓存
          scope.put(name, scopedObject);
      }
      return scopedObject;
  }
  ```

  - `org.springframework.beans.factory.config.Scope` 接口，有**多种**实现类。其他的 Scope 实现类，感兴趣的胖友，可以单独去看。

## 4. 小结

对于上面三个模块，其中最重要的有两个方法：

- 一个是 `#createBean(String beanName, RootBeanDefinition mbd, Object[] args)` 方法。
- 一个是 `#getObjectForBeanInstance(Object beanInstance, String name, String beanName, RootBeanDefinition mbd)` 方法。

这两个方法在上面三个模块都有调用。

- `#createBean(String beanName, RootBeanDefinition mbd, Object[] args)` 方法，后续详细说明。

- `#getObjectForBeanInstance(Object beanInstance, String name, String beanName, RootBeanDefinition mbd)` 方法，在博客 [《【死磕 Spring】—— ﻿IoC 之加载 Bean：从单例缓存中获取单》](http://svip.iooder.cn/Spring/IoC-get-Bean-getSingleton) 中有详细讲解。这里再次阐述下（此段内容来自《**Spring 源码深度解析**》）：

  > 这个方法主要是验证以下我们得到的 bean 的正确性，其实就是检测当前 bean 是否是 FactoryBean 类型的 bean 。
  >
  > 如果是，那么需要调用该 bean 对应的 FactoryBean 实例的 `#getObject()` 方法，作为返回值。
  >
  > 无论是从缓存中获得到的 bean 还是通过不同的 scope 策略加载的 bean 都只是最原始的 bean 状态，并不一定就是我们最终想要的 bean。
  >
  > 举个例子，加入我们需要对工厂 bean 进行处理，那么这里得到的其实是工厂 bean 的初始状态，但是我们真正需要的是工厂 bean 中定义 `factory-method` 方法中返回的 bean，而 `#getObjectForBeanInstance(Object beanInstance, String name, String beanName, RootBeanDefinition mbd)` 方法，就是完成这个工作的。

至此，Spring 加载 bean 的三个部分（LZ自己划分的）已经分析完毕了。

# 5、创建 Bean（一）之主流程

## 1. createBean 抽象方法

﻿在上篇博客 [《【死磕 Spring】—— IoC 之加载 Bean：分析各 scope 的 Bean 创建》](http://svip.iocoder.cn/Spring/IoC-get-Bean-different-scope) 中，有一个核心方法没有讲到， `#createBean(String beanName, RootBeanDefinition mbd, Object[] args)` 方法，代码如下：

```
// AbstractBeanFactory.java

protected abstract Object createBean(String beanName, RootBeanDefinition mbd, @Nullable Object[] args)
		throws BeanCreationException;
```

- 该方法定义在 AbstractBeanFactory 中，其含义是根据给定的 BeanDefinition 和 `args` 实例化一个 Bean 对象。
- 如果该 BeanDefinition 存在父类，则该 BeanDefinition 已经合并了父类的属性。
- 所有 Bean 实例的创建，都会委托给该方法实现。
- 该方法接受三个方法参数：
  - `beanName` ：bean 的名字。
  - `mbd` ：已经合并了父类属性的（如果有的话）BeanDefinition 对象。
  - `args` ：用于构造函数或者工厂方法创建 Bean 实例对象的参数。

## 2. createBean 默认实现

该抽象方法的默认实现是在类 AbstractAutowireCapableBeanFactory 中实现，代码如下：

```
// AbstractAutowireCapableBeanFactory.java

@Override
protected Object createBean(String beanName, RootBeanDefinition mbd, @Nullable Object[] args)
        throws BeanCreationException {

    if (logger.isTraceEnabled()) {
        logger.trace("Creating instance of bean '" + beanName + "'");
    }
    RootBeanDefinition mbdToUse = mbd;

    // Make sure bean class is actually resolved at this point, and
    // clone the bean definition in case of a dynamically resolved Class
    // which cannot be stored in the shared merged bean definition.
    // <1> 确保此时的 bean 已经被解析了
    // 如果获取的class 属性不为null，则克隆该 BeanDefinition
    // 主要是因为该动态解析的 class 无法保存到到共享的 BeanDefinition
    Class<?> resolvedClass = resolveBeanClass(mbd, beanName);
    if (resolvedClass != null && !mbd.hasBeanClass() && mbd.getBeanClassName() != null) {
        mbdToUse = new RootBeanDefinition(mbd);
        mbdToUse.setBeanClass(resolvedClass);
    }

    // Prepare method overrides.
    try {
        // <2> 验证和准备覆盖方法
        mbdToUse.prepareMethodOverrides();
    } catch (BeanDefinitionValidationException ex) {
        throw new BeanDefinitionStoreException(mbdToUse.getResourceDescription(),
                beanName, "Validation of method overrides failed", ex);
    }

    try {
        // Give BeanPostProcessors a chance to return a proxy instead of the target bean instance.
        // <3> 实例化的前置处理
        // 给 BeanPostProcessors 一个机会用来返回一个代理类而不是真正的类实例
        // AOP 的功能就是基于这个地方
        Object bean = resolveBeforeInstantiation(beanName, mbdToUse);
        if (bean != null) {
            return bean;
        }
    } catch (Throwable ex) {
        throw new BeanCreationException(mbdToUse.getResourceDescription(), beanName,
                "BeanPostProcessor before instantiation of bean failed", ex);
    }

    try {
        // <4> 创建 Bean 对象
        Object beanInstance = doCreateBean(beanName, mbdToUse, args);
        if (logger.isTraceEnabled()) {
            logger.trace("Finished creating instance of bean '" + beanName + "'");
        }
        return beanInstance;
    } catch (BeanCreationException | ImplicitlyAppearedSingletonException ex) {
        // A previously detected exception with proper bean creation context already,
        // or illegal singleton state to be communicated up to DefaultSingletonBeanRegistry.
        throw ex;
    } catch (Throwable ex) {
        throw new BeanCreationException(
                mbdToUse.getResourceDescription(), beanName, "Unexpected exception during bean creation", ex);
    }
}
```

过程如下：

- `<1>` 处，解析指定 BeanDefinition 的 class 属性。
- `<2>` 处，处理 `override` 属性。
- `<3>` 处，实例化的前置处理。
- `<4>` 处，创建 Bean 对象。

详细解析，每 `<x>` 处，对应下面的 [「2.x」](http://svip.iocoder.cn/Spring/IoC-get-Bean-createBean-1/#) 。

#### 2.1 解析指定 BeanDefinition 的 class

```
// AbstractAutowireCapableBeanFactory.java

Class<?> resolvedClass = resolveBeanClass(mbd, beanName);
if (resolvedClass != null && !mbd.hasBeanClass() && mbd.getBeanClassName() != null) {
    mbdToUse = new RootBeanDefinition(mbd);
    mbdToUse.setBeanClass(resolvedClass);
}
```

- `#resolveBeanClass(final RootBeanDefinition mbd, String beanName, final Class<?>... typesToMatch)` 方法，主要是解析 bean definition 的 class 类，并将已经解析的 Class 存储在 bean definition 中以供后面使用。
- 如果解析的 class 不为空，则会将该 BeanDefinition 进行设置到 `mbdToUse` 中。这样做的主要目的是，以为动态解析的 class 是无法保存到共享的 BeanDefinition 中。

详细解析，见 TODO

#### 2.2 处理 override 属性

大家还记得 `lookup-method` 和 `replace-method` 这两个配置功能？在博客 [《【死磕 Spring】—— IoC 之解析 标签：meta、lookup-method、replace-method》](http://svip.iocoder.cn/Spring/IoC-parse-BeanDefinitions-for-meta-and-look-method-and-replace-method) 中，已经详细分析了这两个标签的用法和解析过程，知道解析过程其实就是讲这两个配置存放在 BeanDefinition 中的 `methodOverrides` 属性中。

我们知道在 bean 实例化的过程中如果检测到存在 `methodOverrides` ，则会动态地位为当前 bean 生成代理并使用对应的拦截器为 bean 做**增强处理**。具体的实现我们后续分析，现在先看 `mbdToUse.prepareMethodOverrides()` 代码块，都干了些什么事，代码如下：

```
// AbstractBeanDefinition.java

public void prepareMethodOverrides() throws BeanDefinitionValidationException {
    // Check that lookup methods exists.
    if (hasMethodOverrides()) {
        Set<MethodOverride> overrides = getMethodOverrides().getOverrides();
        synchronized (overrides) { // 同步
            // 循环，执行 prepareMethodOverride
            for (MethodOverride mo : overrides) {
                prepareMethodOverride(mo);
            }
        }
    }
}
```

- 如果存在 `methodOverrides` ，则获取所有的 override method ，然后通过迭代的方法一次调用 `#prepareMethodOverride(MethodOverride mo)` 方法。代码如下：

  ```
  // AbstractBeanDefinition.java
  
  protected void prepareMethodOverride(MethodOverride mo) throws BeanDefinitionValidationException {
  	int count = ClassUtils.getMethodCountForName(getBeanClass(), mo.getMethodName());
  	if (count == 0) {
  		throw new BeanDefinitionValidationException(
  				"Invalid method override: no method with name '" + mo.getMethodName() +
  				"' on class [" + getBeanClassName() + "]");
  	} else if (count == 1) {
  		// Mark override as not overloaded, to avoid the overhead of arg type checking.
  		mo.setOverloaded(false);
  	}
  }
  ```

  - 根据方法名称，从 class 中获取该方法名的个数：
    - 如果个数为 0 ，则抛出 BeanDefinitionValidationException 异常。
    - 如果个数为 1 ，则设置该重载方法**没有被重载**。
  - 若一个类中存在**多个**重载方法，则在方法调用的时候还需要根据参数类型来判断到底重载的是哪个方法。在设置重载的时候其实这里做了一个**小小优化**，那就是当 `count == 1` 时，设置 `overloaded = false` ，这样表示该方法没有重载。这样，在后续调用的时候，便可以直接找到方法而不需要进行方法参数的校验。

诚然，其实 `mbdToUse.prepareMethodOverrides()` 代码块，并没有做什么实质性的工作，只是对 `methodOverrides` 属性做了一些简单的校验而已。

#### 2.3 实例化的前置处理

`#resolveBeforeInstantiation(String beanName, RootBeanDefinition mbd)` 方法的作用，是给 BeanPostProcessors 后置处理器返回一个**代理对象**的机会。其，实在调用该方法之前 Spring 一直都没有创建 bean ，那么这里返回一个 bean 的代理类有什么作用呢？作用体现在后面的 `if` 判断，代码如下：

```
// AbstractBeanDefinition.java

Object bean = resolveBeforeInstantiation(beanName, mbdToUse);
// ↓↓↓ 
if (bean != null) {
	return bean;
}
```

- 如果代理对象不为空，则直接返回代理对象，这一步骤有非常重要的作用，Spring 后续实现 AOP 就是基于这个地方判断的。

- `#resolveBeforeInstantiation(String beanName, RootBeanDefinition mbd)` 方法，代码如下：

  ```
  // AbstractAutowireCapableBeanFactory.java
  
  @Nullable
  protected Object resolveBeforeInstantiation(String beanName, RootBeanDefinition mbd) {
      Object bean = null;
      if (!Boolean.FALSE.equals(mbd.beforeInstantiationResolved)) {
          // Make sure bean class is actually resolved at this point.
          if (!mbd.isSynthetic() && hasInstantiationAwareBeanPostProcessors()) {
              Class<?> targetType = determineTargetType(beanName, mbd);
              if (targetType != null) {
                  // 前置
                  bean = applyBeanPostProcessorsBeforeInstantiation(targetType, beanName);
                  if (bean != null) {
                      // 后置
                      bean = applyBeanPostProcessorsAfterInitialization(bean, beanName);
                  }
              }
          }
          mbd.beforeInstantiationResolved = (bean != null);
      }
      return bean;
  }
  ```

  - 这个方法核心就在于 `applyBeanPostProcessorsBeforeInstantiation()` 和 `applyBeanPostProcessorsAfterInitialization()` 两个方法，before 为实例化前的后处理器应用，after 为实例化后的后处理器应用。
  - 由于本文的主题是创建 bean ，关于 Bean 的增强处理后续 LZ 会单独出博文来做详细说明。

详细解析，见 TODO

#### 2.4 创建 Bean

如果没有代理对象，就只能走常规的路线进行 bean 的创建了，该过程有 `#doCreateBean(final String beanName, final RootBeanDefinition mbd, final @Nullable Object[] args)` 方法来实现。代码如下：

```
// AbstractAutowireCapableBeanFactory.java

protected Object doCreateBean(final String beanName, final RootBeanDefinition mbd, final @Nullable Object[] args)
        throws BeanCreationException {

    // Instantiate the bean.
    // BeanWrapper 是对 Bean 的包装，其接口中所定义的功能很简单包括设置获取被包装的对象，获取被包装 bean 的属性描述器
    BeanWrapper instanceWrapper = null;
    // <1> 单例模型，则从未完成的 FactoryBean 缓存中删除
    if (mbd.isSingleton()) {
        instanceWrapper = this.factoryBeanInstanceCache.remove(beanName);
    }
    // <2> 使用合适的实例化策略来创建新的实例：工厂方法、构造函数自动注入、简单初始化
    if (instanceWrapper == null) {
        instanceWrapper = createBeanInstance(beanName, mbd, args);
    }
    // 包装的实例对象
    final Object bean = instanceWrapper.getWrappedInstance();
    // 包装的实例对象的类型
    Class<?> beanType = instanceWrapper.getWrappedClass();
    if (beanType != NullBean.class) {
        mbd.resolvedTargetType = beanType;
    }

    // Allow post-processors to modify the merged bean definition.
    // <3> 判断是否有后置处理
    // 如果有后置处理，则允许后置处理修改 BeanDefinition
    synchronized (mbd.postProcessingLock) {
        if (!mbd.postProcessed) {
            try {
                // 后置处理修改 BeanDefinition
                applyMergedBeanDefinitionPostProcessors(mbd, beanType, beanName);
            } catch (Throwable ex) {
                throw new BeanCreationException(mbd.getResourceDescription(), beanName,
                        "Post-processing of merged bean definition failed", ex);
            }
            mbd.postProcessed = true;
        }
    }

    // Eagerly cache singletons to be able to resolve circular references
    // even when triggered by lifecycle interfaces like BeanFactoryAware.
    // <4> 解决单例模式的循环依赖
    boolean earlySingletonExposure = (mbd.isSingleton() // 单例模式
            && this.allowCircularReferences // 运行循环依赖
            && isSingletonCurrentlyInCreation(beanName)); // 当前单例 bean 是否正在被创建
    if (earlySingletonExposure) {
        if (logger.isTraceEnabled()) {
            logger.trace("Eagerly caching bean '" + beanName +
                    "' to allow for resolving potential circular references");
        }
        // 提前将创建的 bean 实例加入到 singletonFactories 中
        // 这里是为了后期避免循环依赖
        addSingletonFactory(beanName, () -> getEarlyBeanReference(beanName, mbd, bean));
    }

    // Initialize the bean instance.
    // 开始初始化 bean 实例对象
    Object exposedObject = bean;
    try {
        // <5> 对 bean 进行填充，将各个属性值注入，其中，可能存在依赖于其他 bean 的属性
        // 则会递归初始依赖 bean
        populateBean(beanName, mbd, instanceWrapper);
        // <6> 调用初始化方法
        exposedObject = initializeBean(beanName, exposedObject, mbd);
    } catch (Throwable ex) {
        if (ex instanceof BeanCreationException && beanName.equals(((BeanCreationException) ex).getBeanName())) {
            throw (BeanCreationException) ex;
        } else {
            throw new BeanCreationException(
                    mbd.getResourceDescription(), beanName, "Initialization of bean failed", ex);
        }
    }

    // <7> 循环依赖处理
    if (earlySingletonExposure) {
        // 获取 earlySingletonReference
        Object earlySingletonReference = getSingleton(beanName, false);
        // 只有在存在循环依赖的情况下，earlySingletonReference 才不会为空
        if (earlySingletonReference != null) {
            // 如果 exposedObject 没有在初始化方法中被改变，也就是没有被增强
            if (exposedObject == bean) {
                exposedObject = earlySingletonReference;
            // 处理依赖
            } else if (!this.allowRawInjectionDespiteWrapping && hasDependentBean(beanName)) {
                String[] dependentBeans = getDependentBeans(beanName);
                Set<String> actualDependentBeans = new LinkedHashSet<>(dependentBeans.length);
                for (String dependentBean : dependentBeans) {
                    if (!removeSingletonIfCreatedForTypeCheckOnly(dependentBean)) {
                        actualDependentBeans.add(dependentBean);
                    }
                }
                if (!actualDependentBeans.isEmpty()) {
                    throw new BeanCurrentlyInCreationException(beanName,
                            "Bean with name '" + beanName + "' has been injected into other beans [" +
                            StringUtils.collectionToCommaDelimitedString(actualDependentBeans) +
                            "] in its raw version as part of a circular reference, but has eventually been " +
                            "wrapped. This means that said other beans do not use the final version of the " +
                            "bean. This is often the result of over-eager type matching - consider using " +
                            "'getBeanNamesOfType' with the 'allowEagerInit' flag turned off, for example.");
                }
            }
        }
    }

    // Register bean as disposable.
    // <8> 注册 bean
    try {
        registerDisposableBeanIfNecessary(beanName, bean, mbd);
    } catch (BeanDefinitionValidationException ex) {
        throw new BeanCreationException(
                mbd.getResourceDescription(), beanName, "Invalid destruction signature", ex);
    }

    return exposedObject;
}
```

整体的思路：

- ```
  <1>
  ```

   

  处，如果是单例模式，则清除缓存。

  - 详细解析，见 TODO

- ```
  <2>
  ```

   

  处，调用

   

  ```
  #createBeanInstance(String beanName, RootBeanDefinition mbd, Object[] args)
  ```

   

  方法，实例化 bean ，主要是将 BeanDefinition 转换为

   

  ```
  org.springframework.beans.BeanWrapper
  ```

   

  对象。

  - 详细解析，见 [《【死磕 Spring】—— IoC 之加载 Bean：创建 Bean（二）》](http://svip.iocoder.cn/Spring/IoC-get-Bean-createBean-2) 和 [《【死磕 Spring】—— IoC 之加载 Bean：创建 Bean（三）》](http://svip.iocoder.cn/Spring/IoC-get-Bean-createBean-3) 中。

- ```
  <3>
  ```

   

  处，MergedBeanDefinitionPostProcessor 的应用。

  - 详细解析，见 TODO

- ```
  <4>
  ```

   

  处，单例模式的循环依赖处理。

  - 详细解析，见 [《【死磕 Spring】—— IoC 之加载 Bean：创建 Bean（五）之循环依赖处理》](http://svip.iocoder.cn/Spring/IoC-get-Bean-createBean-5) 。

- ```
  <5>
  ```

   

  处，调用

   

  ```
  #populateBean(String beanName, RootBeanDefinition mbd, BeanWrapper bw)
  ```

   

  方法，进行属性填充。将所有属性填充至 bean 的实例中。

  - 详细解析，见 [《【死磕 Spring】—— ﻿IoC 之加载 bean：创建 Bean（四）之属性填充》](http://svip.iocoder.cn/Spring/IoC-get-Bean-createBean-4) 。

- ```
  <6>
  ```

   

  处，调用

   

  ```
  #initializeBean(final String beanName, final Object bean, RootBeanDefinition mbd)
  ```

   

  方法，初始化 bean 。

  - 详细解析，见 [《死磕 Spring】—— IoC 之加载 Bean：创建 Bean（六）之初始化 Bean 对象》](http://svip.iocoder.cn/Spring/IoC-get-Bean-createBean-6) 。

- ```
  <7>
  ```

   

  处，依赖检查。

  - 详细解析，见 TODO

- ```
  <8>
  ```

   

  处，注册 DisposableBean 。

  - 详细解析，见 TODO

## 3. 小结

`#doCreateBean(...)` 方法，完成 bean 的创建和初始化工作，内容太多，这里就只列出整体思路。**下文开始**，将该方法进行拆分进行详细讲解，分布从以下几个方面进行阐述：

- `#createBeanInstance(String beanName, RootBeanDefinition mbd, Object[] args)` 方法，实例化 bean 。
- 循环依赖的处理。
- `#populateBean(String beanName, RootBeanDefinition mbd, BeanWrapper bw)` 方法，进行属性填充。
- `#initializeBean(final String beanName, final Object bean, RootBeanDefinition mbd)` 方法，初始化 Bean 。

# 6、创建 Bean（二）之实例化 Bean 对象

## 1. createBeanInstance

﻿这篇我们关注创建 bean 过程中的**第一个**步骤：实例化 bean，对应的方法为 `#createBeanInstance(String beanName, RootBeanDefinition mbd, Object[] args)` 。代码如下：

```
// AbstractAutowireCapableBeanFactory.java

protected BeanWrapper createBeanInstance(String beanName, RootBeanDefinition mbd, @Nullable Object[] args) {
    // Make sure bean class is actually resolved at this point.
    // 解析 bean ，将 bean 类名解析为 class 引用。
    Class<?> beanClass = resolveBeanClass(mbd, beanName);

    if (beanClass != null && !Modifier.isPublic(beanClass.getModifiers()) && !mbd.isNonPublicAccessAllowed()) { // 校验
        throw new BeanCreationException(mbd.getResourceDescription(), beanName,
                "Bean class isn't public, and non-public access not allowed: " + beanClass.getName());
    }

    // <1> 如果存在 Supplier 回调，则使用给定的回调方法初始化策略
    Supplier<?> instanceSupplier = mbd.getInstanceSupplier();
    if (instanceSupplier != null) {
        return obtainFromSupplier(instanceSupplier, beanName);
    }

    // <2> 使用 FactoryBean 的 factory-method 来创建，支持静态工厂和实例工厂
    if (mbd.getFactoryMethodName() != null)  {
        return instantiateUsingFactoryMethod(beanName, mbd, args);
    }

    // <3> Shortcut when re-creating the same bean...
    boolean resolved = false;
    boolean autowireNecessary = false;
    if (args == null) {
        // constructorArgumentLock 构造函数的常用锁
        synchronized (mbd.constructorArgumentLock) {
            // 如果已缓存的解析的构造函数或者工厂方法不为空，则可以利用构造函数解析
            // 因为需要根据参数确认到底使用哪个构造函数，该过程比较消耗性能，所有采用缓存机制
            if (mbd.resolvedConstructorOrFactoryMethod != null) {
                resolved = true;
                autowireNecessary = mbd.constructorArgumentsResolved;
            }
        }
    }
    // 已经解析好了，直接注入即可
    if (resolved) {
        // <3.1> autowire 自动注入，调用构造函数自动注入
        if (autowireNecessary) {
            return autowireConstructor(beanName, mbd, null, null);
        } else {
            // <3.2> 使用默认构造函数构造
            return instantiateBean(beanName, mbd);
        }
    }

    // Candidate constructors for autowiring?
    // <4> 确定解析的构造函数
    // 主要是检查已经注册的 SmartInstantiationAwareBeanPostProcessor
    Constructor<?>[] ctors = determineConstructorsFromBeanPostProcessors(beanClass, beanName);
    // <4.1> 有参数情况时，创建 Bean 。先利用参数个数，类型等，确定最精确匹配的构造方法。
    if (ctors != null || mbd.getResolvedAutowireMode() == AUTOWIRE_CONSTRUCTOR ||
            mbd.hasConstructorArgumentValues() || !ObjectUtils.isEmpty(args))  {
        return autowireConstructor(beanName, mbd, ctors, args);
    }

    // Preferred constructors for default construction?
    // <4.1> 选择构造方法，创建 Bean 。
    ctors = mbd.getPreferredConstructors();
    if (ctors != null) {
        return autowireConstructor(beanName, mbd, ctors, null); // args = null
    }

    // No special handling: simply use no-arg constructor.
    // <4.2> 有参数时，又没获取到构造方法，则只能调用无参构造方法来创建实例了(兜底方法)
    return instantiateBean(beanName, mbd);
}
```

实例化 Bean 对象，是一个**复杂**的过程，其主要的逻辑为：

- ```
  <1>
  ```

   

  处，如果存在 Supplier 回调，则调用

   

  ```
  #obtainFromSupplier(Supplier<?> instanceSupplier, String beanName)
  ```

   

  方法，进行初始化。

  - 详细解析，见 [「1.1 obtainFromSupplier」](http://svip.iocoder.cn/Spring/IoC-get-Bean-createBean-2/#) 。

- ```
  <2>
  ```

   

  处，如果存在工厂方法，则使用工厂方法进行初始化。

  - 详细解析，见 [「1.2 instantiateUsingFactoryMethod」](http://svip.iocoder.cn/Spring/IoC-get-Bean-createBean-2/#) 。

- ```
  <3>
  ```

   

  处，首先判断缓存，如果

  缓存中存在

  ，即已经解析过了，则直接使用已经解析了的。根据

   

  ```
  constructorArgumentsResolved
  ```

   

  参数来判断：

  - ```
    <3.1>
    ```

     

    处，是使用构造函数自动注入，即调用

     

    ```
    #autowireConstructor(String beanName, RootBeanDefinition mbd, Constructor<?>[] ctors, Object[] explicitArgs)
    ```

     

    方法。

    - 详细解析，[《【死磕 Spring】—— IoC 之加载 Bean：创建 Bean（三）之实例化 Bean 对象(2)》](http://svip.iocoder.cn/Spring/IoC-get-Bean-createBean-3) 。

  - ```
    <3.2>
    ```

     

    处，还是默认构造函数，即调用

     

    ```
    #instantiateBean(final String beanName, final RootBeanDefinition mbd)
    ```

     

    方法。

    - 详细解析，[《【死磕 Spring】—— IoC 之加载 Bean：创建 Bean（三）之实例化 Bean 对象(2)》](http://svip.iocoder.cn/Spring/IoC-get-Bean-createBean-3) 。

- ```
  <4>
  ```

   

  处，如果

  缓存中没有

  ，则需要先确定到底使用哪个构造函数来完成解析工作，因为一个类有多个构造函数，每个构造函数都有不同的构造参数，所以需要根据参数来锁定构造函数并完成初始化。

  - ```
    <4.1>
    ```

     

    处，如果存在参数，则使用相应的带有参数的构造函数，即调用

     

    ```
    #autowireConstructor(String beanName, RootBeanDefinition mbd, Constructor<?>[] ctors, Object[] explicitArgs)
    ```

     

    方法。

    - 详细解析，[《【死磕 Spring】—— IoC 之加载 Bean：创建 Bean（三）之实例化 Bean 对象(2)》](http://svip.iocoder.cn/Spring/IoC-get-Bean-createBean-3) 。

  - ```
    <4.2>
    ```

     

    处，否则，使用默认构造函数，即调用

     

    ```
    #instantiateBean(final String beanName, final RootBeanDefinition mbd)
    ```

     

    方法。

    - 详细解析，[《【死磕 Spring】—— IoC 之加载 Bean：创建 Bean（三）之实例化 Bean 对象(2)》](http://svip.iocoder.cn/Spring/IoC-get-Bean-createBean-3) 。

😈 下面就上面四种情况做分别说明。

#### 1.1 obtainFromSupplier

调用对应代码块如下：

```
// AbstractAutowireCapableBeanFactory.java
// protected BeanWrapper createBeanInstance(String beanName, RootBeanDefinition mbd, @Nullable Object[] args) {}

Supplier<?> instanceSupplier = mbd.getInstanceSupplier();
if (instanceSupplier != null) {
    return obtainFromSupplier(instanceSupplier, beanName);
}
```

- 首先,从 BeanDefinition 中获取 Supplier 对象。如果不为空，则调用 `#obtainFromSupplier(final String beanName, final RootBeanDefinition mbd)` 方法。

###### 1.1.1 Supplier

那么 Supplier 是什么呢？在这之前也没有提到过这个字段。`java.util.function.Supplier` 接口，代码如下：

```
public interface Supplier<T> {

    T get();
    
}
```

- Supplier 接口仅有一个功能性的 `#get()` 方法，该方法会返回一个 `<T>` 类型的对象，有点儿类似工厂方法。
- 这个接口有什么作用？用于指定创建 bean 的回调。如果我们设置了这样的回调，那么其他的构造器或者工厂方法都会没有用。

在什么设置该 Supplier 参数呢？Spring 提供了相应的 setter 方法，如下：

```
// AbstractBeanDefinition.java

/**
 * 创建 Bean 的 Supplier 对象
 */
@Nullable
private Supplier<?> instanceSupplier;

public void setInstanceSupplier(@Nullable Supplier<?> instanceSupplier) {
	this.instanceSupplier = instanceSupplier;
}
```

- 在构造 BeanDefinition 对象的时候，设置了 `instanceSupplier` 该值，代码如下（以 RootBeanDefinition 为例）：

  ```
  // RootBeanDefinition.java
  
  public <T> RootBeanDefinition(@Nullable Class<T> beanClass, String scope, @Nullable Supplier<T> instanceSupplier) {
  	super();
  	setBeanClass(beanClass);
  	setScope(scope);
  	// 设置 instanceSupplier 属性
  	setInstanceSupplier(instanceSupplier);
  }
  ```

###### 1.1.2 obtainFromSupplier

如果设置了 `instanceSupplier` 属性，则可以调用 `#obtainFromSupplier(Supplier<?> instanceSupplier, String beanName)` 方法，完成 Bean 的初始化。代码如下：

```
// AbstractAutowireCapableBeanFactory.java

/**
 * 当前线程，正在创建的 Bean 对象的名字
 *
 * The name of the currently created bean, for implicit dependency registration
 * on getBean etc invocations triggered from a user-specified Supplier callback.
 */
private final NamedThreadLocal<String> currentlyCreatedBean = new NamedThreadLocal<>("Currently created bean");

protected BeanWrapper obtainFromSupplier(Supplier<?> instanceSupplier, String beanName) {
    Object instance;
    // 获得原创建的 Bean 的对象名
    String outerBean = this.currentlyCreatedBean.get();
    // 设置新的 Bean 的对象名，到 currentlyCreatedBean 中
    this.currentlyCreatedBean.set(beanName);
    try {
        // <1> 调用 Supplier 的 get()，返回一个 Bean 对象
        instance = instanceSupplier.get();
    } finally {
        // 设置原创建的 Bean 的对象名，到 currentlyCreatedBean 中
        if (outerBean != null) {
            this.currentlyCreatedBean.set(outerBean);
        } else {
            this.currentlyCreatedBean.remove();
        }
    }

    // 未创建 Bean 对象，则创建 NullBean 对象
    if (instance == null) {
        instance = new NullBean();
    }
    // <2> 创建 BeanWrapper 对象
    BeanWrapper bw = new BeanWrapperImpl(instance);
    // <3> 初始化 BeanWrapper 对象
    initBeanWrapper(bw);
    return bw;
}
```

代码很简单，流程如下：

- `<1>` 首先，调用 Supplier 的 `get()` 方法，获得一个 Bean 实例对象。
- `<2>` 然后，根据该实例对象构造一个 BeanWrapper 对象 `bw` 。
- `<3>` 最后，初始化该对象。

有关于 BeanWrapper ，后面专门出文讲解。

#### 1.2 instantiateUsingFactoryMethod()

如果存在工厂方法，则调用 `#instantiateUsingFactoryMethod(String beanName, RootBeanDefinition mbd, @Nullable Object[] explicitArgs)` 方法完成 bean 的初始化工作（方法实现比较长，细节比较复杂，**各位就硬着头皮看吧**）。代码如下：

```
// AbstractAutowireCapableBeanFactory.java

protected BeanWrapper instantiateUsingFactoryMethod(String beanName, RootBeanDefinition mbd, @Nullable Object[] explicitArgs) {
	return new ConstructorResolver(this).instantiateUsingFactoryMethod(beanName, mbd, explicitArgs);
}
```

- 构造一个 ConstructorResolver 对象，然后调用其 `#instantiateUsingFactoryMethod(EvaluationContext context, String typeName, List<TypeDescriptor> argumentTypes)` 方法。

###### 1.2.1 ConstructorResolver

`org.springframework.expression.ConstructorResolver` 是构造方法或者工厂类初始化 bean 的委托类。代码如下：

```
// ConstructorResolver.java

public BeanWrapper instantiateUsingFactoryMethod(
        String beanName, RootBeanDefinition mbd, @Nullable Object[] explicitArgs) {
    // 构造 BeanWrapperImpl 对象
    BeanWrapperImpl bw = new BeanWrapperImpl();
    // 初始化 BeanWrapperImpl
    // 向BeanWrapper对象中添加 ConversionService 对象和属性编辑器 PropertyEditor 对象
    this.beanFactory.initBeanWrapper(bw);

    // <1> 获得 factoryBean、factoryClass、isStatic、factoryBeanName 属性
    Object factoryBean;
    Class<?> factoryClass;
    boolean isStatic;
    String factoryBeanName = mbd.getFactoryBeanName();

    // 工厂名不为空
    if (factoryBeanName != null) {
        if (factoryBeanName.equals(beanName)) { // 抛出 BeanDefinitionStoreException 异常
            throw new BeanDefinitionStoreException(mbd.getResourceDescription(), beanName,
                    "factory-bean reference points back to the same bean definition");
        }
        // 获取工厂实例
        factoryBean = this.beanFactory.getBean(factoryBeanName);
        if (mbd.isSingleton() && this.beanFactory.containsSingleton(beanName)) { // 抛出 ImplicitlyAppearedSingletonException 异常
            throw new ImplicitlyAppearedSingletonException();
        }
        factoryClass = factoryBean.getClass();
        isStatic = false;
    } else {
        // 工厂名为空，则其可能是一个静态工厂
        // 静态工厂创建bean，必须要提供工厂的全类名
        // It's a static factory method on the bean class.
        if (!mbd.hasBeanClass()) {
            throw new BeanDefinitionStoreException(mbd.getResourceDescription(), beanName,
                    "bean definition declares neither a bean class nor a factory-bean reference");
        }
        factoryBean = null;
        factoryClass = mbd.getBeanClass();
        isStatic = true;
    }

    // <2> 获得 factoryMethodToUse、argsHolderToUse、argsToUse 属性
    Method factoryMethodToUse = null;  // 工厂方法
    ArgumentsHolder argsHolderToUse = null;
    Object[] argsToUse = null; // 参数

    // <2.1> 如果指定了构造参数则直接使用
    // 在调用 getBean 方法的时候指定了方法参数
    if (explicitArgs != null) {
        argsToUse = explicitArgs;
    } else {
        // 没有指定，则尝试从配置文件中解析
        Object[] argsToResolve = null;
        // <2.2> 首先尝试从缓存中获取
        synchronized (mbd.constructorArgumentLock) {
            // 获取缓存中的构造函数或者工厂方法
            factoryMethodToUse = (Method) mbd.resolvedConstructorOrFactoryMethod;
            if (factoryMethodToUse != null && mbd.constructorArgumentsResolved) {
                // Found a cached factory method...
                // 获取缓存中的构造参数
                argsToUse = mbd.resolvedConstructorArguments;
                if (argsToUse == null) {
                    // 获取缓存中的构造函数参数的包可见字段
                    argsToResolve = mbd.preparedConstructorArguments;
                }
            }
        }
        // 缓存中存在,则解析存储在 BeanDefinition 中的参数
        // 如给定方法的构造函数 A(int ,int )，则通过此方法后就会把配置文件中的("1","1")转换为 (1,1)
        // 缓存中的值可能是原始值也有可能是最终值
        if (argsToResolve != null) {
            argsToUse = resolvePreparedArguments(beanName, mbd, bw, factoryMethodToUse, argsToResolve, true);
        }
    }

    // <3> 
    if (factoryMethodToUse == null || argsToUse == null) {
        // Need to determine the factory method...
        // Try all methods with this name to see if they match the given arguments.
        // 获取工厂方法的类全名称
        factoryClass = ClassUtils.getUserClass(factoryClass);

        // 获取所有待定方法
        Method[] rawCandidates = getCandidateMethods(factoryClass, mbd);
        // 检索所有方法，这里是对方法进行过滤
        List<Method> candidateList = new ArrayList<>();
        for (Method candidate : rawCandidates) {
            // 如果有static 且为工厂方法，则添加到 candidateSet 中
            if (Modifier.isStatic(candidate.getModifiers()) == isStatic && mbd.isFactoryMethod(candidate)) {
                candidateList.add(candidate);
            }
        }

        // TODO 芋艿 创建 Bean
        if (candidateList.size() == 1 && explicitArgs == null && !mbd.hasConstructorArgumentValues()) {
            Method uniqueCandidate = candidateList.get(0);
            if (uniqueCandidate.getParameterCount() == 0) {
                synchronized (mbd.constructorArgumentLock) {
                    mbd.resolvedConstructorOrFactoryMethod = uniqueCandidate;
                    mbd.constructorArgumentsResolved = true;
                    mbd.resolvedConstructorArguments = EMPTY_ARGS;
                }
                bw.setBeanInstance(instantiate(beanName, mbd, factoryBean, uniqueCandidate, EMPTY_ARGS));
                return bw;
            }
        }

        Method[] candidates = candidateList.toArray(new Method[0]);
        // 排序构造函数
        // public 构造函数优先参数数量降序，非 public 构造函数参数数量降序
        AutowireUtils.sortFactoryMethods(candidates);

        // 用于承载解析后的构造函数参数的值
        ConstructorArgumentValues resolvedValues = null;
        boolean autowiring = (mbd.getResolvedAutowireMode() == AutowireCapableBeanFactory.AUTOWIRE_CONSTRUCTOR);
        int minTypeDiffWeight = Integer.MAX_VALUE;
        Set<Method> ambiguousFactoryMethods = null;

        int minNrOfArgs;
        if (explicitArgs != null) {
            minNrOfArgs = explicitArgs.length;
        } else {
            // We don't have arguments passed in programmatically, so we need to resolve the
            // arguments specified in the constructor arguments held in the bean definition.
            // <2.3> getBean() 没有传递参数，则需要解析保存在 BeanDefinition 构造函数中指定的参数
            if (mbd.hasConstructorArgumentValues()) {
                // 构造函数的参数
                ConstructorArgumentValues cargs = mbd.getConstructorArgumentValues();
                resolvedValues = new ConstructorArgumentValues();
                // 解析构造函数的参数
                // 将该 bean 的构造函数参数解析为 resolvedValues 对象，其中会涉及到其他 bean
                minNrOfArgs = resolveConstructorArguments(beanName, mbd, bw, cargs, resolvedValues);
            } else {
                minNrOfArgs = 0;
            }
        }

        LinkedList<UnsatisfiedDependencyException> causes = null; // 记录 UnsatisfiedDependencyException 异常的集合
        // 遍历 candidates 数组
        for (Method candidate : candidates) {
            // 方法体的参数
            Class<?>[] paramTypes = candidate.getParameterTypes();

            if (paramTypes.length >= minNrOfArgs) {
                // 保存参数的对象
                ArgumentsHolder argsHolder;

                // #getBean(...) 传递了参数
                if (explicitArgs != null) {
                    // Explicit arguments given -> arguments length must match exactly.
                    // 显示给定参数，参数长度必须完全匹配
                    if (paramTypes.length != explicitArgs.length) {
                        continue;
                    }
                    // 根据参数创建参数持有者 ArgumentsHolder 对象
                    argsHolder = new ArgumentsHolder(explicitArgs);
                } else {
                    // Resolved constructor arguments: type conversion and/or autowiring necessary.
                    // 为提供参数，解析构造参数
                    try {
                        String[] paramNames = null;
                        // 获取 ParameterNameDiscoverer 对象
                        // ParameterNameDiscoverer 是用于解析方法和构造函数的参数名称的接口，为参数名称探测器
                        ParameterNameDiscoverer pnd = this.beanFactory.getParameterNameDiscoverer();
                        // 获取指定构造函数的参数名称
                        if (pnd != null) {
                            paramNames = pnd.getParameterNames(candidate);
                        }
                        // 在已经解析的构造函数参数值的情况下，创建一个参数持有者 ArgumentsHolder 对象
                        argsHolder = createArgumentArray(beanName, mbd, resolvedValues, bw,
                                paramTypes, paramNames, candidate, autowiring, candidates.length == 1);
                    } catch (UnsatisfiedDependencyException ex) {
                        // 若发生 UnsatisfiedDependencyException 异常，添加到 causes 中。
                        if (logger.isTraceEnabled()) {
                            logger.trace("Ignoring factory method [" + candidate + "] of bean '" + beanName + "': " + ex);
                        }
                        // Swallow and try next overloaded factory method.
                        if (causes == null) {
                            causes = new LinkedList<>();
                        }
                        causes.add(ex);
                        continue; // continue ，继续执行
                    }
                }

                // isLenientConstructorResolution 判断解析构造函数的时候是否以宽松模式还是严格模式
                // 严格模式：解析构造函数时，必须所有的都需要匹配，否则抛出异常
                // 宽松模式：使用具有"最接近的模式"进行匹配
                // typeDiffWeight：类型差异权重
                int typeDiffWeight = (mbd.isLenientConstructorResolution() ?
                        argsHolder.getTypeDifferenceWeight(paramTypes) : argsHolder.getAssignabilityWeight(paramTypes));
                // Choose this factory method if it represents the closest match.
                // 代表最接近的类型匹配，则选择作为构造函数
                if (typeDiffWeight < minTypeDiffWeight) {
                    factoryMethodToUse = candidate;
                    argsHolderToUse = argsHolder;
                    argsToUse = argsHolder.arguments;
                    minTypeDiffWeight = typeDiffWeight;
                    ambiguousFactoryMethods = null;
                }
                // Find out about ambiguity: In case of the same type difference weight
                // for methods with the same number of parameters, collect such candidates
                // and eventually raise an ambiguity exception.
                // However, only perform that check in non-lenient constructor resolution mode,
                // and explicitly ignore overridden methods (with the same parameter signature).
                // 如果具有相同参数数量的方法具有相同的类型差异权重，则收集此类型选项
                // 但是，仅在非宽松构造函数解析模式下执行该检查，并显式忽略重写方法（具有相同的参数签名）
                else if (factoryMethodToUse != null && typeDiffWeight == minTypeDiffWeight &&
                        !mbd.isLenientConstructorResolution() &&
                        paramTypes.length == factoryMethodToUse.getParameterCount() &&
                        !Arrays.equals(paramTypes, factoryMethodToUse.getParameterTypes())) {
                    // 查找到多个可匹配的方法
                    if (ambiguousFactoryMethods == null) {
                        ambiguousFactoryMethods = new LinkedHashSet<>();
                        ambiguousFactoryMethods.add(factoryMethodToUse);
                    }
                    ambiguousFactoryMethods.add(candidate);
                }
            }
        }

        // 没有可执行的工厂方法，抛出异常
        if (factoryMethodToUse == null) {
            if (causes != null) {
                UnsatisfiedDependencyException ex = causes.removeLast();
                for (Exception cause : causes) {
                    this.beanFactory.onSuppressedException(cause);
                }
                throw ex;
            }
            List<String> argTypes = new ArrayList<>(minNrOfArgs);
            if (explicitArgs != null) {
                for (Object arg : explicitArgs) {
                    argTypes.add(arg != null ? arg.getClass().getSimpleName() : "null");
                }
            } else if (resolvedValues != null) {
                Set<ValueHolder> valueHolders = new LinkedHashSet<>(resolvedValues.getArgumentCount());
                valueHolders.addAll(resolvedValues.getIndexedArgumentValues().values());
                valueHolders.addAll(resolvedValues.getGenericArgumentValues());
                for (ValueHolder value : valueHolders) {
                    String argType = (value.getType() != null ? ClassUtils.getShortName(value.getType()) :
                            (value.getValue() != null ? value.getValue().getClass().getSimpleName() : "null"));
                    argTypes.add(argType);
                }
            }
            String argDesc = StringUtils.collectionToCommaDelimitedString(argTypes);
            throw new BeanCreationException(mbd.getResourceDescription(), beanName,
                    "No matching factory method found: " +
                    (mbd.getFactoryBeanName() != null ?
                        "factory bean '" + mbd.getFactoryBeanName() + "'; " : "") +
                    "factory method '" + mbd.getFactoryMethodName() + "(" + argDesc + ")'. " +
                    "Check that a method with the specified name " +
                    (minNrOfArgs > 0 ? "and arguments " : "") +
                    "exists and that it is " +
                    (isStatic ? "static" : "non-static") + ".");
        } else if (void.class == factoryMethodToUse.getReturnType()) {
            throw new BeanCreationException(mbd.getResourceDescription(), beanName,
                    "Invalid factory method '" + mbd.getFactoryMethodName() +
                    "': needs to have a non-void return type!");
        } else if (ambiguousFactoryMethods != null) {
            throw new BeanCreationException(mbd.getResourceDescription(), beanName,
                    "Ambiguous factory method matches found in bean '" + beanName + "' " +
                    "(hint: specify index/type/name arguments for simple parameters to avoid type ambiguities): " +
                    ambiguousFactoryMethods);
        }

        if (explicitArgs == null && argsHolderToUse != null) {
            // 将解析的构造函数加入缓存
            argsHolderToUse.storeCache(mbd, factoryMethodToUse);
        }
    }

    // 创建 Bean 对象，并设置到 bw 中
    bw.setBeanInstance(instantiate(beanName, mbd, factoryBean, factoryMethodToUse, argsToUse));
    return bw;
}

private Object instantiate(
        String beanName, RootBeanDefinition mbd, Constructor constructorToUse, Object[] argsToUse) {

    try {
        InstantiationStrategy strategy = this.beanFactory.getInstantiationStrategy();
        if (System.getSecurityManager() != null) {
            return AccessController.doPrivileged((PrivilegedAction<Object>) () ->
                    strategy.instantiate(mbd, beanName, this.beanFactory, constructorToUse, argsToUse),
                    this.beanFactory.getAccessControlContext());
        } else {
            return strategy.instantiate(mbd, beanName, this.beanFactory, constructorToUse, argsToUse);
        }
    } catch (Throwable ex) {
        throw new BeanCreationException(mbd.getResourceDescription(), beanName,
                "Bean instantiation via constructor failed", ex);
    }
}
```

- `#instantiateUsingFactoryMethod(...)` 方法，实在是**太大**了，处理细节感觉很复杂，LZ是硬着头皮看完的，中间断断续续的。吐槽这里的代码风格，完全不符合我们前面看的 Spring 代码风格。Spring 的一贯做法是将一个复杂逻辑进行拆分，分为多个细小的模块进行嵌套，每个模块负责一部分功能，模块与模块之间层层嵌套，上一层一般都是对下一层的总结和概括，这样就会使得每一层的逻辑变得清晰易懂。
- 回归到上面的方法体，虽然代码体量大，但是总体我们还是可看清楚这个方法要做的事情。**一句话概括就是**：确定工厂对象，然后获取构造函数和构造参数，最后调用 InstantiationStrategy 对象的 `#instantiate(RootBeanDefinition bd, String beanName, BeanFactory owner, Constructor<?> ctor, Object... args)` 方法，**来创建 Bean 实例**。

下面我们就这个句概括的话进行拆分并详细说明。

####### 1.2.1.1 确定工厂对象

> 对应 `<1>` 处。

首先获取工厂方法名：

- 若工厂方法名不为空，则调用 `AbstractAutowireCapableBeanFactory#getBean(String name)` 方法，获取工厂对象，
- 若为空，则可能为一个静态工厂，对于静态工厂则必须提供工厂类的全类名，同时设置 `factoryBean = null` 。

####### 1.2.1.2 构造参数确认

> 对应 `<2>` 处。

工厂对象确定后，则是确认构造参数。构造参数的确认主要分为**三种**情况：

- `explicitArgs` 参数
- 缓存中获取
- 配置文件中解析

######## 1.2.1.2.1 explicitArgs 参数

> 对应 `<2.1>` 处。

`explicitArgs` 参数，是我们调用 `#getBean(...)` 方法时传递进来的。一般该参数，该参数就是用于初始化 Bean 时所传递的参数。如果该参数不为空，则可以确定构造函数的参数就是它了。

######## 1.2.1.2.2 缓存中获取

> 对应 `<2.2>` 处。

在该方法的最后，我们会发现这样一段 `argsHolderToUse.storeCache(mbd, factoryMethodToUse)` 代码。这段代码主要是将构造函数、构造参数保存到缓存中，代码如下：

```
// ConstructorResolver.ArgumentsHolder.java

public void storeCache(RootBeanDefinition mbd, Executable constructorOrFactoryMethod) {
	synchronized (mbd.constructorArgumentLock) {
		mbd.resolvedConstructorOrFactoryMethod = constructorOrFactoryMethod;
		mbd.constructorArgumentsResolved = true;
		if (this.resolveNecessary) {
			mbd.preparedConstructorArguments = this.preparedArguments;
		} else {
			mbd.resolvedConstructorArguments = this.arguments;
		}
	}
}

// RootBeanDefinition.java

/** Common lock for the four constructor fields below. */
final Object constructorArgumentLock = new Object(); // 构造函数的缓存锁

/** Package-visible field for caching the resolved constructor or factory method. */
@Nullable
Executable resolvedConstructorOrFactoryMethod; // 缓存已经解析的构造函数或者工厂方法

/** Package-visible field that marks the constructor arguments as resolved. */
boolean constructorArgumentsResolved = false; // 标记字段，标记构造函数、参数已经解析了。默认为 `false` 。

/** Package-visible field for caching fully resolved constructor arguments. */
@Nullable
Object[] resolvedConstructorArguments; // 缓存已经解析的构造函数参数，包可见字段。

/** Package-visible field for caching partly prepared constructor arguments. */
@Nullable
Object[] preparedConstructorArguments;
```

其中涉及到的几个参数，都是跟构造函数、构造函数缓存有关的。如下：

- `constructorArgumentLock` ：构造函数的缓存锁。
- `resolvedConstructorOrFactoryMethod` ：缓存已经解析的构造函数或者工厂方法。
- `constructorArgumentsResolved` ：标记字段，标记构造函数、参数已经解析了。默认为 `false` 。
- `resolvedConstructorArguments` ：缓存已经解析的构造函数参数，包可见字段 。
- `preparedConstructorArguments`

------

所以，从缓存中获取就是提取这几个参数的值。代码如下：

```
// ConstructorResolver.java

// 没有指定，则尝试从配置文件中解析
Object[] argsToResolve = null;
// <2.2> 首先尝试从缓存中获取
synchronized (mbd.constructorArgumentLock) {
    // 获取缓存中的构造函数或者工厂方法
    factoryMethodToUse = (Method) mbd.resolvedConstructorOrFactoryMethod;
    if (factoryMethodToUse != null && mbd.constructorArgumentsResolved) {
        // Found a cached factory method...
        // 获取缓存中的构造参数
        argsToUse = mbd.resolvedConstructorArguments;
        if (argsToUse == null) {
            // 获取缓存中的构造函数参数的包可见字段
            argsToResolve = mbd.preparedConstructorArguments;
        }
    }
}
// 缓存中存在,则解析存储在 BeanDefinition 中的参数
// 如给定方法的构造函数 A(int ,int )，则通过此方法后就会把配置文件中的("1","1")转换为 (1,1)
// 缓存中的值可能是原始值也有可能是最终值
if (argsToResolve != null) {
    argsToUse = resolvePreparedArguments(beanName, mbd, bw, factoryMethodToUse, argsToResolve, true);
}
```

- 如果缓存中存在构造参数，则需要调用

   

  ```
  #resolvePreparedArguments(String beanName, RootBeanDefinition mbd, BeanWrapper bw, Executable executable, Object[] argsToResolve, boolean fallback)
  ```

   

  方法，进行转换。

  - 本文不详细解析该方法，感兴趣的胖友，可以自己查看。

- 因为缓存中的值有可能是最终值，**也有可能不是最终值**。比如我们构造函数中的类型为 Integer 类型的 1 ，但是原始的参数类型有可能是 String 类型的 `"1"` ，所以即便是从缓存中得到了构造参数，也**需要经过一番的类型转换确保参数类型完全对应**。

######## 1.2.1.2.3 配置文件中解析

> 对应 `<2.3>` 处。

即没有通过传递参数的方式传递构造参数，缓存中也没有，那就只能通过解析配置文件获取构造参数了。

在 bean 解析类的博文中我们了解了，配置文件中的信息都会转换到 BeanDefinition 实例对象中，所以配置文件中的参数可以直接通过 BeanDefinition 对象获取。代码如下：

```
// AbstractAutowireCapableBeanFactory.java

// <2.3> getBean() 没有传递参数，则需要解析保存在 BeanDefinition 构造函数中指定的参数
if (mbd.hasConstructorArgumentValues()) {
    // <2.3.1> 构造函数的参数
    ConstructorArgumentValues cargs = mbd.getConstructorArgumentValues();
    resolvedValues = new ConstructorArgumentValues();
    // <2.3.2> 解析构造函数的参数
    // 将该 bean 的构造函数参数解析为 resolvedValues 对象，其中会涉及到其他 bean
    minNrOfArgs = resolveConstructorArguments(beanName, mbd, bw, cargs, resolvedValues);
}
```

- `<2.3.1>` ，通过 BeanDefinition 的 `#getConstructorArgumentValues()` 方法，就可以获取构造信息了。
- `<2.3.2>` ，有了构造信息就可以获取相关的参数值信息了，获取的参数信息包括直接值和引用，这一步骤的处理交由 `#resolveConstructorArguments(String beanName, RootBeanDefinition mbd, BeanWrapper bw, ConstructorArgumentValues cargs, ConstructorArgumentValues resolvedValues)` 方法来完成。该方法会将构造参数信息解析为 `resolvedValues` 对象 并返回解析到的参数个数 `minNrOfArgs` 。

####### 1.2.1.3 构造函数

> 对应 `<3>` 处。

确定构造参数后，下一步则是确定构造函数。

- 第一步，是通过 `#getCandidateMethods()` 方法，获取所有的构造方法，同时对构造方法进行刷选。
- 然后，在对其进行排序处理（`AutowireUtils.sortFactoryMethods(candidates)`）。排序的主要目的，是为了能够**更加方便的**找到匹配的构造函数，因为构造函数的确认是根据参数个数确认的。排序的规则是：先按照 `public` / 非 `public` 构造函数**升序**，再按照构造参数数量**降序**。

通过迭代 `candidates`（包含了所有要匹配的构造函数）的方式，依次比较其参数：

- 如果显示提供了参数（`explicitArgs != null`），则直接比较两者**长度**是否相等，如果相等则表示找到了，否则继续比较。
- 如果没有显示提供参数，则需要获取 `org.springframework.core.ParameterNameDiscoverer` 对象。该对象为参数名称探测器，主要用于发现方法和构造函数的参数名称。

将参数包装成 `ConstructorResolver.ArgumentsHolder` 对象。该对象用于保存参数，我们称之为参数持有者。当将对象包装成 ArgumentsHolder 对象后，我们就可以通过它来进行构造函数**匹配**。匹配分为严格模式和宽松模式：

- 严格模式：解析构造函数时，必须所有参数都需要匹配，否则抛出异常。
- 宽松模式：使用具有”最接近的模式”进行匹配。

判断的依据是根据 BeanDefinition 的 `isLenientConstructorResolution` 属性（该参数是我们在构造 AbstractBeanDefinition 对象是传递的）来获取类型差异权重（`typeDiffWeight`） 的。

- 如果 `typeDiffWeight < minTypeDiffWeight` ，则代表“最接近的模式”，选择其作为构造函数。
- 否则，只有两者具有相同的参数数量，且类型差异权重相等才会纳入考虑范围。

至此，构造函数已经确认了。

####### 1.2.1.4 创建 bean 实例

> 对应 `<4>` 处。

工厂对象、构造函数、构造参数都已经确认了，则最后一步就是调用 `org.springframework.beans.factory.support.InstantiationStrategy` 对象的 `#instantiate(RootBeanDefinition bd, String beanName, BeanFactory owner, Object factoryBean, final Method factoryMethod, @Nullable Object... args)` 方法，来创建 bean 实例。代码如下：

```
// SimpleInstantiationStrategy.java

/**
 * 线程变量，正在创建 Bean 的 Method 对象
 */
private static final ThreadLocal<Method> currentlyInvokedFactoryMethod = new ThreadLocal<>();

@Override
public Object instantiate(RootBeanDefinition bd, @Nullable String beanName, BeanFactory owner,
        @Nullable Object factoryBean, final Method factoryMethod, Object... args) {
    try {
        // 设置 Method 可访问
        if (System.getSecurityManager() != null) {
            AccessController.doPrivileged((PrivilegedAction<Object>) () -> {
                ReflectionUtils.makeAccessible(factoryMethod);
                return null;
            });
        } else {
            ReflectionUtils.makeAccessible(factoryMethod);
        }

        // 获得原 Method 对象
        Method priorInvokedFactoryMethod = currentlyInvokedFactoryMethod.get();
        try {
            // 设置新的 Method 对象，到 currentlyInvokedFactoryMethod 中
            currentlyInvokedFactoryMethod.set(factoryMethod);
            // <x> 创建 Bean 对象
            Object result = factoryMethod.invoke(factoryBean, args);
            // 未创建，则创建 NullBean 对象
            if (result == null) {
                result = new NullBean();
            }
            return result;
        } finally {
            // 设置老的 Method 对象，到 currentlyInvokedFactoryMethod 中
            if (priorInvokedFactoryMethod != null) {
                currentlyInvokedFactoryMethod.set(priorInvokedFactoryMethod);
            } else {
                currentlyInvokedFactoryMethod.remove();
            }
        }
    } catch (IllegalArgumentException ex) {
        throw new BeanInstantiationException(factoryMethod,
                "Illegal arguments to factory method '" + factoryMethod.getName() + "'; " +
                "args: " + StringUtils.arrayToCommaDelimitedString(args), ex);
    } catch (IllegalAccessException ex) {
        throw new BeanInstantiationException(factoryMethod,
                "Cannot access factory method '" + factoryMethod.getName() + "'; is it public?", ex);
    } catch (InvocationTargetException ex) {
        String msg = "Factory method '" + factoryMethod.getName() + "' threw exception";
        if (bd.getFactoryBeanName() != null && owner instanceof ConfigurableBeanFactory &&
                ((ConfigurableBeanFactory) owner).isCurrentlyInCreation(bd.getFactoryBeanName())) {
            msg = "Circular reference involving containing bean '" + bd.getFactoryBeanName() + "' - consider " +
                    "declaring the factory method as static for independence from its containing instance. " + msg;
        }
        throw new BeanInstantiationException(factoryMethod, msg, ex.getTargetException());
    }
}
```

- 核心的部分，在于 `<x>` 处，利用 Java 反射执行工厂方法并返回创建好的实例，也就是这段代码：

  ```
  // SimpleInstantiationStrategy.java
  
  Object result = factoryMethod.invoke(factoryBean, args);
  ```

#### 1.2.2 独白

到这里 `#instantiateUsingFactoryMethod(...)` 方法，已经分析完毕了。这里， LZ 有些题外话需要说下，**看源码真心是一个痛苦的过程，尤其是复杂的源码，比如这个方法我看了三天才弄清楚点皮毛，当然这里跟 LZ 的智商有些关系（智商捉急 ┭┮﹏┭┮），写这篇博客也花了五天时间才写完（最后截稿日为：2018.08.10 01:23:49），所以每一个坚持写博客的都是折翼的天使，值得各位尊敬**

`#createBeanInstance(String beanName, RootBeanDefinition mbd, Object[] args)` 方法，用于实例化 Bean 对象。它会根据不同情况，选择不同的实例化策略来完成 Bean 的初始化，主要包括：

- Supplier 回调：`#obtainFromSupplier(final String beanName, final RootBeanDefinition mbd)` 方法。
- 工厂方法初始化：`#instantiateUsingFactoryMethod(String beanName, RootBeanDefinition mbd, @Nullable Object[] explicitArgs)` 方法。
- 构造函数自动注入初始化：`#autowireConstructor(final String beanName, final RootBeanDefinition mbd, Constructor<?>[] chosenCtors, final Object[] explicitArgs)` 方法。
- 默认构造函数注入：`#instantiateBean(final String beanName, final RootBeanDefinition mbd)` 方法。

在上篇博客([《【死磕 Spring】—— IoC 之加载 bean：创建 bean（二）》](http://svip.iocoder.cn/Spring/IoC-get-Bean-createBean-2)) 中，分析了**前两种** Supplier 回调和工厂方法初始化。这篇博文，分析**后两种**构造函数注入。

## 1. autowireConstructor

这个初始化方法，我们可以简单理解为是**带有参数的构造方法**，来初始化 Bean 对象。代码段如下：

```
// AbstractAutowireCapableBeanFactory.java

protected BeanWrapper autowireConstructor(String beanName, RootBeanDefinition mbd, @Nullable Constructor<?>[] ctors, @Nullable Object[] explicitArgs) {
    return new ConstructorResolver(this).autowireConstructor(beanName, mbd, ctors, explicitArgs);
}

// ConstructorResolver.java

public BeanWrapper autowireConstructor(String beanName, RootBeanDefinition mbd,
        @Nullable Constructor<?>[] chosenCtors, @Nullable Object[] explicitArgs) {
    // 封装 BeanWrapperImpl 对象，并完成初始化
    BeanWrapperImpl bw = new BeanWrapperImpl();
    this.beanFactory.initBeanWrapper(bw);

    // 获得 constructorToUse、argsHolderToUse、argsToUse
    Constructor<?> constructorToUse = null; // 构造函数
    ArgumentsHolder argsHolderToUse = null; // 构造参数
    Object[] argsToUse = null; // 构造参数

    // 确定构造参数
    // 如果 getBean() 已经传递，则直接使用
    if (explicitArgs != null) {
        argsToUse = explicitArgs;
    } else {
        // 尝试从缓存中获取
        Object[] argsToResolve = null;
        synchronized (mbd.constructorArgumentLock) {
            // 缓存中的构造函数或者工厂方法
            constructorToUse = (Constructor<?>) mbd.resolvedConstructorOrFactoryMethod;
            if (constructorToUse != null && mbd.constructorArgumentsResolved) {
                // Found a cached constructor...
                // 缓存中的构造参数
                argsToUse = mbd.resolvedConstructorArguments;
                if (argsToUse == null) {
                    argsToResolve = mbd.preparedConstructorArguments;
                }
            }
        }
        // 缓存中存在,则解析存储在 BeanDefinition 中的参数
        // 如给定方法的构造函数 A(int ,int )，则通过此方法后就会把配置文件中的("1","1")转换为 (1,1)
        // 缓存中的值可能是原始值也有可能是最终值
        if (argsToResolve != null) {
            argsToUse = resolvePreparedArguments(beanName, mbd, bw, constructorToUse, argsToResolve, true);
        }
    }

    // 没有缓存，则尝试从配置文件中获取参数
    if (constructorToUse == null || argsToUse == null) {
        // Take specified constructors, if any.
        // 如果 chosenCtors 未传入，则获取构造方法们
        Constructor<?>[] candidates = chosenCtors;
        if (candidates == null) {
            Class<?> beanClass = mbd.getBeanClass();
            try {
                candidates = (mbd.isNonPublicAccessAllowed() ?
                        beanClass.getDeclaredConstructors() : beanClass.getConstructors());
            } catch (Throwable ex) {
                throw new BeanCreationException(mbd.getResourceDescription(), beanName,
                        "Resolution of declared constructors on bean Class [" + beanClass.getName() +
                        "] from ClassLoader [" + beanClass.getClassLoader() + "] failed", ex);
            }
        }

        // TODO 芋艿 创建 Bean
        if (candidates.length == 1 && explicitArgs == null && !mbd.hasConstructorArgumentValues()) {
            Constructor<?> uniqueCandidate = candidates[0];
            if (uniqueCandidate.getParameterCount() == 0) {
                synchronized (mbd.constructorArgumentLock) {
                    mbd.resolvedConstructorOrFactoryMethod = uniqueCandidate;
                    mbd.constructorArgumentsResolved = true;
                    mbd.resolvedConstructorArguments = EMPTY_ARGS;
                }
                bw.setBeanInstance(instantiate(beanName, mbd, uniqueCandidate, EMPTY_ARGS));
                return bw;
            }
        }

        // 是否需要解析构造器
        // Need to resolve the constructor.
        boolean autowiring = (chosenCtors != null ||
                mbd.getResolvedAutowireMode() == AutowireCapableBeanFactory.AUTOWIRE_CONSTRUCTOR);
        // 用于承载解析后的构造函数参数的值
        ConstructorArgumentValues resolvedValues = null;
        int minNrOfArgs;
        if (explicitArgs != null) {
            minNrOfArgs = explicitArgs.length;
        } else {
            // 从 BeanDefinition 中获取构造参数，也就是从配置文件中提取构造参数
            ConstructorArgumentValues cargs = mbd.getConstructorArgumentValues();
            resolvedValues = new ConstructorArgumentValues();
            // 解析构造函数的参数
            // 将该 bean 的构造函数参数解析为 resolvedValues 对象，其中会涉及到其他 bean
            minNrOfArgs = resolveConstructorArguments(beanName, mbd, bw, cargs, resolvedValues);
        }

        // 对构造函数进行排序处理
        // public 构造函数优先参数数量降序，非public 构造函数参数数量降序
        AutowireUtils.sortConstructors(candidates);

        // 最小参数类型权重
        int minTypeDiffWeight = Integer.MAX_VALUE;
        Set<Constructor<?>> ambiguousConstructors = null;
        LinkedList<UnsatisfiedDependencyException> causes = null;

        // 迭代所有构造函数
        for (Constructor<?> candidate : candidates) {
            // 获取该构造函数的参数类型
            Class<?>[] paramTypes = candidate.getParameterTypes();

            // 如果已经找到选用的构造函数或者需要的参数个数小于当前的构造函数参数个数，则终止。
            // 因为，已经按照参数个数降序排列了
            if (constructorToUse != null && argsToUse.length > paramTypes.length) {
                // Already found greedy constructor that can be satisfied ->
                // do not look any further, there are only less greedy constructors left.
                break;
            }
            // 参数个数不等，继续
            if (paramTypes.length < minNrOfArgs) {
                continue;
            }

            // 参数持有者 ArgumentsHolder 对象
            ArgumentsHolder argsHolder;
            if (resolvedValues != null) {
                try {
                    // 注释上获取参数名称
                    String[] paramNames = ConstructorPropertiesChecker.evaluate(candidate, paramTypes.length);
                    if (paramNames == null) {
                        // 获取构造函数、方法参数的探测器
                        ParameterNameDiscoverer pnd = this.beanFactory.getParameterNameDiscoverer();
                        if (pnd != null) {
                            // 通过探测器获取构造函数的参数名称
                            paramNames = pnd.getParameterNames(candidate);
                        }
                    }
                    // 根据构造函数和构造参数，创建参数持有者 ArgumentsHolder 对象
                    argsHolder = createArgumentArray(beanName, mbd, resolvedValues, bw, paramTypes, paramNames,
                            getUserDeclaredConstructor(candidate), autowiring, candidates.length == 1);
                } catch (UnsatisfiedDependencyException ex) {
                    // 若发生 UnsatisfiedDependencyException 异常，添加到 causes 中。
                    if (logger.isTraceEnabled()) {
                        logger.trace("Ignoring constructor [" + candidate + "] of bean '" + beanName + "': " + ex);
                    }
                    // Swallow and try next constructor.
                    if (causes == null) {
                        causes = new LinkedList<>();
                    }
                    causes.add(ex);
                    continue; // continue ，继续执行
                }
            } else {
                // continue 构造函数没有参数
                // Explicit arguments given -> arguments length must match exactly.
                if (paramTypes.length != explicitArgs.length) {
                    continue;
                }
                // 根据 explicitArgs ，创建 ArgumentsHolder 对象
                argsHolder = new ArgumentsHolder(explicitArgs);
            }

            // isLenientConstructorResolution 判断解析构造函数的时候是否以宽松模式还是严格模式
            // 严格模式：解析构造函数时，必须所有的都需要匹配，否则抛出异常
            // 宽松模式：使用具有"最接近的模式"进行匹配
            // typeDiffWeight：类型差异权重
            int typeDiffWeight = (mbd.isLenientConstructorResolution() ?
                    argsHolder.getTypeDifferenceWeight(paramTypes) : argsHolder.getAssignabilityWeight(paramTypes));
            // Choose this constructor if it represents the closest match.
            // 如果它代表着当前最接近的匹配则选择其作为构造函数
            if (typeDiffWeight < minTypeDiffWeight) {
                constructorToUse = candidate;
                argsHolderToUse = argsHolder;
                argsToUse = argsHolder.arguments;
                minTypeDiffWeight = typeDiffWeight;
                ambiguousConstructors = null;
            } else if (constructorToUse != null && typeDiffWeight == minTypeDiffWeight) {
                if (ambiguousConstructors == null) {
                    ambiguousConstructors = new LinkedHashSet<>();
                    ambiguousConstructors.add(constructorToUse);
                }
                ambiguousConstructors.add(candidate);
            }
        }

        // 没有可执行的工厂方法，抛出异常
        if (constructorToUse == null) {
            if (causes != null) {
                UnsatisfiedDependencyException ex = causes.removeLast();
                for (Exception cause : causes) {
                    this.beanFactory.onSuppressedException(cause);
                }
                throw ex;
            }
            throw new BeanCreationException(mbd.getResourceDescription(), beanName,
                    "Could not resolve matching constructor " +
                    "(hint: specify index/type/name arguments for simple parameters to avoid type ambiguities)");
        } else if (ambiguousConstructors != null && !mbd.isLenientConstructorResolution()) {
            throw new BeanCreationException(mbd.getResourceDescription(), beanName,
                    "Ambiguous constructor matches found in bean '" + beanName + "' " +
                    "(hint: specify index/type/name arguments for simple parameters to avoid type ambiguities): " +
                    ambiguousConstructors);
        }

        if (explicitArgs == null) {
            // 将解析的构造函数加入缓存
            argsHolderToUse.storeCache(mbd, constructorToUse);
        }
    }

    // 创建 Bean 对象，并设置到 bw 中
    bw.setBeanInstance(instantiate(beanName, mbd, constructorToUse, argsToUse));
    return bw;
}
```

- 代码与 `#instantiateUsingFactoryMethod(String beanName, RootBeanDefinition mbd, @Nullable Object[] explicitArgs)` 方法，一样，又长又难懂。但是如果理解了 `#instantiateUsingFactoryMethod(...)` 方法的初始化 bean 的过程，那么 `#autowireConstructor(...)` 方法，**也不存在什么难的地方了**。
- 一句话概括：首先确定构造函数参数、构造函数，然后调用相应的初始化策略进行 bean 的初始化。关于如何确定构造函数、构造参数，该部分逻辑和 `#instantiateUsingFactoryMethod(...)` 方法，基本一致。所以这里不再重复阐述了，具体过程请移步[【死磕 Spring】—— IoC 之加载 bean：创建 bean（二）](http://svip.iocoder.cn/Spring/IoC-get-Bean-createBean-2)，这里我们重点分析初始化策略。

#### 1.1 instantiate

```
// BeanUtils.java

public static <T> T instantiateClass(Constructor<T> ctor, Object... args) throws BeanInstantiationException {
    Assert.notNull(ctor, "Constructor must not be null");
    try {
        // 设置构造方法，可访问
        ReflectionUtils.makeAccessible(ctor);
        // 使用构造方法，创建对象
        return (KotlinDetector.isKotlinReflectPresent() && KotlinDetector.isKotlinType(ctor.getDeclaringClass()) ?
                KotlinDelegate.instantiateClass(ctor, args) : ctor.newInstance(args));
    // 各种异常的翻译，最终统一抛出 BeanInstantiationException 异常
    } catch (InstantiationException ex) {
        throw new BeanInstantiationException(ctor, "Is it an abstract class?", ex);
    } catch (IllegalAccessException ex) {
        throw new BeanInstantiationException(ctor, "Is the constructor accessible?", ex);
    } catch (IllegalArgumentException ex) {
        throw new BeanInstantiationException(ctor, "Illegal arguments for constructor", ex);
    } catch (InvocationTargetException ex) {
        throw new BeanInstantiationException(ctor, "Constructor threw exception", ex.getTargetException());
    }
}
```

- `<1>` 首先，是获取实例化 Bean 的策略 InstantiationStrategy 对象。

- `<2>` 然后，调用其 `#instantiate(RootBeanDefinition bd, String beanName, BeanFactory owner, Constructor<?> ctor, Object... args)` 方法，该方法在 SimpleInstantiationStrategy 中实现。代码如下：

  ```
  // SimpleInstantiationStrategy.java
  
  @Override
  public Object instantiate(RootBeanDefinition bd, @Nullable String beanName, BeanFactory owner,
          final Constructor<?> ctor, Object... args) {
      // <x1> 没有覆盖，直接使用反射实例化即可
      if (!bd.hasMethodOverrides()) {
          if (System.getSecurityManager() != null) {
              // 设置构造方法，可访问
              // use own privileged to change accessibility (when security is on)
              AccessController.doPrivileged((PrivilegedAction<Object>) () -> {
                  ReflectionUtils.makeAccessible(ctor);
                  return null;
              });
          }
          // 通过 BeanUtils 直接使用构造器对象实例化 Bean 对象
          return BeanUtils.instantiateClass(ctor, args);
      } else {
          // <x2> 生成 CGLIB 创建的子类对象
          return instantiateWithMethodInjection(bd, beanName, owner, ctor, args);
      }
  }
  ```

  - `<x1>` 如果该 bean 没有配置 `lookup-method`、`replaced-method` 标签或者 `@Lookup` 注解，则直接通过**反射**的方式实例化 Bean 对象即可，方便快捷。详细解析，见 [「1.1.1 反射创建 Bean 对象」](http://svip.iocoder.cn/Spring/IoC-get-Bean-createBean-3/#) 中。
  - `<x2>` 但是，如果存在需要覆盖的方法或者动态替换的方法时，则需要使用 CGLIB 进行动态代理，因为可以在创建代理的同时将动态方法织入类中。详细解析，见 [「1.1.2 CGLIB 创建 Bean 对象」](http://svip.iocoder.cn/Spring/IoC-get-Bean-createBean-3/#) 中。

###### 1.1.1 反射创建 Bean 对象

调用工具类 BeanUtils 的 `#instantiateClass(Constructor<T> ctor, Object... args)` 方法，完成反射工作，创建对象。代码如下：

```
// BeanUtils.java

public static <T> T instantiateClass(Constructor<T> ctor, Object... args) throws BeanInstantiationException {
    Assert.notNull(ctor, "Constructor must not be null");
    try {
        // 设置构造方法，可访问
        ReflectionUtils.makeAccessible(ctor);
        // 使用构造方法，创建对象
        return (KotlinDetector.isKotlinReflectPresent() && KotlinDetector.isKotlinType(ctor.getDeclaringClass()) ?
                KotlinDelegate.instantiateClass(ctor, args) : ctor.newInstance(args));
    // 各种异常的翻译，最终统一抛出 BeanInstantiationException 异常
    } catch (InstantiationException ex) {
        throw new BeanInstantiationException(ctor, "Is it an abstract class?", ex);
    } catch (IllegalAccessException ex) {
        throw new BeanInstantiationException(ctor, "Is the constructor accessible?", ex);
    } catch (IllegalArgumentException ex) {
        throw new BeanInstantiationException(ctor, "Illegal arguments for constructor", ex);
    } catch (InvocationTargetException ex) {
        throw new BeanInstantiationException(ctor, "Constructor threw exception", ex.getTargetException());
    }
}
```

###### 1.1.2 CGLIB 创建 Bean 对象

```
// SimpleInstantiationStrategy.java

protected Object instantiateWithMethodInjection(RootBeanDefinition bd, @Nullable String beanName, BeanFactory owner) {
	throw new UnsupportedOperationException("Method Injection not supported in SimpleInstantiationStrategy");
}
```

- 方法默认是**没有实现**的，具体过程由其子类 `org.springframework.beans.factory.support.CglibSubclassingInstantiationStrategy` 来实现。代码如下：

  ```
  // CglibSubclassingInstantiationStrategy.java
  
  @Override
  protected Object instantiateWithMethodInjection(RootBeanDefinition bd, @Nullable String beanName, BeanFactory owner) {
      return instantiateWithMethodInjection(bd, beanName, owner, null);
  }
  @Override
  protected Object instantiateWithMethodInjection(RootBeanDefinition bd, @Nullable String beanName, BeanFactory owner, @Nullable Constructor<?> ctor, Object... args) {
      // Must generate CGLIB subclass...
      // 通过CGLIB生成一个子类对象
      return new CglibSubclassCreator(bd, owner).instantiate(ctor, args);
  }
  ```

  - 创建一个 CglibSubclassCreator 对象，后调用其 `#instantiate(Constructor<?> ctor, Object... args)` 方法，生成其子类对象。代码如下：

    ```
    // CglibSubclassingInstantiationStrategy.java
    
    public Object instantiate(@Nullable Constructor<?> ctor, Object... args) {
        // 通过 Cglib 创建一个代理类
        Class<?> subclass = createEnhancedSubclass(this.beanDefinition);
        Object instance;
        // 没有构造器，通过 BeanUtils 使用默认构造器创建一个bean实例
        if (ctor == null) {
            instance = BeanUtils.instantiateClass(subclass);
        } else {
            try {
                // 获取代理类对应的构造器对象，并实例化 bean
                Constructor<?> enhancedSubclassConstructor = subclass.getConstructor(ctor.getParameterTypes());
                instance = enhancedSubclassConstructor.newInstance(args);
            } catch (Exception ex) {
                throw new BeanInstantiationException(this.beanDefinition.getBeanClass(),
                        "Failed to invoke constructor for CGLIB enhanced subclass [" + subclass.getName() + "]", ex);
            }
        }
        // SPR-10785: set callbacks directly on the instance instead of in the
        // enhanced class (via the Enhancer) in order to avoid memory leaks.
        // 为了避免 memory leaks 异常，直接在 bean 实例上设置回调对象
        Factory factory = (Factory) instance;
        factory.setCallbacks(new Callback[] {NoOp.INSTANCE,
                new LookupOverrideMethodInterceptor(this.beanDefinition, this.owner),
                new ReplaceOverrideMethodInterceptor(this.beanDefinition, this.owner)});
        return instance;
    }
    ```

    - 到这类 CGLIB 的方式分析完毕了，当然这里还没有具体分析 CGLIB 生成子类的详细过程，具体的过程等后续分析 AOP 的时候再详细地介绍。

## 2. instantiateBean

```
// AbstractAutowireCapableBeanFactory.java

protected BeanWrapper instantiateBean(final String beanName, final RootBeanDefinition mbd) {
    try {
        Object beanInstance;
        final BeanFactory parent = this;
        // 安全模式
        if (System.getSecurityManager() != null) {
            beanInstance = AccessController.doPrivileged((PrivilegedAction<Object>) () ->
                    // 获得 InstantiationStrategy 对象，并使用它，创建 Bean 对象
                    getInstantiationStrategy().instantiate(mbd, beanName, parent),
                    getAccessControlContext());
        } else {
            // 获得 InstantiationStrategy 对象，并使用它，创建 Bean 对象
            beanInstance = getInstantiationStrategy().instantiate(mbd, beanName, parent);
        }
        // 封装 BeanWrapperImpl  并完成初始化
        BeanWrapper bw = new BeanWrapperImpl(beanInstance);
        initBeanWrapper(bw);
        return bw;
    } catch (Throwable ex) {
        throw new BeanCreationException(
                mbd.getResourceDescription(), beanName, "Instantiation of bean failed", ex);
    }
}
```

- 这个方法,相比于 `#instantiateUsingFactoryMethod(...)` 、 `#autowireConstructor(...)` 方法，**实在是太简单了**，因为，它没有参数，所以不需要确认经过复杂的过来来确定构造器、构造参数，所以这里就不过多阐述了。

#### 2.1 instantiate

```
// SimpleInstantiationStrategy.java

@Override
public Object instantiate(RootBeanDefinition bd, @Nullable String beanName, BeanFactory owner) {
    // Don't override the class with CGLIB if no overrides.
    // 没有覆盖，直接使用反射实例化即可
    if (!bd.hasMethodOverrides()) {
        Constructor<?> constructorToUse;
        synchronized (bd.constructorArgumentLock) {
            // 获得构造方法 constructorToUse
            constructorToUse = (Constructor<?>) bd.resolvedConstructorOrFactoryMethod;
            if (constructorToUse == null) {
                final Class<?> clazz = bd.getBeanClass();
                // 如果是接口，抛出 BeanInstantiationException 异常
                if (clazz.isInterface()) {
                    throw new BeanInstantiationException(clazz, "Specified class is an interface");
                }
                try {
                    // 从 clazz 中，获得构造方法
                    if (System.getSecurityManager() != null) { // 安全模式
                        constructorToUse = AccessController.doPrivileged(
                                (PrivilegedExceptionAction<Constructor<?>>) clazz::getDeclaredConstructor);
                    } else {
                        constructorToUse =  clazz.getDeclaredConstructor();
                    }
                    // 标记 resolvedConstructorOrFactoryMethod 属性
                    bd.resolvedConstructorOrFactoryMethod = constructorToUse;
                } catch (Throwable ex) {
                    throw new BeanInstantiationException(clazz, "No default constructor found", ex);
                }
            }
        }
        // 通过 BeanUtils 直接使用构造器对象实例化 Bean 对象
        return BeanUtils.instantiateClass(constructorToUse);
    } else {
        // Must generate CGLIB subclass.
        // 生成 CGLIB 创建的子类对象
        return instantiateWithMethodInjection(bd, beanName, owner);
    }
}
```

## 3. 小结

对于 `#createBeanInstance(...)` 方法而言，他就是**选择合适实例化策略**来为 bean 创建实例对象，具体的策略有：

- Supplier 回调方式
- 工厂方法初始化
- 构造函数自动注入初始化
- 默认构造函数注入。

其中，工厂方法初始化和构造函数自动注入初始化两种方式**最为复杂**，主要是因为构造函数和构造参数的不确定性，Spring 需要花大量的精力来确定构造函数和构造参数，如果确定了则好办，直接选择实例化策略即可。

当然，在实例化的时候会根据是否有需要覆盖或者动态替换掉的方法，因为存在覆盖或者织入的话需要创建动态代理将方法织入，这个时候就只能选择 CGLIB 的方式来实例化，否则直接利用反射的方式即可，方便快捷。

------

到这里 `#createBeanInstance(...)` 的过程就已经分析完毕了，下篇介绍 `#doCreateBean(...)` 方法中的第二个过程：**循环依赖的处理**。其实，在整个 bean 的加载过程中都涉及到了循环依赖的处理，所以下面分析并不是仅仅只针对 `#doCreateBean(...)` 方法中的循环依赖处理，**而是 Spring 在整个 IoC 体系中是如何解决循环依赖的**。

# 7、创建 Bean（三）属性填充

`#doCreateBean(...)` 方法，主要用于完成 bean 的创建和初始化工作，我们可以将其分为四个过程：

- `#createBeanInstance(String beanName, RootBeanDefinition mbd, Object[] args)` 方法，实例化 bean 。
- 循环依赖的处理。
- `#populateBean(String beanName, RootBeanDefinition mbd, BeanWrapper bw)` 方法，进行属性填充。
- `#initializeBean(final String beanName, final Object bean, RootBeanDefinition mbd)` 方法，初始化 Bean 。

第一个过程，实例化 bean 已经在前面两篇博客分析完毕了。
这篇博客开始，分析属性填充，也就是 `#populateBean(...)` 方法。该函数的作用是将 BeanDefinition 中的属性值赋值给 BeanWrapper 实例对象(对于 BeanWrapper ，我们后续专门写文分析)。

## 1. populateBean

```
// AbstractAutowireCapableBeanFactory.java

protected void populateBean(String beanName, RootBeanDefinition mbd, @Nullable BeanWrapper bw) {
    // 没有实例化对象
    if (bw == null) {
        // 有属性，则抛出 BeanCreationException 异常
        if (mbd.hasPropertyValues()) {
            throw new BeanCreationException(
                    mbd.getResourceDescription(), beanName, "Cannot apply property values to null instance");
            // 没有属性，直接 return 返回
        } else {
            // Skip property population phase for null instance.
            return;
        }
    }

    // <1> 在设置属性之前给 InstantiationAwareBeanPostProcessors 最后一次改变 bean 的机会
    // Give any InstantiationAwareBeanPostProcessors the opportunity to modify the
    // state of the bean before properties are set. This can be used, for example,
    // to support styles of field injection.
    boolean continueWithPropertyPopulation = true;
    if (!mbd.isSynthetic()  // bean 不是"合成"的，即未由应用程序本身定义
            && hasInstantiationAwareBeanPostProcessors()) { // 是否持有 InstantiationAwareBeanPostProcessor
        // 迭代所有的 BeanPostProcessors
        for (BeanPostProcessor bp : getBeanPostProcessors()) {
            if (bp instanceof InstantiationAwareBeanPostProcessor) { // 如果为 InstantiationAwareBeanPostProcessor
                InstantiationAwareBeanPostProcessor ibp = (InstantiationAwareBeanPostProcessor) bp;
                // 返回值为是否继续填充 bean
                // postProcessAfterInstantiation：如果应该在 bean上面设置属性则返回 true，否则返回 false
                // 一般情况下，应该是返回true 。
                // 返回 false 的话，将会阻止在此 Bean 实例上调用任何后续的 InstantiationAwareBeanPostProcessor 实例。
                if (!ibp.postProcessAfterInstantiation(bw.getWrappedInstance(), beanName)) {
                    continueWithPropertyPopulation = false;
                    break;
                }
            }
        }
    }
    // 如果后续处理器发出停止填充命令，则终止后续操作
    if (!continueWithPropertyPopulation) {
        return;
    }

    // bean 的属性值
    PropertyValues pvs = (mbd.hasPropertyValues() ? mbd.getPropertyValues() : null);

    // <2> 自动注入
    if (mbd.getResolvedAutowireMode() == AUTOWIRE_BY_NAME || mbd.getResolvedAutowireMode() == AUTOWIRE_BY_TYPE) {
        // 将 PropertyValues 封装成 MutablePropertyValues 对象
        // MutablePropertyValues 允许对属性进行简单的操作，并提供构造函数以支持Map的深度复制和构造。
        MutablePropertyValues newPvs = new MutablePropertyValues(pvs);
        // Add property values based on autowire by name if applicable.
        // 根据名称自动注入
        if (mbd.getResolvedAutowireMode() == AUTOWIRE_BY_NAME) {
            autowireByName(beanName, mbd, bw, newPvs);
        }
        // Add property values based on autowire by type if applicable.
        // 根据类型自动注入
        if (mbd.getResolvedAutowireMode() == AUTOWIRE_BY_TYPE) {
            autowireByType(beanName, mbd, bw, newPvs);
        }
        pvs = newPvs;
    }

    // 是否已经注册了 InstantiationAwareBeanPostProcessors
    boolean hasInstAwareBpps = hasInstantiationAwareBeanPostProcessors();
    // 是否需要进行【依赖检查】
    boolean needsDepCheck = (mbd.getDependencyCheck() != AbstractBeanDefinition.DEPENDENCY_CHECK_NONE);

    // <3> BeanPostProcessor 处理
    PropertyDescriptor[] filteredPds = null;
    if (hasInstAwareBpps) {
        if (pvs == null) {
            pvs = mbd.getPropertyValues();
        }
        // 遍历 BeanPostProcessor 数组
        for (BeanPostProcessor bp : getBeanPostProcessors()) {
            if (bp instanceof InstantiationAwareBeanPostProcessor) {
                InstantiationAwareBeanPostProcessor ibp = (InstantiationAwareBeanPostProcessor) bp;
                // 对所有需要依赖检查的属性进行后处理
                PropertyValues pvsToUse = ibp.postProcessProperties(pvs, bw.getWrappedInstance(), beanName);
                if (pvsToUse == null) {
                    // 从 bw 对象中提取 PropertyDescriptor 结果集
                    // PropertyDescriptor：可以通过一对存取方法提取一个属性
                    if (filteredPds == null) {
                        filteredPds = filterPropertyDescriptorsForDependencyCheck(bw, mbd.allowCaching);
                    }
                    pvsToUse = ibp.postProcessPropertyValues(pvs, filteredPds, bw.getWrappedInstance(), beanName);
                    if (pvsToUse == null) {
                        return;
                    }
                }
                pvs = pvsToUse;
            }
        }
    }
    
    // <4> 依赖检查
    if (needsDepCheck) {
        if (filteredPds == null) {
            filteredPds = filterPropertyDescriptorsForDependencyCheck(bw, mbd.allowCaching);
        }
        // 依赖检查，对应 depends-on 属性
        checkDependencies(beanName, mbd, filteredPds, pvs);
    }

    // <5> 将属性应用到 bean 中
    if (pvs != null) {
        applyPropertyValues(beanName, mbd, bw, pvs);
    }
}
```

处理流程如下：

- `<1>` ，根据 `hasInstantiationAwareBeanPostProcessors` 属性来判断，是否需要在注入属性之前给 InstantiationAwareBeanPostProcessors 最后一次改变 bean 的机会。**此过程可以控制 Spring 是否继续进行属性填充**。

- 统一存入到 PropertyValues 中，PropertyValues 用于描述 bean 的属性。

  - ```
    <2>
    ```

     

    ，根据注入类型(

     

    ```
    AbstractBeanDefinition#getResolvedAutowireMode()
    ```

     

    方法的返回值 )的不同来判断：

    - 是根据名称来自动注入（`#autowireByName(...)`）
    - 还是根据类型来自动注入（`#autowireByType(...)`）
    - 详细解析，见 [「1.1 自动注入」](http://svip.iocoder.cn/Spring/IoC-get-Bean-createBean-4/#) 。

  - `<3>` ，进行 BeanPostProcessor 处理。

  - `<4>` ，依赖检测。

- `<5>` ，将所有 PropertyValues 中的属性，填充到 BeanWrapper 中。

#### 1.1 自动注入

Spring 会根据注入类型（ `byName` / `byType` ）的不同，调用不同的方法来注入属性值。代码如下：

```
// AbstractBeanDefinition.java

/**
 * 注入模式
 */
private int autowireMode = AUTOWIRE_NO;

public int getResolvedAutowireMode() {
	if (this.autowireMode == AUTOWIRE_AUTODETECT) { // 自动检测模式，获得对应的检测模式
		// Work out whether to apply setter autowiring or constructor autowiring.
		// If it has a no-arg constructor it's deemed to be setter autowiring,
		// otherwise we'll try constructor autowiring.
		Constructor<?>[] constructors = getBeanClass().getConstructors();
		for (Constructor<?> constructor : constructors) {
			if (constructor.getParameterCount() == 0) {
				return AUTOWIRE_BY_TYPE;
			}
		}
		return AUTOWIRE_CONSTRUCTOR;
	} else {
		return this.autowireMode;
	}
}
```

###### 1.1.1 autowireByName

`#autowireByName(String beanName, AbstractBeanDefinition mbd, BeanWrapper bw, MutablePropertyValues pvs)` 方法，是根据**属性名称**，完成自动依赖注入的。代码如下：

```
// AbstractAutowireCapableBeanFactory.java

protected void autowireByName(String beanName, AbstractBeanDefinition mbd, BeanWrapper bw, MutablePropertyValues pvs) {
    // <1> 对 Bean 对象中非简单属性
    String[] propertyNames = unsatisfiedNonSimpleProperties(mbd, bw);
    // 遍历 propertyName 数组
    for (String propertyName : propertyNames) {
        // 如果容器中包含指定名称的 bean，则将该 bean 注入到 bean中
        if (containsBean(propertyName)) {
            // 递归初始化相关 bean
            Object bean = getBean(propertyName);
            // 为指定名称的属性赋予属性值
            pvs.add(propertyName, bean);
            // 属性依赖注入
            registerDependentBean(propertyName, beanName);
            if (logger.isTraceEnabled()) {
                logger.trace("Added autowiring by name from bean name '" + beanName +
                        "' via property '" + propertyName + "' to bean named '" + propertyName + "'");
            }
        } else {
            if (logger.isTraceEnabled()) {
                logger.trace("Not autowiring property '" + propertyName + "' of bean '" + beanName +
                        "' by name: no matching bean found");
            }
        }
    }
}
```

- `<1>` 处，该方法逻辑很简单，获取该 bean 的非简单属性。**什么叫做非简单属性呢**？就是类型为对象类型的属性，但是这里并不是将所有的对象类型都都会找到，比如 8 个原始类型，String 类型 ，Number类型、Date类型、URL类型、URI类型等都会被忽略。代码如下：

  ```
  // AbstractAutowireCapableBeanFactory.java
  
  protected String[] unsatisfiedNonSimpleProperties(AbstractBeanDefinition mbd, BeanWrapper bw) {
      // 创建 result 集合
      Set<String> result = new TreeSet<>();
      PropertyValues pvs = mbd.getPropertyValues();
      // 遍历 PropertyDescriptor 数组
      PropertyDescriptor[] pds = bw.getPropertyDescriptors();
      for (PropertyDescriptor pd : pds) {
          if (pd.getWriteMethod() != null // 有可写方法
                  && !isExcludedFromDependencyCheck(pd) // 依赖检测中没有被忽略
                  && !pvs.contains(pd.getName()) // pvs 不包含该属性名
                  && !BeanUtils.isSimpleProperty(pd.getPropertyType())) { // 不是简单属性类型
              result.add(pd.getName()); // 添加到 result 中
          }
      }
      return StringUtils.toStringArray(result);
  }
  ```

  - 过滤**条件**为：有可写方法、依赖检测中没有被忽略、不是简单属性类型。
  - 过滤**结果**为：其实这里获取的就是需要依赖注入的属性。

- 获取需要依赖注入的属性后，通过迭代、递归的方式初始化相关的 bean ，然后调用 `#registerDependentBean(String beanName, String dependentBeanName)` 方法，完成注册依赖。代码如下：

  ```
  // DefaultSingletonBeanRegistry.java
  
  /**
   * Map between dependent bean names: bean name to Set of dependent bean names.
   *
   * 保存的是依赖 beanName 之间的映射关系：beanName - > 依赖 beanName 的集合
   */
  private final Map<String, Set<String>> dependentBeanMap = new ConcurrentHashMap<>(64);
  
  /**
   * Map between depending bean names: bean name to Set of bean names for the bean's dependencies.
   *
   * 保存的是依赖 beanName 之间的映射关系：依赖 beanName - > beanName 的集合
   */
  private final Map<String, Set<String>> dependenciesForBeanMap = new ConcurrentHashMap<>(64);
  
  public void registerDependentBean(String beanName, String dependentBeanName) {
      // 获取 beanName
      String canonicalName = canonicalName(beanName);
      // 添加 <canonicalName, <dependentBeanName>> 到 dependentBeanMap 中
      synchronized (this.dependentBeanMap) {
          Set<String> dependentBeans =
                  this.dependentBeanMap.computeIfAbsent(canonicalName, k -> new LinkedHashSet<>(8));
          if (!dependentBeans.add(dependentBeanName)) {
              return;
          }
      }
      // 添加 <dependentBeanName, <canonicalName>> 到 dependenciesForBeanMap 中
      synchronized (this.dependenciesForBeanMap) {
          Set<String> dependenciesForBean =
                  this.dependenciesForBeanMap.computeIfAbsent(dependentBeanName, k -> new LinkedHashSet<>(8));
          dependenciesForBean.add(canonicalName);
      }
  }
  ```

###### 1.1.2 autowireByType

`#autowireByType(String beanName, AbstractBeanDefinition mbd, BeanWrapper bw, MutablePropertyValues pvs)` 方法，是根据**属性类型**，完成自动依赖注入的。代码如下：

```
// AbstractAutowireCapableBeanFactory.java

protected void autowireByType(String beanName, AbstractBeanDefinition mbd, BeanWrapper bw, MutablePropertyValues pvs) {

    // 获取 TypeConverter 实例
    // 使用自定义的 TypeConverter，用于取代默认的 PropertyEditor 机制
    TypeConverter converter = getCustomTypeConverter();
    if (converter == null) {
        converter = bw;
    }

    Set<String> autowiredBeanNames = new LinkedHashSet<>(4);
    // 获取非简单属性
    String[] propertyNames = unsatisfiedNonSimpleProperties(mbd, bw);
    // 遍历 propertyName 数组
    for (String propertyName : propertyNames) {
        try {
            // 获取 PropertyDescriptor 实例
            PropertyDescriptor pd = bw.getPropertyDescriptor(propertyName);
            // Don't try autowiring by type for type Object: never makes sense,
            // even if it technically is a unsatisfied, non-simple property.
            // 不要尝试按类型
            if (Object.class != pd.getPropertyType()) {
                // 探测指定属性的 set 方法
                MethodParameter methodParam = BeanUtils.getWriteMethodParameter(pd);
                // Do not allow eager init for type matching in case of a prioritized post-processor.
                boolean eager = !PriorityOrdered.class.isInstance(bw.getWrappedInstance());
                DependencyDescriptor desc = new AutowireByTypeDependencyDescriptor(methodParam, eager);
                // 解析指定 beanName 的属性所匹配的值，并把解析到的属性名称存储在 autowiredBeanNames 中
                // 当属性存在过个封装 bean 时将会找到所有匹配的 bean 并将其注入
                Object autowiredArgument = resolveDependency(desc, beanName, autowiredBeanNames, converter);
                if (autowiredArgument != null) {
                    pvs.add(propertyName, autowiredArgument);
                }
                // 遍历 autowiredBeanName 数组
                for (String autowiredBeanName : autowiredBeanNames) {
                    // 属性依赖注入
                    registerDependentBean(autowiredBeanName, beanName);
                    if (logger.isTraceEnabled()) {
                        logger.trace("Autowiring by type from bean name '" + beanName + "' via property '" +
                                propertyName + "' to bean named '" + autowiredBeanName + "'");
                    }
                }
                // 清空 autowiredBeanName 数组
                autowiredBeanNames.clear();
            }
        } catch (BeansException ex) {
            throw new UnsatisfiedDependencyException(mbd.getResourceDescription(), beanName, propertyName, ex);
        }
    }
}
```

- 其实主要过程和根据名称自动注入**差不多**，都是找到需要依赖注入的属性，然后通过迭代的方式寻找所匹配的 bean，最后调用 `#registerDependentBean(...)` 方法，来注册依赖。不过相对于 `#autowireByName(...)` 方法而言，根据类型寻找相匹配的 bean 过程**比较复杂**。

####### 1.1.2.1 resolveDependency

下面我们就分析这个复杂的过程，代码如下：

```
// DefaultListableBeanFactory.java

@Nullable
private static Class<?> javaxInjectProviderClass;

static {
	try {
		javaxInjectProviderClass = ClassUtils.forName("javax.inject.Provider", DefaultListableBeanFactory.class.getClassLoader());
	} catch (ClassNotFoundException ex) {
		// JSR-330 API not available - Provider interface simply not supported then.
		javaxInjectProviderClass = null;
	}
}

@Override
@Nullable
public Object resolveDependency(DependencyDescriptor descriptor, @Nullable String requestingBeanName,
        @Nullable Set<String> autowiredBeanNames, @Nullable TypeConverter typeConverter) throws BeansException {
    // 初始化参数名称发现器，该方法并不会在这个时候尝试检索参数名称
    // getParameterNameDiscoverer 返回 parameterNameDiscoverer 实例，parameterNameDiscoverer 方法参数名称的解析器
    descriptor.initParameterNameDiscovery(getParameterNameDiscoverer());
    // 依赖类型为 Optional 类型
    if (Optional.class == descriptor.getDependencyType()) {
        return createOptionalDependency(descriptor, requestingBeanName);
    // 依赖类型为ObjectFactory、ObjectProvider
    } else if (ObjectFactory.class == descriptor.getDependencyType() ||
            ObjectProvider.class == descriptor.getDependencyType()) {
        return new DependencyObjectProvider(descriptor, requestingBeanName);
    // javaxInjectProviderClass 类注入的特殊处理
    } else if (javaxInjectProviderClass == descriptor.getDependencyType()) {
        return new Jsr330Factory().createDependencyProvider(descriptor, requestingBeanName);
    } else {
        // 为实际依赖关系目标的延迟解析构建代理
        // 默认实现返回 null
        Object result = getAutowireCandidateResolver().getLazyResolutionProxyIfNecessary(descriptor, requestingBeanName);
        if (result == null) {
            // 通用处理逻辑
            result = doResolveDependency(descriptor, requestingBeanName, autowiredBeanNames, typeConverter);
        }
        return result;
    }
}
```

- 这里我们关注**通用处理逻辑** `#doResolveDependency(DependencyDescriptor descriptor, @Nullable String beanName, Set<String> autowiredBeanNames, TypeConverter typeConverter)` 方法，代码如下：

  ```
  // DefaultListableBeanFactory.java
  
  @Nullable
  public Object doResolveDependency(DependencyDescriptor descriptor, @Nullable String beanName,
      @Nullable Set<String> autowiredBeanNames, @Nullable TypeConverter typeConverter) throws BeansException {
      // 注入点
      InjectionPoint previousInjectionPoint = ConstructorResolver.setCurrentInjectionPoint(descriptor);
      try {
          // 针对给定的工厂给定一个快捷实现的方式，例如考虑一些预先解析的信息
          // 在进入所有bean的常规类型匹配算法之前，解析算法将首先尝试通过此方法解析快捷方式。
          // 子类可以覆盖此方法
          Object shortcut = descriptor.resolveShortcut(this);
          if (shortcut != null) {
              // 返回快捷的解析信息
              return shortcut;
          }
          // 依赖的类型
          Class<?> type = descriptor.getDependencyType();
          // 支持 Spring 的注解 @value
          Object value = getAutowireCandidateResolver().getSuggestedValue(descriptor);
          if (value != null) {
              if (value instanceof String) {
                  String strVal = resolveEmbeddedValue((String) value);
                  BeanDefinition bd = (beanName != null && containsBean(beanName) ? getMergedBeanDefinition(beanName) : null);
                  value = evaluateBeanDefinitionString(strVal, bd);
              }
              TypeConverter converter = (typeConverter != null ? typeConverter : getTypeConverter());
              return (descriptor.getField() != null ?
                      converter.convertIfNecessary(value, type, descriptor.getField()) :
                      converter.convertIfNecessary(value, type, descriptor.getMethodParameter()));
          }
          // 解析复合 bean，其实就是对 bean 的属性进行解析
          // 包括：数组、Collection 、Map 类型
          Object multipleBeans = resolveMultipleBeans(descriptor, beanName, autowiredBeanNames, typeConverter);
          if (multipleBeans != null) {
              return multipleBeans;
          }
          // 查找与类型相匹配的 bean
          // 返回值构成为：key = 匹配的 beanName，value = beanName 对应的实例化 bean
          Map<String, Object> matchingBeans = findAutowireCandidates(beanName, type, descriptor);
          // 没有找到，检验 @autowire  的 require 是否为 true
          if (matchingBeans.isEmpty()) {
              // 如果 @autowire 的 require 属性为 true ，但是没有找到相应的匹配项，则抛出异常
              if (isRequired(descriptor)) {
                  raiseNoMatchingBeanFound(type, descriptor.getResolvableType(), descriptor);
              }
              return null;
          }
          String autowiredBeanName;
          Object instanceCandidate;
          if (matchingBeans.size() > 1) {
              // 确认给定 bean autowire 的候选者
              // 按照 @Primary 和 @Priority 的顺序
              autowiredBeanName = determineAutowireCandidate(matchingBeans, descriptor);
              if (autowiredBeanName == null) {
                  if (isRequired(descriptor) || !indicatesMultipleBeans(type)) {
                      // 唯一性处理
                      return descriptor.resolveNotUnique(descriptor.getResolvableType(), matchingBeans);
                  }
                  else {
                      // In case of an optional Collection/Map, silently ignore a non-unique case:
                      // possibly it was meant to be an empty collection of multiple regular beans
                      // (before 4.3 in particular when we didn't even look for collection beans).
                      // 在可选的Collection / Map的情况下，默默地忽略一个非唯一的情况：可能它是一个多个常规bean的空集合
                      return null;
                  }
              }
              instanceCandidate = matchingBeans.get(autowiredBeanName);
          } else {
              // We have exactly one match.
              Map.Entry<String, Object> entry = matchingBeans.entrySet().iterator().next();
              autowiredBeanName = entry.getKey();
              instanceCandidate = entry.getValue();
          }
          if (autowiredBeanNames != null) {
              autowiredBeanNames.add(autowiredBeanName);
          }
          if (instanceCandidate instanceof Class) {
              instanceCandidate = descriptor.resolveCandidate(autowiredBeanName, type, this);
          }
          Object result = instanceCandidate;
          if (result instanceof NullBean) {
              if (isRequired(descriptor)) {
                  raiseNoMatchingBeanFound(type, descriptor.getResolvableType(), descriptor);
              }
              result = null;
          }
          if (!ClassUtils.isAssignableValue(type, result)) {
              throw new BeanNotOfRequiredTypeException(autowiredBeanName, type, instanceCandidate.getClass());
          }
          return result;
      } finally {
          ConstructorResolver.setCurrentInjectionPoint(previousInjectionPoint);
      }
  }
  ```

  - 代码比较多，胖友调试看看。看懂大体逻辑即可。

------

到这里就已经完成了所有属性的注入了。`populateBean()` 该方法就已经完成了一大半工作了：

- 下一步，则是对依赖 bean 的依赖检测和 PostProcessor 处理，**这个我们后面分析**。
- 下面，分析该方法的最后一步：`#applyPropertyValues(String beanName, BeanDefinition mbd, BeanWrapper bw, PropertyValues pvs)` 方法。

#### 1.2 applyPropertyValues

其实，上面只是完成了所有注入属性的获取，将获取的属性封装在 PropertyValues 的实例对象 `pvs` 中，并没有应用到已经实例化的 bean 中。而 `#applyPropertyValues(String beanName, BeanDefinition mbd, BeanWrapper bw, PropertyValues pvs)` 方法，则是完成这一步骤的。代码如下：

```
// AbstractAutowireCapableBeanFactory.java

protected void applyPropertyValues(String beanName, BeanDefinition mbd, BeanWrapper bw, PropertyValues pvs) {
    if (pvs.isEmpty()) {
        return;
    }

    // 设置 BeanWrapperImpl 的 SecurityContext 属性
    if (System.getSecurityManager() != null && bw instanceof BeanWrapperImpl) {
        ((BeanWrapperImpl) bw).setSecurityContext(getAccessControlContext());
    }

    // MutablePropertyValues 类型属性
    MutablePropertyValues mpvs = null;

    // 原始类型
    List<PropertyValue> original;
    // 获得 original
    if (pvs instanceof MutablePropertyValues) {
        mpvs = (MutablePropertyValues) pvs;
        // 属性值已经转换
        if (mpvs.isConverted()) {
            // Shortcut: use the pre-converted values as-is.
            try {
                // 为实例化对象设置属性值 ，依赖注入真真正正地实现在此！！！！！
                bw.setPropertyValues(mpvs);
                return;
            } catch (BeansException ex) {
                throw new BeanCreationException(
                        mbd.getResourceDescription(), beanName, "Error setting property values", ex);
            }
        }
        original = mpvs.getPropertyValueList();
    } else {
        // 如果 pvs 不是 MutablePropertyValues 类型，则直接使用原始类型
        original = Arrays.asList(pvs.getPropertyValues());
    }

    // 获取 TypeConverter = 获取用户自定义的类型转换
    TypeConverter converter = getCustomTypeConverter();
    if (converter == null) {
        converter = bw;
    }

    // 获取对应的解析器
    BeanDefinitionValueResolver valueResolver = new BeanDefinitionValueResolver(this, beanName, mbd, converter);

    // Create a deep copy, resolving any references for values.
    List<PropertyValue> deepCopy = new ArrayList<>(original.size());
    boolean resolveNecessary = false;
    // 遍历属性，将属性转换为对应类的对应属性的类型
    for (PropertyValue pv : original) {
        // 属性值不需要转换
        if (pv.isConverted()) {
            deepCopy.add(pv);
        // 属性值需要转换
        } else {
            String propertyName = pv.getName();
            Object originalValue = pv.getValue(); // 原始的属性值，即转换之前的属性值
            Object resolvedValue = valueResolver.resolveValueIfNecessary(pv, originalValue); // 转换属性值，例如将引用转换为IoC容器中实例化对象引用 ！！！！！ 对属性值的解析！！
            Object convertedValue = resolvedValue; // 转换之后的属性值
            boolean convertible = bw.isWritableProperty(propertyName) &&
                    !PropertyAccessorUtils.isNestedOrIndexedProperty(propertyName);  // 属性值是否可以转换
            // 使用用户自定义的类型转换器转换属性值
            if (convertible) {
                convertedValue = convertForProperty(resolvedValue, propertyName, bw, converter);
            }
            // Possibly store converted value in merged bean definition,
            // in order to avoid re-conversion for every created bean instance.
            // 存储转换后的属性值，避免每次属性注入时的转换工作
            if (resolvedValue == originalValue) {
                if (convertible) {
                    // 设置属性转换之后的值
                    pv.setConvertedValue(convertedValue);
                }
                deepCopy.add(pv);
            // 属性是可转换的，且属性原始值是字符串类型，且属性的原始类型值不是
            // 动态生成的字符串，且属性的原始值不是集合或者数组类型
            } else if (convertible && originalValue instanceof TypedStringValue &&
                    !((TypedStringValue) originalValue).isDynamic() &&
                    !(convertedValue instanceof Collection || ObjectUtils.isArray(convertedValue))) {
                pv.setConvertedValue(convertedValue);
                deepCopy.add(pv);
            } else {
                resolveNecessary = true;
                // 重新封装属性的值
                deepCopy.add(new PropertyValue(pv, convertedValue));
            }
        }
    }
    // 标记属性值已经转换过
    if (mpvs != null && !resolveNecessary) {
        mpvs.setConverted();
    }

    // Set our (possibly massaged) deep copy.
    // 进行属性依赖注入，依赖注入的真真正正实现依赖的注入方法在此！！！
    try {
        bw.setPropertyValues(new MutablePropertyValues(deepCopy));
    } catch (BeansException ex) {
        throw new BeanCreationException(
                mbd.getResourceDescription(), beanName, "Error setting property values", ex);
    }
}
```

总结 `#applyPropertyValues(...)` 方法（完成属性转换）：

- 属性值类型**不需要**转换时，不需要解析属性值，直接准备进行依赖注入。
- 属性值**需要**进行类型转换时，如对其他对象的引用等，首先需要解析属性值，然后对解析后的属性值进行依赖注入。

而且，我们看到调用了 `#resolveValueIfNecessary(...)`方法对属性值的解析。详细解析，可见 [《Spring应用、原理以及粗读源码系列（一）–框架总述、以Bean为核心的机制（IoC容器初始化以及依赖注入）》](https://blog.csdn.net/Jack__Frost/article/details/70229593) 的 [「7. 追踪 resolveValueIfNecessary ，发现是在 BeanDefinitionValueResolver 类」](http://svip.iocoder.cn/Spring/IoC-get-Bean-createBean-4/#) 。

## 2. 小结

至此，`#doCreateBean(...)` 方法的第二个过程：**属性填充**已经分析完成了，下篇分析第三个过程：**循环依赖的处理**。其实，循环依赖并不仅仅只是在 `#doCreateBean(...)` 方法中处理，在整个加载 bean 的过程中都有涉及。所以下篇内容并不仅仅只局限于 `#doCreateBean(...)` 方法。

# 8、创建 Bean（四）循环依赖处理

这篇分析 `#doCreateBean(...)` 方法的**第三个**过程：**循环依赖处理**。其实，循环依赖并不仅仅只是在 `#doCreateBean(...)` 方法中处理，而是在整个加载 bean 的过程中都有涉及。所以，本文内容并不仅仅只局限于 `#doCreateBean(...)` 方法，而是从整个 Bean 的加载过程进行分析。

## 1. 什么是循环依赖

循环依赖，其实就是循环引用，就是两个或者两个以上的 bean 互相引用对方，最终形成一个闭环，如 A 依赖 B，B 依赖 C，C 依赖 A。如下图所示：

[![循环依赖](http://static.iocoder.cn/20170912082357749.jpeg)](http://static.iocoder.cn/20170912082357749.jpeg)循环依赖

循环依赖，其实就是一个**死循环**的过程，在初始化 A 的时候发现引用了 B，这时就会去初始化 B，然后又发现 B 引用 C，跑去初始化 C，初始化 C 的时候发现引用了 A，则又会去初始化 A，依次循环永不退出，除非有**终结条件**。

Spring 循环依赖的**场景**有两种：

1. 构造器的循环依赖。
2. field 属性的循环依赖。

对于构造器的循环依赖，Spring 是无法解决的，只能抛出 BeanCurrentlyInCreationException 异常表示循环依赖，**所以下面我们分析的都是基于 field 属性的循环依赖**。

在博客 [《【【死磕 Spring】—— IoC 之开启 Bean 的加载》](http://svip.iocoder.cn/Spring/IoC-get-Bean-begin) 中提到，Spring 只解决 scope 为 singleton 的循环依赖。对于scope 为 prototype 的 bean ，Spring 无法解决，直接抛出 BeanCurrentlyInCreationException 异常。

为什么 Spring 不处理 prototype bean 呢？其实如果理解 Spring 是如何解决 singleton bean 的循环依赖就明白了。这里先卖一个关子，我们先来关注 Spring 是如何解决 singleton bean 的循环依赖的。

## 2. 解决循环依赖

#### 2.1 getSingleton

我们先从加载 bean 最初始的方法 AbstractBeanFactory 的 `#doGetBean(final String name, final Class<T> requiredType, final Object[] args, boolean typeCheckOnly)` 方法开始。

在 `#doGetBean(...)` 方法中，首先会根据 `beanName` 从单例 bean 缓存中获取，**如果不为空则直接返回**。代码如下：

```
// AbstractBeanFactory.java

Object sharedInstance = getSingleton(beanName);
```

- 调用 `#getSingleton(String beanName, boolean allowEarlyReference)` 方法，从单例缓存中获取。代码如下：

  ```
  // DefaultSingletonBeanRegistry.java
  
  @Nullable
  protected Object getSingleton(String beanName, boolean allowEarlyReference) {
      // 从单例缓冲中加载 bean
      Object singletonObject = this.singletonObjects.get(beanName);
      // 缓存中的 bean 为空，且当前 bean 正在创建
      if (singletonObject == null && isSingletonCurrentlyInCreation(beanName)) {
          // 加锁
          synchronized (this.singletonObjects) {
              // 从 earlySingletonObjects 获取
              singletonObject = this.earlySingletonObjects.get(beanName);
              // earlySingletonObjects 中没有，且允许提前创建
              if (singletonObject == null && allowEarlyReference) {
                  // 从 singletonFactories 中获取对应的 ObjectFactory
                  ObjectFactory<?> singletonFactory = this.singletonFactories.get(beanName);
                  if (singletonFactory != null) {
                      // 获得 bean
                      singletonObject = singletonFactory.getObject();
                      // 添加 bean 到 earlySingletonObjects 中
                      this.earlySingletonObjects.put(beanName, singletonObject);
                      // 从 singletonFactories 中移除对应的 ObjectFactory
                      this.singletonFactories.remove(beanName);
                  }
              }
          }
      }
      return singletonObject;
  }
  ```

  - 这个方法主要是从三个缓存中获取，分别是：`singletonObjects`、`earlySingletonObjects`、`singletonFactories` 。三者定义如下：

    ```
    // DefaultSingletonBeanRegistry.java
            
    /**
     * Cache of singleton objects: bean name to bean instance.
     *
     * 一级缓存，存放的是单例 bean 的映射。
     *
     * 注意，这里的 bean 是已经创建完成的。
     *
     * 对应关系为 bean name --> bean instance
     */
    private final Map<String, Object> singletonObjects = new ConcurrentHashMap<>(256);
        
    /**
     * Cache of early singleton objects: bean name to bean instance.
     *
     * 二级缓存，存放的是早期半成品（未初始化完）的 bean，对应关系也是 bean name --> bean instance。
     *
     * 它与 {@link #singletonObjects} 区别在于， 它自己存放的 bean 不一定是完整。
     *
     * 这个 Map 也是【循环依赖】的关键所在。
     */
    private final Map<String, Object> earlySingletonObjects = new HashMap<>(16);
        
    /**
     * Cache of singleton factories: bean name to ObjectFactory.
     *
     * 三级缓存，存放的是 ObjectFactory，可以理解为创建早期半成品（未初始化完）的 bean 的 factory ，最终添加到二级缓存 {@link #earlySingletonObjects} 中
     *
     * 对应关系是 bean name --> ObjectFactory
     *
     * 这个 Map 也是【循环依赖】的关键所在。
     */
    private final Map<String, ObjectFactory<?>> singletonFactories = new HashMap<>(16);
    ```

    - `singletonObjects` ：单例对象的 Cache 。
    - `earlySingletonObjects` ：**提前曝光**的单例对象的 Cache 。
    - `singletonFactories` ： 单例对象工厂的 Cache 。

它们三，就是 Spring 解决 singleton bean 的关键因素所在，我称他们为**三级缓存**：

- 第一级为 `singletonObjects`
- 第二级为 `earlySingletonObjects`
- 第三级为 `singletonFactories`

这里，我们已经通过 `#getSingleton(String beanName, boolean allowEarlyReference)` 方法，看到他们是如何配合的。详细分析该方法之前，提下其中的 `#isSingletonCurrentlyInCreation(String beanName)` 方法和 `allowEarlyReference` 变量：

- `#isSingletonCurrentlyInCreation(String beanName)` 方法：判断当前 singleton bean 是否处于创建中。bean 处于创建中，也就是说 bean 在初始化但是没有完成初始化，有一个这样的过程其实和 Spring 解决 bean 循环依赖的理念相辅相成。**因为 Spring 解决 singleton bean 的核心就在于提前曝光 bean** 。
- `allowEarlyReference` 变量：从字面意思上面理解就是允许提前拿到引用。其实真正的意思是，是否允许从 `singletonFactories` 缓存中通过 `#getObject()` 方法，拿到对象。为什么会有这样一个字段呢？**原因就在于 `singletonFactories` 才是 Spring 解决 singleton bean 的诀窍所在**，这个我们后续分析。

------

`#getSingleton(String beanName, boolean allowEarlyReference)` 方法，整个过程如下：

- 首先，从一级缓存 `singletonObjects` 获取。

- 如果，没有且当前指定的 `beanName` 正在创建，就再从二级缓存 `earlySingletonObjects` 中获取。

- 如果，还是没有获取到且允许 `singletonFactories` 通过 `#getObject()` 获取，则从三级缓存 `singletonFactories` 获取。如果获取到，则通过其 `#getObject()` 方法，获取对象，并将其加入到二级缓存 `earlySingletonObjects` 中，并从三级缓存 `singletonFactories` 删除。代码如下：

  ```
  // DefaultSingletonBeanRegistry.java
  
  singletonObject = singletonFactory.getObject();
  this.earlySingletonObjects.put(beanName, singletonObject);
  this.singletonFactories.remove(beanName);
  ```

  - 这样，就从三级缓存**升级**到二级缓存了。
  - 😈 所以，二级缓存存在的**意义**，就是缓存三级缓存中的 ObjectFactory 的 `#getObject()` 方法的执行结果，提早曝光的**单例** Bean 对象。

#### 2.2 addSingletonFactory

上面是从缓存中获取，但是缓存中的数据从哪里添加进来的呢？一直往下跟会发现在 AbstractAutowireCapableBeanFactory 的 `#doCreateBean(final String beanName, final RootBeanDefinition mbd, final Object[] args)` 方法中，有这么一段代码：

```
// AbstractAutowireCapableBeanFactory.java

boolean earlySingletonExposure = (mbd.isSingleton() // 单例模式
        && this.allowCircularReferences // 运行循环依赖
        && isSingletonCurrentlyInCreation(beanName)); // 当前单例 bean 是否正在被创建
if (earlySingletonExposure) {
    if (logger.isTraceEnabled()) {
        logger.trace("Eagerly caching bean '" + beanName +
                "' to allow for resolving potential circular references");
    }
    // 提前将创建的 bean 实例加入到 singletonFactories 中
    // <X> 这里是为了后期避免循环依赖
    addSingletonFactory(beanName, () -> getEarlyBeanReference(beanName, mbd, bean));
}
```

- 当一个 Bean 满足三个条件时，则调用

   

  ```
  #addSingletonFactory(...)
  ```

   

  方法，将它添加到缓存中。三个条件如下：

  - 单例
  - 运行提前暴露 bean
  - 当前 bean 正在创建中

- `#addSingletonFactory(String beanName, ObjectFactory<?> singletonFactory)` 方法，代码如下：

  ```
  // DefaultSingletonBeanRegistry.java
  
  protected void addSingletonFactory(String beanName, ObjectFactory<?> singletonFactory) {
  	Assert.notNull(singletonFactory, "Singleton factory must not be null");
  	synchronized (this.singletonObjects) {
  		if (!this.singletonObjects.containsKey(beanName)) {
  			this.singletonFactories.put(beanName, singletonFactory);
  			this.earlySingletonObjects.remove(beanName);
  			this.registeredSingletons.add(beanName);
  		}
  	}
  }
  ```

  - 从这段代码我们可以看出，`singletonFactories` 这个三级缓存才是解决 Spring Bean 循环依赖的诀窍所在。同时这段代码发生在 `#createBeanInstance(...)` 方法之后，也就是说这个 bean 其实已经被创建出来了，**但是它还不是很完美（没有进行属性填充和初始化），但是对于其他依赖它的对象而言已经足够了（可以根据对象引用定位到堆中对象），能够被认出来了**。所以 Spring 在这个时候，选择将该对象提前曝光出来让大家认识认识。

另外，`<X>` 处的 `#getEarlyBeanReference(String beanName, RootBeanDefinition mbd, Object bean)` 方法也**非常重要**，这里会创建早期初始化 Bean 可能存在的 AOP 代理等等。代码如下：

```
// AbstractAutowireCapableBeanFactory.java

/**
 * 对创建的早期半成品（未初始化）的 Bean 处理引用
 *
 * 例如说，AOP 就是在这里动态织入，创建其代理 Bean 返回
 *
 * Obtain a reference for early access to the specified bean,
 * typically for the purpose of resolving a circular reference.
 * @param beanName the name of the bean (for error handling purposes)
 * @param mbd the merged bean definition for the bean
 * @param bean the raw bean instance
 * @return the object to expose as bean reference
 */
protected Object getEarlyBeanReference(String beanName, RootBeanDefinition mbd, Object bean) {
	Object exposedObject = bean;
	if (!mbd.isSynthetic() && hasInstantiationAwareBeanPostProcessors()) {
		for (BeanPostProcessor bp : getBeanPostProcessors()) {
			if (bp instanceof SmartInstantiationAwareBeanPostProcessor) {
				SmartInstantiationAwareBeanPostProcessor ibp = (SmartInstantiationAwareBeanPostProcessor) bp;
				exposedObject = ibp.getEarlyBeanReference(exposedObject, beanName);
			}
		}
	}
	return exposedObject;
}
```

- 这也是为什么 Spring 需要额外增加 `singletonFactories` 三级缓存的原因，解决 Spring 循环依赖情况下的 Bean 存在动态代理等情况，不然循环注入到别人的 Bean 就是原始的，而不是经过动态代理的！
- 另外，这里在推荐一篇[《Spring循环依赖三级缓存是否可以减少为二级缓存？》](https://segmentfault.com/a/1190000023647227)文章，解释的也非常不错。

#### 2.3 addSingleton

介绍到这里我们发现三级缓存 `singletonFactories` 和 二级缓存 `earlySingletonObjects` 中的值都有出处了，那一级缓存在哪里设置的呢？在类 DefaultSingletonBeanRegistry 中，可以发现这个 `#addSingleton(String beanName, Object singletonObject)` 方法，代码如下：

```
// DefaultSingletonBeanRegistry.java

protected void addSingleton(String beanName, Object singletonObject) {
	synchronized (this.singletonObjects) {
		this.singletonObjects.put(beanName, singletonObject);
		this.singletonFactories.remove(beanName);
		this.earlySingletonObjects.remove(beanName);
		this.registeredSingletons.add(beanName);
	}
}
```

- 添加至一级缓存，同时从二级、三级缓存中删除。

- 这个方法在我们创建 bean 的链路中有哪个地方引用呢？其实在前面博客 LZ 已经提到过了，在 `#doGetBean(...)` 方法中，处理不同 scope 时，如果是 singleton，则调用 `#getSingleton(...)` 方法，如下图所示：

  [![getSingleton](http://static.iocoder.cn/20170912091609918.jpeg)](http://static.iocoder.cn/20170912091609918.jpeg)getSingleton

- 前面几篇博客已经分析了 `#createBean(...)` 方法，这里就不再阐述了，我们关注 `#getSingleton(String beanName, ObjectFactory<?> singletonFactory)` 方法，代码如下：

  ```
  // AbstractBeanFactory.java
  
  public Object getSingleton(String beanName, ObjectFactory<?> singletonFactory) {
      Assert.notNull(beanName, "Bean name must not be null");
      synchronized (this.singletonObjects) {
          Object singletonObject = this.singletonObjects.get(beanName);
          if (singletonObject == null) {
              //....
              try {
                  singletonObject = singletonFactory.getObject();
                  newSingleton = true;
              }
              //.....
              if (newSingleton) {
                  addSingleton(beanName, singletonObject);
              }
          }
          return singletonObject;
      }
  }
  ```

  - 😈 注意，此处的 `#getSingleton(String beanName, ObjectFactory<?> singletonFactory)` 方法，在 AbstractBeanFactory 类中实现，和 [「2.1 getSingleton」](http://svip.iocoder.cn/Spring/IoC-get-Bean-createBean-5/#) **不同**。

## 3. 小结

至此，Spring 关于 singleton bean 循环依赖已经分析完毕了。所以我们基本上可以确定 Spring 解决循环依赖的方案了：

- Spring 在创建 bean 的时候并不是等它完全完成，而是在创建过程中将创建中的 bean 的 ObjectFactory 提前曝光（即加入到 `singletonFactories` 缓存中）。
- 这样，一旦下一个 bean 创建的时候需要依赖 bean ，则直接使用 ObjectFactory 的 `#getObject()` 方法来获取了，也就是 [「2.1 getSingleton」](http://svip.iocoder.cn/Spring/IoC-get-Bean-createBean-5/#) 小结中的方法中的代码片段了。

到这里，关于 Spring 解决 bean 循环依赖就已经分析完毕了。最后来描述下就上面那个循环依赖 Spring 解决的过程：

- 首先 A 完成初始化第一步并将自己提前曝光出来（通过 ObjectFactory 将自己提前曝光），在初始化的时候，发现自己依赖对象 B，此时就会去尝试 get(B)，这个时候发现 B 还没有被创建出来
- 然后 B 就走创建流程，在 B 初始化的时候，同样发现自己依赖 C，C 也没有被创建出来
- 这个时候 C 又开始初始化进程，但是在初始化的过程中发现自己依赖 A，于是尝试 get(A)，这个时候由于 A 已经添加至缓存中（一般都是添加至三级缓存 `singletonFactories` ），通过 ObjectFactory 提前曝光，所以可以通过 `ObjectFactory#getObject()` 方法来拿到 A 对象，C 拿到 A 对象后顺利完成初始化，然后将自己添加到一级缓存中
- 回到 B ，B 也可以拿到 C 对象，完成初始化，A 可以顺利拿到 B 完成初始化。到这里整个链路就已经完成了初始化过程了

> 老艿艿的建议
>
> 可能逻辑干看比较绕，胖友可以拿出一个草稿纸，画一画上面提到的 A、B、C 初始化的过程。
>
> 相信，胖友会很快明白了。
>
> 如下是《Spring 源码深度解析》P114 页的一张图，非常有助于理解。
>
> [![处理依赖循环](http://static.iocoder.cn/images/Spring/2019-06-13/01.png)](http://static.iocoder.cn/images/Spring/2019-06-13/01.png)处理依赖循环

# 9、创建 Bean（五）初始化 Bean 对象

一个 bean 经历了 `#createBeanInstance(String beanName, RootBeanDefinition mbd, Object[] args)` 方法，被创建出来，然后又经过一番属性注入，依赖处理，历经千辛万苦，千锤百炼，终于有点儿 bean 实例的样子，能堪大任了，只需要经历最后一步就破茧成蝶了。

这**最后一步**就是初始化，也就是 `#initializeBean(final String beanName, final Object bean, RootBeanDefinition mbd)` 方法。所以，这篇文章我们分析 `#doCreateBean(...)` 方法的中最后一步：初始化 bean 对象。

## 1. initializeBean

```
// AbstractAutowireCapableBeanFactory.java

protected Object initializeBean(final String beanName, final Object bean, @Nullable RootBeanDefinition mbd) {
    if (System.getSecurityManager() != null) { // 安全模式
        AccessController.doPrivileged((PrivilegedAction<Object>) () -> {
            // <1> 激活 Aware 方法，对特殊的 bean 处理：Aware、BeanClassLoaderAware、BeanFactoryAware
            invokeAwareMethods(beanName, bean);
            return null;
        }, getAccessControlContext());
    } else {
        // <1> 激活 Aware 方法，对特殊的 bean 处理：Aware、BeanClassLoaderAware、BeanFactoryAware
        invokeAwareMethods(beanName, bean);
    }

    // <2> 后处理器，before
    Object wrappedBean = bean;
    if (mbd == null || !mbd.isSynthetic()) {
        wrappedBean = applyBeanPostProcessorsBeforeInitialization(wrappedBean, beanName);
    }

    // <3> 激活用户自定义的 init 方法
    try {
        invokeInitMethods(beanName, wrappedBean, mbd);
    } catch (Throwable ex) {
        throw new BeanCreationException(
                (mbd != null ? mbd.getResourceDescription() : null),
                beanName, "Invocation of init method failed", ex);
    }

    // <2> 后处理器，after
    if (mbd == null || !mbd.isSynthetic()) {
        wrappedBean = applyBeanPostProcessorsAfterInitialization(wrappedBean, beanName);
    }

    return wrappedBean;
}
```

初始化 bean 的方法其实就是三个步骤的处理，而这三个步骤主要还是根据**用户设定**的来进行初始化，这三个过程为：

- `<1>` 激活 Aware 方法。
- `<3>` 后置处理器的应用。
- `<2>` 激活自定义的 init 方法。

#### 1.1 激活 Aware 方法

Aware ，英文翻译是意识到的，感知的。Spring 提供了诸多 Aware 接口，用于辅助 Spring Bean 以编程的方式调用 Spring 容器，通过实现这些接口，可以增强 Spring Bean 的功能。

Spring 提供了如下系列的 Aware 接口：

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

------

`#invokeAwareMethods(final String beanName, final Object bean)` 方法，代码如下：

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

这里代码就没有什么好说的，主要是处理 BeanNameAware、BeanClassLoaderAware、BeanFactoryAware 。

关于 Aware 接口，后面会专门出篇文章对其进行详细分析说明的。

#### 1.2 后置处理器的应用

BeanPostProcessor 在前面介绍 bean 加载的过程曾多次遇到，相信各位不陌生，这是 Spring 中开放式框架中必不可少的一个亮点。

BeanPostProcessor 的作用是：如果我们想要在 Spring 容器完成 Bean 的实例化，配置和其他的初始化后添加一些自己的逻辑处理，那么请使用该接口，这个接口给与了用户充足的权限去更改或者扩展 Spring，是我们对 Spring 进行扩展和增强处理一个必不可少的接口。

- `#applyBeanPostProcessorsBeforeInitialization(...)` 方法，代码如下：

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
  ```

- `#applyBeanPostProcessorsAfterInitialization(...)` 方法，代码如下：

  ```
  // AbstractAutowireCapableBeanFactory.java
  
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

其实，逻辑就是通过 `#getBeanPostProcessors()` 方法，获取定义的 BeanPostProcessor ，然后分别调用其 `#postProcessBeforeInitialization(...)`、`#postProcessAfterInitialization(...)` 方法，进行**自定义**的业务处理。

#### 1.3 激活自定义的 init 方法

如果熟悉 `<bean>` 标签的配置，一定不会忘记 `init-method` 方法，该方法的执行就是在这里执行的。代码如下：

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
                    // <1> 属性初始化的处理
                    ((InitializingBean) bean).afterPropertiesSet();
                    return null;
                }, getAccessControlContext());
            } catch (PrivilegedActionException pae) {
                throw pae.getException();
            }
        } else {
            // <1> 属性初始化的处理
            ((InitializingBean) bean).afterPropertiesSet();
        }
    }

    if (mbd != null && bean.getClass() != NullBean.class) {
        String initMethodName = mbd.getInitMethodName();
        if (StringUtils.hasLength(initMethodName) &&
                !(isInitializingBean && "afterPropertiesSet".equals(initMethodName)) &&
                !mbd.isExternallyManagedInitMethod(initMethodName)) {
            // <2> 激活用户自定义的初始化方法
            invokeCustomInitMethod(beanName, bean, mbd);
        }
    }
}
```

- 首先，检查是否为 InitializingBean 。如果**是**的话，需要执行 `#afterPropertiesSet()` 方法，因为我们除了可以使用 `init-method` 来自定初始化方法外，还可以实现 InitializingBean 接口。接口仅有一个 `#afterPropertiesSet()` 方法。
- 两者的执行先后顺序是先 `<1>` 的 `#afterPropertiesSet()` 方法，后 `<2>` 的 `init-method` 对应的方法。

## 2. 小结

关于这篇博客的三个方法，LZ 后面会单独写博客来进行分析说明。

经过六篇博客终于把 Spring 创建 bean 的过程进行详细说明了，过程是艰辛的，但是收获很大，关键还是要耐着性子看。

# 10、实例化策略-InstantiationStrategy

我们先来简单回顾下 Bean 的实例化过程：

1. Bean 的创建，主要是 `AbstractAutowireCapableBeanFactory#doCreateBean(...)` 方法。在这个方法中有 Bean 的实例化、属性注入和初始化过程，对于 Bean 的实例化过程这是根据 Bean 的类型来判断的，如果是单例模式，则直接从 `factoryBeanInstanceCache` 缓存中获取，否则调用 `#createBeanInstance(...)` 方法来创建。
2. 在 `#createBeanInstance(...)` 方法中，如果 Supplier 不为空，则调用 `#obtainFromSupplier(...)` 实例化 bean。如果 `factory` 不为空，则调用 `#instantiateUsingFactoryMethod(...)` 方法来实例化 Bean 。如果都不是，则调用 `#instantiateBean(...)` 方法来实例化 Bean 。但是无论是 `#instantiateUsingFactoryMethod(...)` 方法，还是 `#instantiateBean()` 方法，最后都一定会调用到 InstantiationStrategy 接口的 `#instantiate(...)` 方法。

## 1. InstantiationStrategy

InstantiationStrategy 接口定义了 Spring Bean 实例化的策略，根据创建对象情况的不同，提供了三种策略：无参构造方法、有参构造方法、工厂方法。代码如下：

```
public interface InstantiationStrategy {

    /**
    * 默认构造方法
    */
    Object instantiate(RootBeanDefinition bd, @Nullable String beanName, BeanFactory owner)
    throws BeansException;

    /**
    * 指定构造方法
    */
    Object instantiate(RootBeanDefinition bd, @Nullable String beanName, BeanFactory owner,
    Constructor<?> ctor, @Nullable Object... args) throws BeansException;

    /**
    * 工厂方法
    */
    Object instantiate(RootBeanDefinition bd, @Nullable String beanName, BeanFactory owner,
    @Nullable Object factoryBean, Method factoryMethod, @Nullable Object... args)
    throws BeansException;

}
```

## 2. SimpleInstantiationStrategy

InstantiationStrategy 接口有两个实现类：SimpleInstantiationStrategy 和 CglibSubclassingInstantiationStrategy。

SimpleInstantiationStrategy 对以上三个方法都做了简单的实现。

① 如果是工厂方法实例化，则直接使用反射创建对象，如下：

```
// SimpleInstantiationStrategy.java

@Override
public Object instantiate(RootBeanDefinition bd, @Nullable String beanName, BeanFactory owner,
        @Nullable Object factoryBean, final Method factoryMethod, Object... args) {
    try {
        // 设置 Method 可访问
        if (System.getSecurityManager() != null) {
            AccessController.doPrivileged((PrivilegedAction<Object>) () -> {
                ReflectionUtils.makeAccessible(factoryMethod);
                return null;
            });
        } else {
            ReflectionUtils.makeAccessible(factoryMethod);
        }

        // 获得原 Method 对象
        Method priorInvokedFactoryMethod = currentlyInvokedFactoryMethod.get();
        try {
            // 设置新的 Method 对象，到 currentlyInvokedFactoryMethod 中
            currentlyInvokedFactoryMethod.set(factoryMethod);
            // 创建 Bean 对象
            Object result = factoryMethod.invoke(factoryBean, args);
            // 未创建，则创建 NullBean 对象
            if (result == null) {
                result = new NullBean();
            }
            return result;
        } finally {
            // 设置老的 Method 对象，到 currentlyInvokedFactoryMethod 中
            if (priorInvokedFactoryMethod != null) {
                currentlyInvokedFactoryMethod.set(priorInvokedFactoryMethod);
            } else {
                currentlyInvokedFactoryMethod.remove();
            }
        }
    // 一大堆 catch 异常
    } catch (IllegalArgumentException ex) {
        throw new BeanInstantiationException(factoryMethod,
                "Illegal arguments to factory method '" + factoryMethod.getName() + "'; " +
                "args: " + StringUtils.arrayToCommaDelimitedString(args), ex);
    } catch (IllegalAccessException ex) {
        throw new BeanInstantiationException(factoryMethod,
                "Cannot access factory method '" + factoryMethod.getName() + "'; is it public?", ex);
    } catch (InvocationTargetException ex) {
        String msg = "Factory method '" + factoryMethod.getName() + "' threw exception";
        if (bd.getFactoryBeanName() != null && owner instanceof ConfigurableBeanFactory &&
                ((ConfigurableBeanFactory) owner).isCurrentlyInCreation(bd.getFactoryBeanName())) {
            msg = "Circular reference involving containing bean '" + bd.getFactoryBeanName() + "' - consider " +
                    "declaring the factory method as static for independence from its containing instance. " + msg;
        }
        throw new BeanInstantiationException(factoryMethod, msg, ex.getTargetException());
    }
}
```

② 如果是构造方法实例化，则是先判断是否有 MethodOverrides，如果没有则是直接使用反射，如果有则就需要 CGLIB 实例化对象。如下：

```
// SimpleInstantiationStrategy.java

// 默认构造方法
@Override
public Object instantiate(RootBeanDefinition bd, @Nullable String beanName, BeanFactory owner) {
	// Don't override the class with CGLIB if no overrides.
    // 没有覆盖，直接使用反射实例化即可
    if (!bd.hasMethodOverrides()) {
		Constructor<?> constructorToUse;
		synchronized (bd.constructorArgumentLock) {
		    // 获得构造方法 constructorToUse
			constructorToUse = (Constructor<?>) bd.resolvedConstructorOrFactoryMethod;
			if (constructorToUse == null) {
				final Class<?> clazz = bd.getBeanClass();
				// 如果是接口，抛出 BeanInstantiationException 异常
				if (clazz.isInterface()) {
					throw new BeanInstantiationException(clazz, "Specified class is an interface");
				}
				try {
				    // 从 clazz 中，获得构造方法
					if (System.getSecurityManager() != null) { // 安全模式
						constructorToUse = AccessController.doPrivileged(
								(PrivilegedExceptionAction<Constructor<?>>) clazz::getDeclaredConstructor);
					} else {
						constructorToUse =	clazz.getDeclaredConstructor();
					}
					// 标记 resolvedConstructorOrFactoryMethod 属性
					bd.resolvedConstructorOrFactoryMethod = constructorToUse;
				} catch (Throwable ex) {
					throw new BeanInstantiationException(clazz, "No default constructor found", ex);
				}
			}
		}
        // 通过 BeanUtils 直接使用构造器对象实例化 Bean 对象
        return BeanUtils.instantiateClass(constructorToUse);
	} else {
		// Must generate CGLIB subclass.
        // 生成 CGLIB 创建的子类对象
        return instantiateWithMethodInjection(bd, beanName, owner);
	}
}

// 指定构造方法
@Override
public Object instantiate(RootBeanDefinition bd, @Nullable String beanName, BeanFactory owner,
		final Constructor<?> ctor, Object... args) {
    // 没有覆盖，直接使用反射实例化即可
	if (!bd.hasMethodOverrides()) {
		if (System.getSecurityManager() != null) {
		    // 设置构造方法，可访问
			// use own privileged to change accessibility (when security is on)
			AccessController.doPrivileged((PrivilegedAction<Object>) () -> {
				ReflectionUtils.makeAccessible(ctor);
				return null;
			});
		}
        // 通过 BeanUtils 直接使用构造器对象实例化 Bean 对象
		return BeanUtils.instantiateClass(ctor, args);
	} else {
        // 生成 CGLIB 创建的子类对象
		return instantiateWithMethodInjection(bd, beanName, owner, ctor, args);
	}
}
```

- SimpleInstantiationStrategy 对 `#instantiateWithMethodInjection(RootBeanDefinition bd, String beanName, BeanFactory owner, Constructor<?> ctor, Object... args)` 的实现任务交给了子类 CglibSubclassingInstantiationStrategy 。

## 3. MethodOverrides

对于 MethodOverrides，如果读者是跟着小编文章一路跟过来的话一定不会陌生，在 BeanDefinitionParserDelegate 类解析 `<bean/>` 的时候是否还记得这两个方法：`#parseLookupOverrideSubElements(...)` 和 `#parseReplacedMethodSubElements(...)` 这两个方法分别用于解析 `lookup-method` 和 `replaced-method` 属性。

其中，`#parseLookupOverrideSubElements(...)` 源码如下：

[![parseLookupOverrideSubElements](http://static.iocoder.cn/4cdb7d0fafb164c00feb74680948e785)](http://static.iocoder.cn/4cdb7d0fafb164c00feb74680948e785)parseLookupOverrideSubElements

更多关于 `lookup-method` 和 `replaced-method` 请看：[【死磕 Spring】—– IoC 之解析 bean 标签：meta、lookup-method、replace-method](http://svip.iocoder.cn/Spring/IoC-parse-BeanDefinitions-for-meta-and-look-method-and-replace-method)

## 4. CglibSubclassingInstantiationStrategy

类 CglibSubclassingInstantiationStrategy 为 Spring 实例化 Bean 的默认实例化策略，其主要功能还是对父类功能进行补充：其父类将 CGLIB 的实例化策略委托其实现。

```
// SimpleInstantiationStrategy.java

protected Object instantiateWithMethodInjection(RootBeanDefinition bd, @Nullable String beanName, BeanFactory owner) {
	throw new UnsupportedOperationException("Method Injection not supported in SimpleInstantiationStrategy");
}

// CglibSubclassingInstantiationStrategy.java

@Override
protected Object instantiateWithMethodInjection(RootBeanDefinition bd, @Nullable String beanName, BeanFactory owner) {
	return instantiateWithMethodInjection(bd, beanName, owner, null);
}
```

- CglibSubclassingInstantiationStrategy 实例化 Bean 策略，是通过其内部类 **CglibSubclassCreator** 来实现的。代码如下：

  ```
  // CglibSubclassingInstantiationStrategy.java
  
  @Override
  protected Object instantiateWithMethodInjection(RootBeanDefinition bd, @Nullable String beanName, BeanFactory owner, @Nullable Constructor<?> ctor, Object... args) {
  	// Must generate CGLIB subclass...
      // 通过CGLIB生成一个子类对象
  	return new CglibSubclassCreator(bd, owner).instantiate(ctor, args);
  }
  ```

- 创建 CglibSubclassCreator 实例，然后调用其 `#instantiate(Constructor<?> ctor, Object... args)` 方法，该方法用于动态创建子类实例，同时实现所需要的 lookups（`lookup-method`、`replace-method`）。

  ```
  // CglibSubclassingInstantiationStrategy.java#CglibSubclassCreator
  
  public Object instantiate(@Nullable Constructor<?> ctor, Object... args) {
      // <x> 通过 Cglib 创建一个代理类
      Class<?> subclass = createEnhancedSubclass(this.beanDefinition);
      Object instance;
      // <y> 没有构造器，通过 BeanUtils 使用默认构造器创建一个bean实例
      if (ctor == null) {
          instance = BeanUtils.instantiateClass(subclass);
      } else {
          try {
              // 获取代理类对应的构造器对象，并实例化 bean
              Constructor<?> enhancedSubclassConstructor = subclass.getConstructor(ctor.getParameterTypes());
              instance = enhancedSubclassConstructor.newInstance(args);
          } catch (Exception ex) {
              throw new BeanInstantiationException(this.beanDefinition.getBeanClass(),
                      "Failed to invoke constructor for CGLIB enhanced subclass [" + subclass.getName() + "]", ex);
          }
      }
      // SPR-10785: set callbacks directly on the instance instead of in the
      // enhanced class (via the Enhancer) in order to avoid memory leaks.
      // 为了避免 memory leaks 异常，直接在 bean 实例上设置回调对象
      Factory factory = (Factory) instance;
      factory.setCallbacks(new Callback[] {NoOp.INSTANCE,
              new LookupOverrideMethodInterceptor(this.beanDefinition, this.owner),
              new ReplaceOverrideMethodInterceptor(this.beanDefinition, this.owner)});
      return instance;
  }
  ```

  - 在 `<x>` 处，调用 `#createEnhancedSubclass(RootBeanDefinition beanDefinition)` 方法，为提供的 BeanDefinition 创建 bean 类的增强子类。代码如下：

    ```
    // CglibSubclassingInstantiationStrategy.java#CglibSubclassCreator
    
    private Class<?> createEnhancedSubclass(RootBeanDefinition beanDefinition) {
        // 创建 Enhancer 对象
        Enhancer enhancer = new Enhancer();
        // 设置 Bean 类
        enhancer.setSuperclass(beanDefinition.getBeanClass());
        // 设置 Spring 的命名策略
        enhancer.setNamingPolicy(SpringNamingPolicy.INSTANCE);
        // 设置生成策略
        if (this.owner instanceof ConfigurableBeanFactory) {
            ClassLoader cl = ((ConfigurableBeanFactory) this.owner).getBeanClassLoader();
            enhancer.setStrategy(new ClassLoaderAwareGeneratorStrategy(cl));
        }
        // 过滤，自定义逻辑来指定调用的callback下标
        enhancer.setCallbackFilter(new MethodOverrideCallbackFilter(beanDefinition));
        enhancer.setCallbackTypes(CALLBACK_TYPES);
        return enhancer.createClass();
    }
    ```

    - CGLIB 的标准 API 的使用。

  - `<y>` 处，获取子类增强 `subclass` 后，如果 Constructor 实例 `ctr` 为空，则调用默认构造函数（`BeanUtils#instantiateClass(subclass)`）来实例化类，否则则根据构造函数类型获取具体的构造器，调用 `Constructor#newInstance(args)` 方法来实例化类。

### 4.1 MethodOverrideCallbackFilter

在 `<x>` 处调用的 `#createEnhancedSubclass(RootBeanDefinition beanDefinition)` 方法，我们注意两行代码：

```
// CglibSubclassingInstantiationStrategy.java#CglibSubclassCreator

enhancer.setCallbackFilter(new MethodOverrideCallbackFilter(beanDefinition));
enhancer.setCallbackTypes(CALLBACK_TYPES);
```

- 通过 MethodOverrideCallbackFilter 来定义调用 callback 类型。

MethodOverrideCallbackFilter 是用来定义 CGLIB 回调过滤方法的拦截器行为，它继承 CglibIdentitySupport 实现 CallbackFilter 接口。

- CallbackFilter 是 CGLIB 的一个回调过滤器。
- CglibIdentitySupport 则为 CGLIB 提供 `#hashCode()` 和 `#equals(Object o)` 方法，以确保 CGLIB 不会为每个 Bean 生成不同的类。

MethodOverrideCallbackFilter 实现 CallbackFilter 的 `#accept(Method method)` 方法，代码如下：

```
// CglibSubclassingInstantiationStrategy.java#MethodOverrideCallbackFilter

@Override
public int accept(Method method) {
	MethodOverride methodOverride = getBeanDefinition().getMethodOverrides().getOverride(method);
	if (logger.isTraceEnabled()) {
		logger.trace("Override for '" + method.getName() + "' is [" + methodOverride + "]");
	}
	if (methodOverride == null) {
		return PASSTHROUGH;
	} else if (methodOverride instanceof LookupOverride) {
		return LOOKUP_OVERRIDE;
	} else if (methodOverride instanceof ReplaceOverride) {
		return METHOD_REPLACER;
	}
	throw new UnsupportedOperationException("Unexpected MethodOverride subclass: " +
			methodOverride.getClass().getName());
}
```

- 根据 BeanDefinition 中定义的 MethodOverride 不同，返回不同的值， 这里返回的 `PASSTHROUGH` 、`LOOKUP_OVERRIDE`、`METHOD_REPLACER` 都是 Callback 数组的**下标**，这里对应的数组为 `CALLBACK_TYPES` 数组，如下：

  ```
  // CglibSubclassingInstantiationStrategy.java#CglibSubclassCreator
  
  private static final Class<?>[] CALLBACK_TYPES = new Class<?>[] {
      NoOp.class,
      LookupOverrideMethodInterceptor.class,
      ReplaceOverrideMethodInterceptor.class
  };
  ```

  - 这里又定义了两个熟悉的拦截器 ：LookupOverrideMethodInterceptor 和 ReplaceOverrideMethodInterceptor，两个拦截器分别对应两个不同的 callback 业务。详细解析，见 [「4.2 LookupOverrideMethodInterceptor」](http://svip.iocoder.cn/Spring/IoC-InstantiationStrategy/#) 和 [「4.3 ReplaceOverrideMethodInterceptor」](http://svip.iocoder.cn/Spring/IoC-InstantiationStrategy/#) 中。

### 4.2 LookupOverrideMethodInterceptor

```
// CglibSubclassingInstantiationStrategy.java#LookupOverrideMethodInterceptor

private static class LookupOverrideMethodInterceptor extends CglibIdentitySupport implements MethodInterceptor {

    private final BeanFactory owner;

    public LookupOverrideMethodInterceptor(RootBeanDefinition beanDefinition, BeanFactory owner) {
        super(beanDefinition);
        this.owner = owner;
    }

    @Override
    public Object intercept(Object obj, Method method, Object[] args, MethodProxy mp) throws Throwable {
        // Cast is safe, as CallbackFilter filters are used selectively.
        // 获得 method 对应的 LookupOverride 对象
        LookupOverride lo = (LookupOverride) getBeanDefinition().getMethodOverrides().getOverride(method);
        Assert.state(lo != null, "LookupOverride not found");
        // 获得参数
        Object[] argsToUse = (args.length > 0 ? args : null);  // if no-arg, don't insist on args at all
        // 获得 Bean
        if (StringUtils.hasText(lo.getBeanName())) { // Bean 的名字
            return (argsToUse != null ? this.owner.getBean(lo.getBeanName(), argsToUse) :
                    this.owner.getBean(lo.getBeanName()));
        } else { // Bean 的类型
            return (argsToUse != null ? this.owner.getBean(method.getReturnType(), argsToUse) :
                    this.owner.getBean(method.getReturnType()));
        }
    }
}
```

### 4.3 ReplaceOverrideMethodInterceptor

```
// CglibSubclassingInstantiationStrategy.java#ReplaceOverrideMethodInterceptor

private static class ReplaceOverrideMethodInterceptor extends CglibIdentitySupport implements MethodInterceptor {

    private final BeanFactory owner;

    public ReplaceOverrideMethodInterceptor(RootBeanDefinition beanDefinition, BeanFactory owner) {
        super(beanDefinition);
        this.owner = owner;
    }

    @Override
    public Object intercept(Object obj, Method method, Object[] args, MethodProxy mp) throws Throwable {
        // 获得 method 对应的 LookupOverride 对象
        ReplaceOverride ro = (ReplaceOverride) getBeanDefinition().getMethodOverrides().getOverride(method);
        Assert.state(ro != null, "ReplaceOverride not found");
        // TODO could cache if a singleton for minor performance optimization
        // 获得 MethodReplacer 对象
        MethodReplacer mr = this.owner.getBean(ro.getMethodReplacerBeanName(), MethodReplacer.class);
        // 执行替换
        return mr.reimplement(obj, method, args);
    }
}
```

通过这两个拦截器，再加上这篇博客：[【死磕 Spring】—— IoC 之解析 bean 标签：meta、lookup-method、replace-method](http://svip.iocoder.cn/Spring/IoC-parse-BeanDefinitions-for-meta-and-look-method-and-replace-method)，是不是一道绝佳的美食。





# 11、总结

﻿在 【死磕 Spring】 Spring bean 解析篇深入分析了一个配置文件经历了哪些过程转变成了 BeanDefinition，但是这个 BeanDefinition 并不是我们真正想要的想要的 bean，因为它还仅仅只是承载了我们需要的目标 bean 的信息。

从 BeanDefinition 到我们需要的目标还需要一个漫长的 bean 的初始化阶段，在 【死磕 Spring】 Spring bean 加载阶段已经详细分析了初始化 bean 的过程，**所以这里做一个概括性的总结**。

bean 的初始化节点，由第一次(显式或者隐式)调用 `#getBean(...)` 方法来开启，所以我们从这个方法开始。代码如下：

```
// AbstractBeanFactory.java

public Object getBean(String name) throws BeansException {
    return doGetBean(name, null, null, false);
}

protected <T> T doGetBean(final String name, @Nullable final Class<T> requiredType,
        @Nullable final Object[] args, boolean typeCheckOnly) throws BeansException {
    // <1> 返回 bean 名称，剥离工厂引用前缀。
    // 如果 name 是 alias ，则获取对应映射的 beanName 。
    final String beanName = transformedBeanName(name);
    Object bean;

    // 从缓存中或者实例工厂中获取 Bean 对象
    // Eagerly check singleton cache for manually registered singletons.
    Object sharedInstance = getSingleton(beanName);
    if (sharedInstance != null && args == null) {
        if (logger.isTraceEnabled()) {
            if (isSingletonCurrentlyInCreation(beanName)) {
                logger.trace("Returning eagerly cached instance of singleton bean '" + beanName +
                        "' that is not fully initialized yet - a consequence of a circular reference");
            } else {
                logger.trace("Returning cached instance of singleton bean '" + beanName + "'");
            }
        }
        // <2> 完成 FactoryBean 的相关处理，并用来获取 FactoryBean 的处理结果
        bean = getObjectForBeanInstance(sharedInstance, name, beanName, null);
    } else {
        // Fail if we're already creating this bean instance:
        // We're assumably within a circular reference.
        // <3> 因为 Spring 只解决单例模式下得循环依赖，在原型模式下如果存在循环依赖则会抛出异常。
        if (isPrototypeCurrentlyInCreation(beanName)) {
            throw new BeanCurrentlyInCreationException(beanName);
        }

        // <4> 如果容器中没有找到，则从父类容器中加载
        // Check if bean definition exists in this factory.
        BeanFactory parentBeanFactory = getParentBeanFactory();
        if (parentBeanFactory != null && !containsBeanDefinition(beanName)) {
            // Not found -> check parent.
            String nameToLookup = originalBeanName(name);
            if (parentBeanFactory instanceof AbstractBeanFactory) {
                return ((AbstractBeanFactory) parentBeanFactory).doGetBean(
                        nameToLookup, requiredType, args, typeCheckOnly);
            } else if (args != null) {
                // Delegation to parent with explicit args.
                return (T) parentBeanFactory.getBean(nameToLookup, args);
            } else if (requiredType != null) {
                // No args -> delegate to standard getBean method.
                return parentBeanFactory.getBean(nameToLookup, requiredType);
            } else {
                return (T) parentBeanFactory.getBean(nameToLookup);
            }
        }

        // <5> 如果不是仅仅做类型检查则是创建bean，这里需要记录 
        if (!typeCheckOnly) {
            markBeanAsCreated(beanName);
        }

        try {
            // <6> 从容器中获取 beanName 相应的 GenericBeanDefinition 对象，并将其转换为 RootBeanDefinition 对象
            final RootBeanDefinition mbd = getMergedLocalBeanDefinition(beanName);
            // 检查给定的合并的 BeanDefinition
            checkMergedBeanDefinition(mbd, beanName, args);

            // Guarantee initialization of beans that the current bean depends on.
            // <7> 处理所依赖的 bean
            String[] dependsOn = mbd.getDependsOn();
            if (dependsOn != null) {
                for (String dep : dependsOn) {
                    // 若给定的依赖 bean 已经注册为依赖给定的 bean
                    // 循环依赖的情况
                    if (isDependent(beanName, dep)) {
                        throw new BeanCreationException(mbd.getResourceDescription(), beanName,
                                "Circular depends-on relationship between '" + beanName + "' and '" + dep + "'");
                    }
                    // 缓存依赖调用 TODO 芋艿
                    registerDependentBean(dep, beanName);
                    try {
                        getBean(dep);
                    } catch (NoSuchBeanDefinitionException ex) {
                        throw new BeanCreationException(mbd.getResourceDescription(), beanName,
                                "'" + beanName + "' depends on missing bean '" + dep + "'", ex);
                    }
                }
            }

            // <8> bean 实例化
            // Create bean instance.
            if (mbd.isSingleton()) { // 单例模式
                sharedInstance = getSingleton(beanName, () -> {
                    try {
                        return createBean(beanName, mbd, args);
                    }
                    catch (BeansException ex) {
                        // Explicitly remove instance from singleton cache: It might have been put there
                        // eagerly by the creation process, to allow for circular reference resolution.
                        // Also remove any beans that received a temporary reference to the bean.
                        // 显式从单例缓存中删除 Bean 实例
                        // 因为单例模式下为了解决循环依赖，可能他已经存在了，所以销毁它。 TODO 芋艿
                        destroySingleton(beanName);
                        throw ex;
                    }
                });
                bean = getObjectForBeanInstance(sharedInstance, name, beanName, mbd);
            } else if (mbd.isPrototype()) { // 原型模式
                // It's a prototype -> create a new instance.
                Object prototypeInstance;
                try {
                    beforePrototypeCreation(beanName);
                    prototypeInstance = createBean(beanName, mbd, args);
                } finally {
                    afterPrototypeCreation(beanName);
                }
                bean = getObjectForBeanInstance(prototypeInstance, name, beanName, mbd);
            } else {
                // 从指定的 scope 下创建 bean
                String scopeName = mbd.getScope();
                final Scope scope = this.scopes.get(scopeName);
                if (scope == null) {
                    throw new IllegalStateException("No Scope registered for scope name '" + scopeName + "'");
                }try {
                    Object scopedInstance = scope.get(beanName, () -> {
                        beforePrototypeCreation(beanName);
                        try {
                            return createBean(beanName, mbd, args);
                        } finally {
                            afterPrototypeCreation(beanName);
                        }
                    });
                    bean = getObjectForBeanInstance(scopedInstance, name, beanName, mbd);
                } catch (IllegalStateException ex) {
                    throw new BeanCreationException(beanName,
                            "Scope '" + scopeName + "' is not active for the current thread; consider " +
                            "defining a scoped proxy for this bean if you intend to refer to it from a singleton",
                            ex);
                }
            }
        } catch (BeansException ex) {
            cleanupAfterBeanCreationFailure(beanName);
            throw ex;
        }
    }

    // <9> 检查需要的类型是否符合 bean 的实际类型
    // Check if required type matches the type of the actual bean instance.
    if (requiredType != null && !requiredType.isInstance(bean)) {
        try {
            T convertedBean = getTypeConverter().convertIfNecessary(bean, requiredType);
            if (convertedBean == null) {
                throw new BeanNotOfRequiredTypeException(name, requiredType, bean.getClass());
            }
            return convertedBean;
        } catch (TypeMismatchException ex) {
            if (logger.isTraceEnabled()) {
                logger.trace("Failed to convert bean '" + name + "' to required type '" +
                        ClassUtils.getQualifiedName(requiredType) + "'", ex);
            }
            throw new BeanNotOfRequiredTypeException(name, requiredType, bean.getClass());
        }
    }
    return (T) bean;
}
```

内部调用 `#doGetBean(...)` 方法。`#doGetBean(...)` 方法的代码量比较多，从这里就可以看出 bean 的加载过程是一个非常复杂的过程，会涉及到各种各样的情况处理。

`#doGetBean(...)` 方法，可以分为以下几个过程：

1. 转换 beanName 。因为我们调用 `#getBean(...)` 方法传入的 `name` 并不一定就是 beanName，可以传入 aliasName，FactoryBean，所以这里需要进行简单的转换过程。
2. 尝试从缓存中加载单例 bean 。
3. bean 的实例化。
4. 原型模式的依赖检查。因为 Spring 只会解决单例模式的循环依赖，对于原型模式的循环依赖都是直接抛出 BeanCurrentlyInCreationException 异常。
5. 尝试从 parentBeanFactory 获取 bean 实例。如果 `parentBeanFactory != null && !containsBeanDefinition(beanName)` 则尝试从 parentBeanFactory 中获取 bean 实例对象，因为 `!containsBeanDefinition(beanName)` 就意味着定义的 xml 文件中没有 beanName 相应的配置，这个时候就只能从 `parentBeanFactory` 中获取。
6. 获取 RootBeanDefinition，并对其进行合并检查。从缓存中获取已经解析的 RootBeanDefinition 。同时，如果父类不为 `null` 的话，则会合并父类的属性。
7. 依赖检查。某个 bean 依赖其他 bean ，则需要先加载依赖的 bean。
8. 对不同的 scope 进行处理。
9. 类型转换处理。如果传递的 `requiredType` 不为 `null`，则需要检测所得到 bean 的类型是否与该 `requiredType` 一致。如果不一致则尝试转换，当然也要能够转换成功，否则抛出 BeanNotOfRequiredTypeException 异常。

下面就以下几个方面进行阐述，说明 Spring bean 的加载过程。

1. 从缓存中获取 bean
2. 创建 bean 实例对象
3. 从 bean 实例中获取对象

## 1. 从缓存中获取 bean

Spring 中根据 scope 可以将 bean 分为以下几类：singleton、prototype 和 其他，这样分的原因在于 Spring 在对不同 scope 处理的时候是这么处理的：

- singleton ：在 Spring 的 IoC 容器中只存在一个对象实例，所有该对象的引用都共享这个实例。Spring 容器只会创建该 bean 定义的唯一实例，这个实例会被保存到缓存中，并且对该bean的所有后续请求和引用都将返回该缓存中的对象实例。
- prototype ：每次对该bean的请求都会创建一个新的实例
- 其他 ：
  - request：每次 http 请求将会有各自的 bean 实例。
  - session：在一个 http session 中，一个 bean 定义对应一个 bean 实例。
  - global session：在一个全局的 http session 中，一个 bean 定义对应一个 bean 实例。

所以，从缓存中获取的 bean 一定是 singleton bean，这也是 **Spring 为何只解决 singleton bean 的循环依赖**。调用 `#getSingleton(String beanName)` 方法，从缓存中获取 singleton bean。代码如下：

```
// DefaultSingletonBeanRegistry.java

public Object getSingleton(String beanName) {
    return getSingleton(beanName, true);
}

@Nullable
protected Object getSingleton(String beanName, boolean allowEarlyReference) {
    // 从单例缓冲中加载 bean
    Object singletonObject = this.singletonObjects.get(beanName);
    // 缓存中的 bean 为空，且当前 bean 正在创建
    if (singletonObject == null && isSingletonCurrentlyInCreation(beanName)) {
        // 加锁
        synchronized (this.singletonObjects) {
            // 从 earlySingletonObjects 获取
            singletonObject = this.earlySingletonObjects.get(beanName);
            // earlySingletonObjects 中没有，且允许提前创建
            if (singletonObject == null && allowEarlyReference) {
                // 从 singletonFactories 中获取对应的 ObjectFactory
                ObjectFactory<?> singletonFactory = this.singletonFactories.get(beanName);
                if (singletonFactory != null) {
                    // 获得 bean
                    singletonObject = singletonFactory.getObject();
                    // 添加 bean 到 earlySingletonObjects 中
                    this.earlySingletonObjects.put(beanName, singletonObject);
                    // 从 singletonFactories 中移除对应的 ObjectFactory
                    this.singletonFactories.remove(beanName);
                }
            }
        }
    }
    return singletonObject;
}
```

- 该方法就是从

   

  ```
  singletonObjects
  ```

  、

  ```
  earlySingletonObjects
  ```

  、

   

  ```
  singletonFactories
  ```

   

  三个缓存中获取，这里也是 Spring 解决 bean 循环依赖的关键之处。详细内容请查看如下内容：

  - [【死磕 Spring】—— IoC 之加载 bean：从单例缓存中获取单例 bean](http://svip.iocoder.cn/Spring/IoC-get-Bean-getSingleton)
  - [【死磕 Spring】—— IoC 之加载 bean：创建 bean（五）之循环依赖处理](http://svip.iocoder.cn/Spring/IoC-get-Bean-createBean-5)

## 2. 创建 bean 实例对象

如果缓存中没有，也没有 `parentBeanFactory` ，则会调用 `#createBean(String beanName, RootBeanDefinition mbd, Object[] args)` 方法，创建 bean 实例。该方法主要是在处理不同 scope 的 bean 的时候进行调用。代码如下：

```
// AbstractBeanFactory.java

protected abstract Object createBean(String beanName, RootBeanDefinition mbd, @Nullable Object[] args)
            throws BeanCreationException;
```

- 该方法是定义在 AbstractBeanFactory 中的**抽象**方法，其含义是根据给定的 BeanDefinition 和 `args` 实例化一个 bean 对象。如果该 BeanDefinition 存在父类，则该 BeanDefinition 已经合并了父类的属性。所有 Bean 实例的创建都会委托给该方法实现。
- 方法接受三个参数：
  - `beanName` ：bean 的名字。
  - `mbd` ：已经合并了父类属性的（如果有的话）BeanDefinition 。
  - `args` ：用于构造函数或者工厂方法创建 bean 实例对象的参数 。

------

该抽象方法的**默认实现**是在类 AbstractAutowireCapableBeanFactory 中实现，该方法其实只是做一些检查和验证工作，真正的初始化工作是由 `#doCreateBean(final String beanName, final RootBeanDefinition mbd, final Object[] args)` 方法来实现。代码如下：

```
// AbstractAutowireCapableBeanFactory.java

protected Object doCreateBean(final String beanName, final RootBeanDefinition mbd, final @Nullable Object[] args)
        throws BeanCreationException {

    // Instantiate the bean.
    // BeanWrapper 是对 Bean 的包装，其接口中所定义的功能很简单包括设置获取被包装的对象，获取被包装 bean 的属性描述器
    BeanWrapper instanceWrapper = null;
    // <1> 单例模型，则从未完成的 FactoryBean 缓存中删除
    if (mbd.isSingleton()) {
        instanceWrapper = this.factoryBeanInstanceCache.remove(beanName);
    }
    // <2> 使用合适的实例化策略来创建新的实例：工厂方法、构造函数自动注入、简单初始化
    if (instanceWrapper == null) {
        instanceWrapper = createBeanInstance(beanName, mbd, args);
    }
    // 包装的实例对象
    final Object bean = instanceWrapper.getWrappedInstance();
    // 包装的实例对象的类型
    Class<?> beanType = instanceWrapper.getWrappedClass();
    if (beanType != NullBean.class) {
        mbd.resolvedTargetType = beanType;
    }

    // Allow post-processors to modify the merged bean definition.
    // <3> 判断是否有后置处理
    // 如果有后置处理，则允许后置处理修改 BeanDefinition
    synchronized (mbd.postProcessingLock) {
        if (!mbd.postProcessed) {
            try {
                // 后置处理修改 BeanDefinition
                applyMergedBeanDefinitionPostProcessors(mbd, beanType, beanName);
            } catch (Throwable ex) {
                throw new BeanCreationException(mbd.getResourceDescription(), beanName,
                        "Post-processing of merged bean definition failed", ex);
            }
            mbd.postProcessed = true;
        }
    }

    // Eagerly cache singletons to be able to resolve circular references
    // even when triggered by lifecycle interfaces like BeanFactoryAware.
    // <4> 解决单例模式的循环依赖
    boolean earlySingletonExposure = (mbd.isSingleton() // 单例模式
            && this.allowCircularReferences // 运行循环依赖
            && isSingletonCurrentlyInCreation(beanName)); // 当前单例 bean 是否正在被创建
    if (earlySingletonExposure) {
        if (logger.isTraceEnabled()) {
            logger.trace("Eagerly caching bean '" + beanName +
                    "' to allow for resolving potential circular references");
        }
        // 提前将创建的 bean 实例加入到 singletonFactories 中
        // 这里是为了后期避免循环依赖
        addSingletonFactory(beanName, () -> getEarlyBeanReference(beanName, mbd, bean));
    }

    // Initialize the bean instance.
    // 开始初始化 bean 实例对象
    Object exposedObject = bean;
    try {
        // <5> 对 bean 进行填充，将各个属性值注入，其中，可能存在依赖于其他 bean 的属性
        // 则会递归初始依赖 bean
        populateBean(beanName, mbd, instanceWrapper);
        // <6> 调用初始化方法
        exposedObject = initializeBean(beanName, exposedObject, mbd);
    } catch (Throwable ex) {
        if (ex instanceof BeanCreationException && beanName.equals(((BeanCreationException) ex).getBeanName())) {
            throw (BeanCreationException) ex;
        } else {
            throw new BeanCreationException(
                    mbd.getResourceDescription(), beanName, "Initialization of bean failed", ex);
        }
    }

    // <7> 循环依赖处理
    if (earlySingletonExposure) {
        // 获取 earlySingletonReference
        Object earlySingletonReference = getSingleton(beanName, false);
        // 只有在存在循环依赖的情况下，earlySingletonReference 才不会为空
        if (earlySingletonReference != null) {
            // 如果 exposedObject 没有在初始化方法中被改变，也就是没有被增强
            if (exposedObject == bean) {
                exposedObject = earlySingletonReference;
            // 处理依赖
            } else if (!this.allowRawInjectionDespiteWrapping && hasDependentBean(beanName)) {
                String[] dependentBeans = getDependentBeans(beanName);
                Set<String> actualDependentBeans = new LinkedHashSet<>(dependentBeans.length);
                for (String dependentBean : dependentBeans) {
                    if (!removeSingletonIfCreatedForTypeCheckOnly(dependentBean)) {
                        actualDependentBeans.add(dependentBean);
                    }
                }
                if (!actualDependentBeans.isEmpty()) {
                    throw new BeanCurrentlyInCreationException(beanName,
                            "Bean with name '" + beanName + "' has been injected into other beans [" +
                            StringUtils.collectionToCommaDelimitedString(actualDependentBeans) +
                            "] in its raw version as part of a circular reference, but has eventually been " +
                            "wrapped. This means that said other beans do not use the final version of the " +
                            "bean. This is often the result of over-eager type matching - consider using " +
                            "'getBeanNamesOfType' with the 'allowEagerInit' flag turned off, for example.");
                }
            }
        }
    }

    // Register bean as disposable.
    // <8> 注册 bean
    try {
        registerDisposableBeanIfNecessary(beanName, bean, mbd);
    } catch (BeanDefinitionValidationException ex) {
        throw new BeanCreationException(
                mbd.getResourceDescription(), beanName, "Invalid destruction signature", ex);
    }

    return exposedObject;
}
```

`#doCreateBean(...)` 方法，是创建 bean 实例的核心方法，它的整体思路是：

- `<1>` 处，如果是单例模式，则清除 `factoryBeanInstanceCache 缓存`，同时返回 BeanWrapper 实例对象，当然如果存在。
- `<2>` 处，如果缓存中没有 BeanWrapper 或者不是单例模式，则调用 `#createBeanInstance(...)` 方法，实例化 bean，主要是将 BeanDefinition 转换为 BeanWrapper 。
- `<3>` 处，MergedBeanDefinitionPostProcessor 的应用。
- `<4>` 处，单例模式的循环依赖处理。
- `<5>` 处，调用 `#populateBean(String beanName, RootBeanDefinition mbd, BeanWrapper bw)` 方法，进行属性填充。将所有属性填充至 bean 的实例中。
- `<6>` 处，调用 `#initializeBean(final String beanName, final Object bean, RootBeanDefinition mbd)` 方法，初始化 bean 。
- `<7>` 处，依赖检查。
- `<8>` 处，注册 DisposableBean 。

#### 2.1 实例化 bean

如果缓存中没有 BeanWrapper 实例对象或者该 bean 不是 singleton，则调用 `#createBeanInstance(...)` 方法。创建 bean 实例。该方法主要是根据参数 BeanDefinition、`args[]` 来调用构造函数实例化 bean 对象。**过程较为复杂**，代码如下：

```
// AbstractAutowireCapableBeanFactory.java

protected BeanWrapper createBeanInstance(String beanName, RootBeanDefinition mbd, @Nullable Object[] args) {
    // Make sure bean class is actually resolved at this point.
    // 解析 bean ，将 bean 类名解析为 class 引用。
    Class<?> beanClass = resolveBeanClass(mbd, beanName);

    if (beanClass != null && !Modifier.isPublic(beanClass.getModifiers()) && !mbd.isNonPublicAccessAllowed()) { // 校验
        throw new BeanCreationException(mbd.getResourceDescription(), beanName,
                "Bean class isn't public, and non-public access not allowed: " + beanClass.getName());
    }

    // <1> 如果存在 Supplier 回调，则使用给定的回调方法初始化策略
    Supplier<?> instanceSupplier = mbd.getInstanceSupplier();
    if (instanceSupplier != null) {
        return obtainFromSupplier(instanceSupplier, beanName);
    }

    // <2> 使用 FactoryBean 的 factory-method 来创建，支持静态工厂和实例工厂
    if (mbd.getFactoryMethodName() != null)  {
        return instantiateUsingFactoryMethod(beanName, mbd, args);
    }

    // <3> Shortcut when re-creating the same bean...
    boolean resolved = false;
    boolean autowireNecessary = false;
    if (args == null) {
        // constructorArgumentLock 构造函数的常用锁
        synchronized (mbd.constructorArgumentLock) {
            // 如果已缓存的解析的构造函数或者工厂方法不为空，则可以利用构造函数解析
            // 因为需要根据参数确认到底使用哪个构造函数，该过程比较消耗性能，所有采用缓存机制
            if (mbd.resolvedConstructorOrFactoryMethod != null) {
                resolved = true;
                autowireNecessary = mbd.constructorArgumentsResolved;
            }
        }
    }
    // 已经解析好了，直接注入即可
    if (resolved) {
        // <3.1> autowire 自动注入，调用构造函数自动注入
        if (autowireNecessary) {
            return autowireConstructor(beanName, mbd, null, null);
        } else {
            // <3.2> 使用默认构造函数构造
            return instantiateBean(beanName, mbd);
        }
    }

    // Candidate constructors for autowiring?
    // <4> 确定解析的构造函数
    // 主要是检查已经注册的 SmartInstantiationAwareBeanPostProcessor
    Constructor<?>[] ctors = determineConstructorsFromBeanPostProcessors(beanClass, beanName);
    // <4.1> 有参数情况时，创建 Bean 。先利用参数个数，类型等，确定最精确匹配的构造方法。
    if (ctors != null || mbd.getResolvedAutowireMode() == AUTOWIRE_CONSTRUCTOR ||
            mbd.hasConstructorArgumentValues() || !ObjectUtils.isEmpty(args))  {
        return autowireConstructor(beanName, mbd, ctors, args);
    }

    // Preferred constructors for default construction?
    // <4.1> 选择构造方法，创建 Bean 。
    ctors = mbd.getPreferredConstructors();
    if (ctors != null) {
        return autowireConstructor(beanName, mbd, ctors, null); // args = null
    }

    // No special handling: simply use no-arg constructor.
    // <4.2> 有参数时，又没获取到构造方法，则只能调用无参构造方法来创建实例了(兜底方法)
    return instantiateBean(beanName, mbd);
}
```

实例化 Bean 对象，是一个**复杂**的过程，其主要的逻辑为：

- `<1>` 处，如果存在 Supplier 回调，则调用 `#obtainFromSupplier(Supplier<?> instanceSupplier, String beanName)` 方法，进行初始化。

- `<2>` 处，如果存在工厂方法，则使用工厂方法进行初始化。

- ```
  <3>
  ```

   

  处，首先判断缓存，如果

  缓存中存在

  ，即已经解析过了，则直接使用已经解析了的。根据

   

  ```
  constructorArgumentsResolved
  ```

   

  参数来判断：

  - `<3.1>` 处，是使用构造函数自动注入，即调用 `#autowireConstructor(String beanName, RootBeanDefinition mbd, Constructor<?>[] ctors, Object[] explicitArgs)` 方法。
  - `<3.2>` 处，还是默认构造函数，即调用 `#instantiateBean(final String beanName, final RootBeanDefinition mbd)` 方法。

- ```
  <4>
  ```

   

  处，如果

  缓存中没有

  ，则需要先确定到底使用哪个构造函数来完成解析工作，因为一个类有多个构造函数，每个构造函数都有不同的构造参数，所以需要根据参数来锁定构造函数并完成初始化。

  - `<4.1>` 处，如果存在参数，则使用相应的带有参数的构造函数，即调用 `#autowireConstructor(String beanName, RootBeanDefinition mbd, Constructor<?>[] ctors, Object[] explicitArgs)` 方法。
  - `<4.2>` 处，否则，使用默认构造函数，即调用 `#instantiateBean(final String beanName, final RootBeanDefinition mbd)` 方法。

其实核心思想还是在于根据不同的情况执行不同的实例化策略，主要是包括如下四种策略：

1. Supplier 回调
2. `#instantiateUsingFactoryMethod(...)` 方法，工厂方法初始化
3. `#autowireConstructor(...)` 方法，构造函数自动注入初始化
4. `#instantiateBean(...)` 方法，默认构造函数注入

其实无论哪种策略，他们的实现逻辑都差不多：确定构造函数和构造方法，然后实例化。只不过相对于 Supplier 回调和默认构造函数注入而言，工厂方法初始化和构造函数自动注入初始化会比较复杂，因为他们构造函数和构造参数的不确定性，Spring 需要花大量的精力来确定构造函数和构造参数，如果确定了则好办，直接选择实例化策略即可。当然在实例化的时候会根据是否有需要覆盖或者动态替换掉的方法，因为存在覆盖或者织入的话需要创建动态代理将方法织入，这个时候就只能选择 CGLIB 的方式来实例化，否则直接利用反射的方式即可。

#### 2.2 属性填充

属性填充其实就是将 BeanDefinition 的属性值赋值给 BeanWrapper 实例对象的过程。在填充的过程需要根据注入的类型不同来区分是根据类型注入还是名字注入，当然在这个过程还会涉及循环依赖的问题的。代码如下：

```
// AbstractAutowireCapableBeanFactory.java

protected void populateBean(String beanName, RootBeanDefinition mbd, @Nullable BeanWrapper bw) {
    // 没有实例化对象
    if (bw == null) {
        // 有属性，则抛出 BeanCreationException 异常
        if (mbd.hasPropertyValues()) {
            throw new BeanCreationException(
                    mbd.getResourceDescription(), beanName, "Cannot apply property values to null instance");
            // 没有属性，直接 return 返回
        } else {
            // Skip property population phase for null instance.
            return;
        }
    }

    // <1> 在设置属性之前给 InstantiationAwareBeanPostProcessors 最后一次改变 bean 的机会
    // Give any InstantiationAwareBeanPostProcessors the opportunity to modify the
    // state of the bean before properties are set. This can be used, for example,
    // to support styles of field injection.
    boolean continueWithPropertyPopulation = true;
    if (!mbd.isSynthetic()  // bean 不是"合成"的，即未由应用程序本身定义
            && hasInstantiationAwareBeanPostProcessors()) { // 是否持有 InstantiationAwareBeanPostProcessor
        // 迭代所有的 BeanPostProcessors
        for (BeanPostProcessor bp : getBeanPostProcessors()) {
            if (bp instanceof InstantiationAwareBeanPostProcessor) { // 如果为 InstantiationAwareBeanPostProcessor
                InstantiationAwareBeanPostProcessor ibp = (InstantiationAwareBeanPostProcessor) bp;
                // 返回值为是否继续填充 bean
                // postProcessAfterInstantiation：如果应该在 bean上面设置属性则返回 true，否则返回 false
                // 一般情况下，应该是返回true 。
                // 返回 false 的话，将会阻止在此 Bean 实例上调用任何后续的 InstantiationAwareBeanPostProcessor 实例。
                if (!ibp.postProcessAfterInstantiation(bw.getWrappedInstance(), beanName)) {
                    continueWithPropertyPopulation = false;
                    break;
                }
            }
        }
    }
    // 如果后续处理器发出停止填充命令，则终止后续操作
    if (!continueWithPropertyPopulation) {
        return;
    }

    // bean 的属性值
    PropertyValues pvs = (mbd.hasPropertyValues() ? mbd.getPropertyValues() : null);

    // <2> 自动注入
    if (mbd.getResolvedAutowireMode() == AUTOWIRE_BY_NAME || mbd.getResolvedAutowireMode() == AUTOWIRE_BY_TYPE) {
        // 将 PropertyValues 封装成 MutablePropertyValues 对象
        // MutablePropertyValues 允许对属性进行简单的操作，并提供构造函数以支持Map的深度复制和构造。
        MutablePropertyValues newPvs = new MutablePropertyValues(pvs);
        // Add property values based on autowire by name if applicable.
        // 根据名称自动注入
        if (mbd.getResolvedAutowireMode() == AUTOWIRE_BY_NAME) {
            autowireByName(beanName, mbd, bw, newPvs);
        }
        // Add property values based on autowire by type if applicable.
        // 根据类型自动注入
        if (mbd.getResolvedAutowireMode() == AUTOWIRE_BY_TYPE) {
            autowireByType(beanName, mbd, bw, newPvs);
        }
        pvs = newPvs;
    }

    // 是否已经注册了 InstantiationAwareBeanPostProcessors
    boolean hasInstAwareBpps = hasInstantiationAwareBeanPostProcessors();
    // 是否需要进行【依赖检查】
    boolean needsDepCheck = (mbd.getDependencyCheck() != AbstractBeanDefinition.DEPENDENCY_CHECK_NONE);

    // <3> BeanPostProcessor 处理
    PropertyDescriptor[] filteredPds = null;
    if (hasInstAwareBpps) {
        if (pvs == null) {
            pvs = mbd.getPropertyValues();
        }
        // 遍历 BeanPostProcessor 数组
        for (BeanPostProcessor bp : getBeanPostProcessors()) {
            if (bp instanceof InstantiationAwareBeanPostProcessor) {
                InstantiationAwareBeanPostProcessor ibp = (InstantiationAwareBeanPostProcessor) bp;
                // 对所有需要依赖检查的属性进行后处理
                PropertyValues pvsToUse = ibp.postProcessProperties(pvs, bw.getWrappedInstance(), beanName);
                if (pvsToUse == null) {
                    // 从 bw 对象中提取 PropertyDescriptor 结果集
                    // PropertyDescriptor：可以通过一对存取方法提取一个属性
                    if (filteredPds == null) {
                        filteredPds = filterPropertyDescriptorsForDependencyCheck(bw, mbd.allowCaching);
                    }
                    pvsToUse = ibp.postProcessPropertyValues(pvs, filteredPds, bw.getWrappedInstance(), beanName);
                    if (pvsToUse == null) {
                        return;
                    }
                }
                pvs = pvsToUse;
            }
        }
    }
    
    // <4> 依赖检查
    if (needsDepCheck) {
        if (filteredPds == null) {
            filteredPds = filterPropertyDescriptorsForDependencyCheck(bw, mbd.allowCaching);
        }
        // 依赖检查，对应 depends-on 属性
        checkDependencies(beanName, mbd, filteredPds, pvs);
    }

    // <5> 将属性应用到 bean 中
    if (pvs != null) {
        applyPropertyValues(beanName, mbd, bw, pvs);
    }
}
```

处理流程如下：

- `<1>` ，根据 `hasInstantiationAwareBeanPostProcessors` 属性来判断，是否需要在注入属性之前给 InstantiationAwareBeanPostProcessors 最后一次改变 bean 的机会。**此过程可以控制 Spring 是否继续进行属性填充**。

- 统一存入到 PropertyValues 中，PropertyValues 用于描述 bean 的属性。

  - ```
    <2>
    ```

     

    ，根据注入类型(

     

    ```
    AbstractBeanDefinition#getResolvedAutowireMode()
    ```

     

    方法的返回值 )的不同来判断：

    - 是根据名称来自动注入（`#autowireByName(...)`）
    - 还是根据类型来自动注入（`#autowireByType(...)`）

  - `<3>` ，进行 BeanPostProcessor 处理。

  - `<4>` ，依赖检测。

- `<5>` ，将所有 PropertyValues 中的属性，填充到 BeanWrapper 中。

#### 2.3 初始化 bean

初始化 bean 为 `#createBean(...)` 方法的最后一个过程，代码如下：

```
// AbstractAutowireCapableBeanFactory.java

protected Object initializeBean(final String beanName, final Object bean, @Nullable RootBeanDefinition mbd) {
    if (System.getSecurityManager() != null) { // 安全模式
        AccessController.doPrivileged((PrivilegedAction<Object>) () -> {
            // <1> 激活 Aware 方法，对特殊的 bean 处理：Aware、BeanClassLoaderAware、BeanFactoryAware
            invokeAwareMethods(beanName, bean);
            return null;
        }, getAccessControlContext());
    } else {
        // <1> 激活 Aware 方法，对特殊的 bean 处理：Aware、BeanClassLoaderAware、BeanFactoryAware
        invokeAwareMethods(beanName, bean);
    }

    // <2> 后处理器，before
    Object wrappedBean = bean;
    if (mbd == null || !mbd.isSynthetic()) {
        wrappedBean = applyBeanPostProcessorsBeforeInitialization(wrappedBean, beanName);
    }

    // <3> 激活用户自定义的 init 方法
    try {
        invokeInitMethods(beanName, wrappedBean, mbd);
    } catch (Throwable ex) {
        throw new BeanCreationException(
                (mbd != null ? mbd.getResourceDescription() : null),
                beanName, "Invocation of init method failed", ex);
    }

    // <2> 后处理器，after
    if (mbd == null || !mbd.isSynthetic()) {
        wrappedBean = applyBeanPostProcessorsAfterInitialization(wrappedBean, beanName);
    }

    return wrappedBean;
}
```

初始化 bean 的方法其实就是三个步骤的处理，而这三个步骤主要还是根据**用户设定**的来进行初始化，这三个过程为：

- `<1>` 激活 Aware 方法。
- `<3>` 后置处理器的应用。
- `<2>` 激活自定义的 init 方法。

## 3. 从 bean 实例中获取对象

无论是从单例缓存中获取的 bean 实例 还是通过 `#createBean(...)` 方法来创建的 bean 实例，最终都会调用 `#getObjectForBeanInstance(...)` 方法来根据传入的 bean 实例获取对象，按照 Spring 的传统，该方法也只是做一些检测工作，真正的实现逻辑是委托给 `#getObjectFromFactoryBean(...)` 方法来实现。代码如下：

```
protected Object getObjectFromFactoryBean(FactoryBean<?> factory, String beanName, boolean shouldPostProcess) {
    // <1> 为单例模式且缓存中存在
    if (factory.isSingleton() && containsSingleton(beanName)) {
        synchronized (getSingletonMutex()) { // <1.1> 单例锁
            // <1.2> 从缓存中获取指定的 factoryBean
            Object object = this.factoryBeanObjectCache.get(beanName);
            if (object == null) {
                // 为空，则从 FactoryBean 中获取对象
                object = doGetObjectFromFactoryBean(factory, beanName);
                // 从缓存中获取
                // TODO 芋艿，具体原因
                // Only post-process and store if not put there already during getObject() call above
                // (e.g. because of circular reference processing triggered by custom getBean calls)
                Object alreadyThere = this.factoryBeanObjectCache.get(beanName);
                if (alreadyThere != null) {
                    object = alreadyThere;
                } else {
                    // <1.3> 需要后续处理
                    if (shouldPostProcess) {
                        // 若该 Bean 处于创建中，则返回非处理对象，而不是存储它
                        if (isSingletonCurrentlyInCreation(beanName)) {
                            // Temporarily return non-post-processed object, not storing it yet..
                            return object;
                        }
                        // 单例 Bean 的前置处理
                        beforeSingletonCreation(beanName);
                        try {
                            // 对从 FactoryBean 获取的对象进行后处理
                            // 生成的对象将暴露给 bean 引用
                            object = postProcessObjectFromFactoryBean(object, beanName);
                        } catch (Throwable ex) {
                            throw new BeanCreationException(beanName,
                                    "Post-processing of FactoryBean's singleton object failed", ex);
                        } finally {
                            // 单例 Bean 的后置处理
                            afterSingletonCreation(beanName);
                        }
                    }
                    // <1.4> 添加到 factoryBeanObjectCache 中，进行缓存
                    if (containsSingleton(beanName)) {
                        this.factoryBeanObjectCache.put(beanName, object);
                    }
                }
            }
            return object;
        }
    // <2>
    } else {
        // 为空，则从 FactoryBean 中获取对象
        Object object = doGetObjectFromFactoryBean(factory, beanName);
        // 需要后续处理
        if (shouldPostProcess) {
            try {
                // 对从 FactoryBean 获取的对象进行后处理
                // 生成的对象将暴露给 bean 引用
                object = postProcessObjectFromFactoryBean(object, beanName);
            }
            catch (Throwable ex) {
                throw new BeanCreationException(beanName, "Post-processing of FactoryBean's object failed", ex);
            }
        }
        return object;
    }
}
```

主要流程如下：

- 若为单例且单例 Bean 缓存中存在 `beanName` ，则 `<1>` 进行后续处理（跳转到下一步），否则，则 `<2>` 从 FactoryBean 中获取 Bean 实例对象。

- `<1.1>` 首先，获取锁。其实我们在前面篇幅中发现了大量的同步锁，锁住的对象都是 `this.singletonObjects`，主要是因为在单例模式中必须要**保证全局唯一**。

- `<1.2>` 然后，从 `factoryBeanObjectCache` 缓存中获取实例对象 `object` 。若 `object` 为空，则调用 `#doGetObjectFromFactoryBean(FactoryBean<?> factory, String beanName)` 方法，从 FactoryBean 获取对象，其实内部就是调用 `FactoryBean#getObject()` 方法。

- ```
  <1.3>
  ```

   

  如果需要后续处理(

   

  ```
  shouldPostProcess = true
  ```

   

  )，则进行进一步处理，步骤如下：

  - 若该 Bean 处于创建中（`#isSingletonCurrentlyInCreation(String beanName)` 方法返回 `true` ），则返回**非处理的 Bean 对象**，而不是存储它。
  - 调用 `#beforeSingletonCreation(String beanName)` 方法，进行创建之前的处理。默认实现将该 Bean 标志为当前创建的。
  - 调用 `#postProcessObjectFromFactoryBean(Object object, String beanName)` 方法，对从 FactoryBean 获取的 Bean 实例对象进行后置处理。
  - 调用 `#afterSingletonCreation(String beanName)` 方法，进行创建 Bean 之后的处理，默认实现是将该 bean 标记为不再在创建中。

- `<1.4>` 最后，加入到 `factoryBeanObjectCache` 缓存中。