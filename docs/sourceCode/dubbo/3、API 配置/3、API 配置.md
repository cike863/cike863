# API 配置（一）之应用

### 1. 概述

我们都“知道”，Dubbo 的配置是非常“灵活”的。

例如，目前提供了四种**配置方式**：

1. API 配置
2. 属性配置
3. XML 配置
4. 注解配置

ps：🙂 后续的几篇文章也是按照这样的顺序，解析 Dubbo 配置的源码。

再例如，可灵活设置的**配置项**：

> FROM [《Dubbo 用户指南 —— schema 配置参考手册》](http://dubbo.apache.org/zh-cn/docs/user/references/xml/introduction.html)
>
> 所有配置项分为三大类，参见下表中的”作用”一列。
>
> - 服务发现：表示该配置项用于服务的注册与发现，目的是让消费方找到提供方。
> - 服务治理：表示该配置项用于治理服务间的关系，或为开发测试提供便利条件。
> - 性能调优：表示该配置项用于调优性能，不同的选项对性能会产生影响。
>
> 所有配置最终都将转换为 Dubbo URL 表示，并由服务提供方生成，经注册中心传递给消费方，各属性对应 URL 的参数，参见配置项一览表中的 “对应URL参数” 列。

ps：🙂 可能转换成 Dubbo [URL](https://github.com/YunaiV/dubbo/blob/9d38b6f9f95798755141d6140e311e8fd51fecc1/dubbo-common/src/main/java/com/alibaba/dubbo/common/URL.java) 不太好理解。良心如笔者，后续有文章会贯串它。

当然，凡事都有两面性，在社区里也存在建议的声音，例如：[《ISSUE#738：XML配置项重新梳理》](https://github.com/alibaba/dubbo/issues/738) ：

> 目前有一些配置项存在暴露的位置不正确、暴露不全面、文档和含义不匹配等问题，期望在2.5.7版本将已知问题予以整理修复
>
> **如果使用中有遇到的配置问题，请在评论中列出以便改进**

### 2. 配置一览

我们来看看 `dubbo-config-api` 的**项目结构**，如下图所示：

[![dubbo-config-api 项目结构](assets/01.png)](http://static.iocoder.cn/images/Dubbo/2018_01_07/01.png)dubbo-config-api 项目结构

一脸懵逼，好多啊。下面我们来整理下**配置之间的关系**，如下图所示：

> FROM [《Dubbo 用户指南 —— XML 配置》](http://dubbo.apache.org/zh-cn/docs/user/configuration/xml.html)
> [![配置之间的关系](assets/02.png)](http://static.iocoder.cn/images/Dubbo/2018_01_07/02.png)配置之间的关系

从这张图中，可以看出分成**四个**部分：

1. application-shared
2. provider-side
3. consumer-side
4. sub-config

实际上，上图和目前版本的代码会存在一点点出入，我们在看看实际的**类关系**，如下图所示：

[![配置类关系](assets/03.png)](http://static.iocoder.cn/images/Dubbo/2018_01_07/03.png)配置类关系

- **红勾**部分，application-shared ，在本文进行分享。
- **黄框**部分，provider-side ，在 [《API 配置（二）之服务提供者》](http://svip.iocoder.cn/Dubbo/configuration-api-2/?self) 分享。
- **红框**部分，consumer-side ，在 [《API 配置（三）之服务消费者》](http://svip.iocoder.cn/Dubbo/configuration-api-3/?self) 分享。
- **其他**部分，sub-config ，在 [《API 配置（二）之服务提供者》](http://svip.iocoder.cn/Dubbo/configuration-api-2/?self) 分享。

### 3. Config

我们先来看一段 [《Dubbo 用户指南 —— API 配置》](http://dubbo.apache.org/zh-cn/docs/user/configuration/api.html) ，提供的消费者的初始化代码：

```
// 当前应用配置
ApplicationConfig application = new ApplicationConfig();
application.setName("yyy");

// 连接注册中心配置
RegistryConfig registry = new RegistryConfig();
registry.setAddress("10.20.130.230:9090");
registry.setUsername("aaa");
registry.setPassword("bbb");

// 注意：ReferenceConfig为重对象，内部封装了与注册中心的连接，以及与服务提供方的连接

// 引用远程服务
ReferenceConfig<XxxService> reference = new ReferenceConfig<XxxService>(); // 此实例很重，封装了与注册中心的连接以及与提供者的连接，请自行缓存，否则可能造成内存和连接泄漏
reference.setApplication(application);
reference.setRegistry(registry); // 多个注册中心可以用setRegistries()
reference.setInterface(XxxService.class);
reference.setVersion("1.0.0");

// 和本地bean一样使用xxxService
XxxService xxxService = reference.get(); // 注意：此代理对象内部封装了所有通讯细节，对象较重，请缓存复用
```

- 可以看到，创建了 ApplicationConfig 和 RegistryConfig 对象，设置到 ReferenceConfig 对象。
- 如果创建 ModuleConfig 或 MonitorConfig 对象，也是可以设置到 ReferenceConfig 对象中。

#### 3.1 AbstractConfig

[`com.alibaba.dubbo.config.AbstractConfig`](https://github.com/YunaiV/dubbo/blob/9d38b6f9f95798755141d6140e311e8fd51fecc1/dubbo-config/dubbo-config-api/src/main/java/com/alibaba/dubbo/config/AbstractConfig.java) ，**抽象**配置类，除了 ArgumentConfig ，我们可以看到所有的配置类都继承该类。

AbstractConfig 主要提供配置解析与校验相关的工具方法。下面我们开始看看它的代码。

[`id`](https://github.com/YunaiV/dubbo/blob/9d38b6f9f95798755141d6140e311e8fd51fecc1/dubbo-config/dubbo-config-api/src/main/java/com/alibaba/dubbo/config/AbstractConfig.java#L110) 属性，配置对象的编号，适用于除了 API 配置**之外**的三种配置方式，标记一个配置对象，可用于对象之间的引用。例如 XML 的 `<dubbo:service provider="${PROVIDER_ID}">` ，其中 `provider` 为 `<dubbo:provider>` 的 ID 属性。

那为什么说不适用 API 配置呢？直接 `#setXXX(config)` 对象即可。

------

配置项校验的工具方法，例如属性值长度限制、格式限制等等，比较简单。相关代码如下：

- [静态属性](https://github.com/YunaiV/dubbo/blob/9d38b6f9f95798755141d6140e311e8fd51fecc1/dubbo-config/dubbo-config-api/src/main/java/com/alibaba/dubbo/config/AbstractConfig.java#L49-L67)
- [静态方法](https://github.com/YunaiV/dubbo/blob/9d38b6f9f95798755141d6140e311e8fd51fecc1/dubbo-config/dubbo-config-api/src/main/java/com/alibaba/dubbo/config/AbstractConfig.java#L413-L494)

------

[`#appendParameters(parameters, config, prefix)`](https://github.com/YunaiV/dubbo/blob/9d38b6f9f95798755141d6140e311e8fd51fecc1/dubbo-config/dubbo-config-api/src/main/java/com/alibaba/dubbo/config/AbstractConfig.java#L244-L313) 方法，将配置对象的属性，添加到参数集合。代码如下 ：

> 在看具体代码之前，我们先来了解 [「4. URL」](http://svip.iocoder.cn/Dubbo/configuration-api-1/#) 和 [「5. @Parameter」](http://svip.iocoder.cn/Dubbo/configuration-api-1/#) 。

```
 1: protected static void appendParameters(Map<String, String> parameters, Object config, String prefix) {
 2:     if (config == null) {
 3:         return;
 4:     }
 5:     Method[] methods = config.getClass().getMethods();
 6:     for (Method method : methods) {
 7:         try {
 8:             String name = method.getName();
 9:             if ((name.startsWith("get") || name.startsWith("is"))
10:                     && !"getClass".equals(name)
11:                     && Modifier.isPublic(method.getModifiers())
12:                     && method.getParameterTypes().length == 0
13:                     && isPrimitive(method.getReturnType())) { // 方法为获取基本类型，public 的 getting 方法。
14:                 Parameter parameter = method.getAnnotation(Parameter.class);
15:                 if (method.getReturnType() == Object.class || parameter != null && parameter.excluded()) {
16:                     continue;
17:                 }
18:                 // 获得属性名
19:                 int i = name.startsWith("get") ? 3 : 2;
20:                 String prop = StringUtils.camelToSplitName(name.substring(i, i + 1).toLowerCase() + name.substring(i + 1), ".");
21:                 String key;
22:                 if (parameter != null && parameter.key() != null && parameter.key().length() > 0) {
23:                     key = parameter.key();
24:                 } else {
25:                     key = prop;
26:                 }
27:                 // 获得属性值
28:                 Object value = method.invoke(config, new Object[0]);
29:                 String str = String.valueOf(value).trim();
30:                 if (value != null && str.length() > 0) {
31:                     // 转义
32:                     if (parameter != null && parameter.escaped()) {
33:                         str = URL.encode(str);
34:                     }
35:                     // 拼接，详细说明参见 `Parameter#append()` 方法的说明。
36:                     if (parameter != null && parameter.append()) {
37:                         String pre = parameters.get(Constants.DEFAULT_KEY + "." + key); // default. 里获取，适用于 ServiceConfig =》ProviderConfig 、ReferenceConfig =》ConsumerConfig 。
38:                         if (pre != null && pre.length() > 0) {
39:                             str = pre + "," + str;
40:                         }
41:                         pre = parameters.get(key); // 通过 `parameters` 属性配置，例如 `AbstractMethodConfig.parameters` 。
42:                         if (pre != null && pre.length() > 0) {
43:                             str = pre + "," + str;
44:                         }
45:                     }
46:                     if (prefix != null && prefix.length() > 0) {
47:                         key = prefix + "." + key;
48:                     }
49:                     parameters.put(key, str);
50: //                    System.out.println("kv:" + key + "\t" + str);
51:                 } else if (parameter != null && parameter.required()) {
52:                     throw new IllegalStateException(config.getClass().getSimpleName() + "." + key + " == null");
53:                 }
54:             } else if ("getParameters".equals(name)
55:                     && Modifier.isPublic(method.getModifiers())
56:                     && method.getParameterTypes().length == 0
57:                     && method.getReturnType() == Map.class) { // `#getParameters()` 方法
58:                 Map<String, String> map = (Map<String, String>) method.invoke(config, new Object[0]);
59:                 if (map != null && map.size() > 0) {
60:                     String pre = (prefix != null && prefix.length() > 0 ? prefix + "." : "");
61:                     for (Map.Entry<String, String> entry : map.entrySet()) {
62:                         parameters.put(pre + entry.getKey().replace('-', '.'), entry.getValue());
63:                     }
64:                 }
65:             }
66:         } catch (Exception e) {
67:             throw new IllegalStateException(e.getMessage(), e);
68:         }
69:     }
70: }
```

- `parameters` ，参数集合。实际上，该集合会用于 `URL.parameters` 。

- `config` ，配置对象。

- `prefix` ，属性前缀。用于配置项添加到 `parameters` 中时的前缀。

- 第 5 行：获得所有方法的数组，为下面通过**反射**获得配置项的值做准备。

- 第 6 行：**循环**每个方法。

- 第 9 至 13 行：方法为获得

  基本类型

   

  +

   

  ```
  public
  ```

   

  的 getting 方法。

  - 第 14 至 17 行：返回值类型为 Object 或排除( [`@Parameter.exclue](mailto:`@Parameter.exclue)=true` )的配置项，跳过。
  - 第 19 至 26 行：获得配置项**名**。
  - 第 28 至 48 行：获得配置项**值**。中间有一些逻辑处理，胖友看下代码的注释。
  - 第 49 行：添加配置项到 `parameters` 。
  - 第 51 至 53 行：当 [`@Parameter.required](mailto:`@Parameter.required) = true` 时，校验配置项非空。

- 第 54 至 57 行：当方法为

   

  ```
  #getParameters()
  ```

   

  时，

  例如

   

  。

  - 第 58 行：通过反射，获得 `#getParameters()` 的返回值为 `map` 。
  - 第 59 至 64 行：将 `map` 添加到 `parameters` ，kv 格式为 `prefix:entry.key` `entry.value` 。
  - 因此，通过 `#getParameters()` 对应的属性，**动态设置配置项，拓展出非 Dubbo 内置好的逻辑**。

------

[`#appendAttributes(parameters, config, prefix)`](https://github.com/YunaiV/dubbo/blob/9d38b6f9f95798755141d6140e311e8fd51fecc1/dubbo-config/dubbo-config-api/src/main/java/com/alibaba/dubbo/config/AbstractConfig.java#L326-L363) 方法，将 `@Parameter(attribute = true)` 配置对象的属性，添加到参数集合。代码如下：

```
 1: protected static void appendAttributes(Map<Object, Object> parameters, Object config, String prefix) {
 2:     if (config == null) {
 3:         return;
 4:     }
 5:     Method[] methods = config.getClass().getMethods();
 6:     for (Method method : methods) {
 7:         try {
 8:             String name = method.getName();
 9:             if ((name.startsWith("get") || name.startsWith("is"))
10:                     && !"getClass".equals(name)
11:                     && Modifier.isPublic(method.getModifiers())
12:                     && method.getParameterTypes().length == 0
13:                     && isPrimitive(method.getReturnType())) { // 方法为获取基本类型，public 的 getting 方法。
14:                 Parameter parameter = method.getAnnotation(Parameter.class);
15:                 if (parameter == null || !parameter.attribute())
16:                     continue;
17:                 // 获得属性名
18:                 String key;
19:                 if (parameter != null && parameter.key() != null && parameter.key().length() > 0) {
20:                     key = parameter.key();
21:                 } else {
22:                     int i = name.startsWith("get") ? 3 : 2;
23:                     key = name.substring(i, i + 1).toLowerCase() + name.substring(i + 1);
24:                 }
25:                 // 获得属性值，存在则添加到 `parameters` 集合
26:                 Object value = method.invoke(config, new Object[0]);
27:                 if (value != null) {
28:                     if (prefix != null && prefix.length() > 0) {
29:                         key = prefix + "." + key;
30:                     }
31:                     parameters.put(key, value);
32:                 }
33:             }
34:         } catch (Exception e) {
35:             throw new IllegalStateException(e.getMessage(), e);
36:         }
37:     }
38: }
```

- 不同于 [`#appendAttributes(parameters, config, prefix)`](https://github.com/YunaiV/dubbo/blob/9d38b6f9f95798755141d6140e311e8fd51fecc1/dubbo-config/dubbo-config-api/src/main/java/com/alibaba/dubbo/config/AbstractConfig.java#L326-L363) 方法，主要用于 [《Dubbo 用户指南 —— 事件通知》](http://dubbo.apache.org/zh-cn/docs/user/demos/events-notify.html) ，注解 `@Parameter(attribute = true)` 的属性如下图：[![@Parameter(attribute = true)](assets/05.png)](http://static.iocoder.cn/images/Dubbo/2018_01_07/05.png)@Parameter(attribute = true)
- 第 9 至 13 行：方法为获得**基本类型** + `public` 的 getting 方法。
- 第 14 至 16 行：**需要**( [`@Parameter.exclue](mailto:`@Parameter.exclue)=true` )的配置项。
- 第 17 至 24 行：获得配置项**名**。
- 第 26 至 30 行：获得配置项**值**。
- 第 31 行：添加配置项到 `parameters` 。

------

[`#appendProperties(config)`](https://github.com/YunaiV/dubbo/blob/9d38b6f9f95798755141d6140e311e8fd51fecc1/dubbo-config/dubbo-config-api/src/main/java/com/alibaba/dubbo/config/AbstractConfig.java#L139-L212) 方法，读取环境变量和 properties 配置到配置对象。在 [《精进 Dubbo 源码解析 —— 属性配置》](http://svip.iocoder.cn/Dubbo/configuration-properties/?self) 详细解析。

[`#appendAnnotation(annotationClass, annotation)`](https://github.com/YunaiV/dubbo/blob/9d38b6f9f95798755141d6140e311e8fd51fecc1/dubbo-config/dubbo-config-api/src/main/java/com/alibaba/dubbo/config/AbstractConfig.java#L505-L541) 方法，读取注解配置到配置对象。在 [《精进 Dubbo 源码解析 —— 注解配置》](http://svip.iocoder.cn/Dubbo/configuration-api-1/TODO) 详细解析。

#### 3.2 ApplicationConfig

[`com.alibaba.dubbo.config.ApplicationConfig`](https://github.com/YunaiV/dubbo/blob/9d38b6f9f95798755141d6140e311e8fd51fecc1/dubbo-config/dubbo-config-api/src/main/java/com/alibaba/dubbo/config/ApplicationConfig.java) ，应用配置。

- 具体属性的解释，参见 [《Dubbo 用户指南 —— dubbo:application》](http://dubbo.apache.org/zh-cn/docs/user/references/xml/dubbo-application.html) 文档。

#### 3.3 RegistryConfig

[`com.alibaba.dubbo.config.RegistryConfig`](https://github.com/YunaiV/dubbo/blob/9d38b6f9f95798755141d6140e311e8fd51fecc1/dubbo-config/dubbo-config-api/src/main/java/com/alibaba/dubbo/config/RegistryConfig.java) ，注册中心配置。

- 具体属性的解释，参见 [《Dubbo 用户指南 —— dubbo:registry》](http://dubbo.apache.org/zh-cn/docs/user/references/xml/dubbo-registry.html) 文档。

#### 3.4 ModuleConfig

[`com.alibaba.dubbo.config.ModuleConfig`](https://github.com/YunaiV/dubbo/blob/9d38b6f9f95798755141d6140e311e8fd51fecc1/dubbo-config/dubbo-config-api/src/main/java/com/alibaba/dubbo/config/ModuleConfig.java) ，模块信息配置。

- 具体属性的解释，参见 [《Dubbo 用户指南 —— dubbo:module》](http://dubbo.apache.org/zh-cn/docs/user/references/xml/dubbo-module.html) 文档。

#### 3.5 MonitorConfig

[`com.alibaba.dubbo.config.MonitorConfig`](https://github.com/YunaiV/dubbo/blob/9d38b6f9f95798755141d6140e311e8fd51fecc1/dubbo-config/dubbo-config-api/src/main/java/com/alibaba/dubbo/config/MonitorConfig.java) ，监控中心配置。

- 具体属性的解释，参见 [《Dubbo 用户指南 —— dubbo:monitor》](http://dubbo.apache.org/zh-cn/docs/user/references/xml/dubbo-monitor.html) 文档。

#### 3.6 ArgumentConfig

[`com.alibaba.dubbo.config.ArgumentConfig`](https://github.com/YunaiV/dubbo/blob/9d38b6f9f95798755141d6140e311e8fd51fecc1/dubbo-config/dubbo-config-api/src/main/java/com/alibaba/dubbo/config/ArgumentConfig.java) ，方法参数配置。

- 具体属性的解释，参见 [《Dubbo 用户指南 —— dubbo:argument》](http://dubbo.apache.org/zh-cn/docs/user/references/xml/dubbo-argument.html) 文档。
- 该配置类设置到 MethodConfig 对象中，在 [《API 配置（二）之服务提供者》](http://svip.iocoder.cn/Dubbo/configuration-api-2/?self) 我们会看到。
- 在 [《Dubbo 用户指南 —— 参数回调》](http://dubbo.apache.org/zh-cn/docs/user/demos/callback-parameter.html) 特性中使用。

### 4. URL

[`com.alibaba.dubbo.common.URL`](https://github.com/YunaiV/dubbo/blob/9d38b6f9f95798755141d6140e311e8fd51fecc1/dubbo-common/src/main/java/com/alibaba/dubbo/common/URL.java) ，Dubbo URL 。代码如下：

```
public final class URL implements Serializable {

    /**
     * 协议名
     */
    private final String protocol;
    /**
     * 用户名
     */
    private final String username;
    /**
     * 密码
     */
    private final String password;
    /**
     * by default, host to registry
     * 地址
     */
    private final String host;
    /**
     * by default, port to registry
     * 端口
     */
    private final int port;
    /**
     * 路径（服务名）
     */
    private final String path;
    /**
     * 参数集合
     */
    private final Map<String, String> parameters;
    
    // ... 省略其他代码
    
}
```

- 上文我们提到**所有配置最终都将转换为 Dubbo URL 表示，并由服务提供方生成，经注册中心传递给消费方，各属性对应 URL 的参数，参见配置项一览表中的 “对应URL参数” 列**。那么一个 Service 注册到注册中心的格式如下：

  ```
  dubbo://192.168.3.17:20880/com.alibaba.dubbo.demo.DemoService?anyhost=true&application=demo-provider&default.delay=-1&default.retries=0&default.service.filter=demoFilter&delay=-1&dubbo=2.0.0&generic=false&interface=com.alibaba.dubbo.demo.DemoService&methods=sayHello&pid=19031&side=provider&timestamp=1519651641799
  ```

  - 格式为 `protocol://username:password@host:port/path?key=value&key=value` ，通过 [`URL#buildString(...)`](https://github.com/YunaiV/dubbo/blob/9d38b6f9f95798755141d6140e311e8fd51fecc1/dubbo-common/src/main/java/com/alibaba/dubbo/common/URL.java#L1176-L1217) 方法生成。

- `parameters` 属性，参数集合。从上面的 Service URL 例子我们可以看到，里面的 `key=value` ，实际上就是 Service 对应的配置项。该属性，通过 [`AbstractConfig#appendParameters(parameters, config, prefix)`](https://github.com/YunaiV/dubbo/blob/9d38b6f9f95798755141d6140e311e8fd51fecc1/dubbo-config/dubbo-config-api/src/main/java/com/alibaba/dubbo/config/AbstractConfig.java#L244-L313) 方法生成。

- 🙂 在后续的文章中，我们会发现 URL 作为一个**通用模型**，贯穿整个 RPC 流程。

### 5. @Parameter

[`com.alibaba.dubbo.config.support.@Parameter`](https://github.com/YunaiV/dubbo/blob/9d38b6f9f95798755141d6140e311e8fd51fecc1/dubbo-config/dubbo-config-api/src/main/java/com/alibaba/dubbo/config/support/Parameter.java) ，Parameter 参数**注解**，用于 Dubbo URL 的 `parameters` 拼接。

在配置对象的 getting 方法上，我们可以看到该注解的使用，例如下图：

[![@Parameter 使用场景](assets/04.png)](http://static.iocoder.cn/images/Dubbo/2018_01_07/04.png)@Parameter 使用场景

@Parameter 代码如下：

```java
@Documented
@Retention(RetentionPolicy.RUNTIME)
@Target({ElementType.METHOD})
public @interface Parameter {

    /**
     * 键（别名）
     */
    String key() default "";

    /**
     * 是否必填
     */
    boolean required() default false;

    /**
     * 是否忽略
     */
    boolean excluded() default false;

    /**
     * 是否转义
     */
    boolean escaped() default false;

    /**
     * 是否为属性
     *
     * 目前用于《事件通知》http://dubbo.apache.org/zh-cn/docs/user/demos/events-notify.html
     */
    boolean attribute() default false;

    /**
     * 是否拼接默认属性，参见 {@link com.alibaba.dubbo.config.AbstractConfig#appendParameters(Map, Object, String)} 方法。
     *
     * 我们来看看 `#append() = true` 的属性，有如下四个：
     *   + {@link AbstractInterfaceConfig#getFilter()}
     *   + {@link AbstractInterfaceConfig#getListener()}
     *   + {@link AbstractReferenceConfig#getFilter()}
     *   + {@link AbstractReferenceConfig#getListener()}
     *   + {@link AbstractServiceConfig#getFilter()}
     *   + {@link AbstractServiceConfig#getListener()}
     * 那么，以 AbstractServiceConfig 举例子。
     *
     * 我们知道 ProviderConfig 和 ServiceConfig 继承 AbstractServiceConfig 类，那么 `filter` , `listener` 对应的相同的键。
     * 下面我们以 `filter` 举例子。
     *
     * 在 ServiceConfig 中，默认会<b>继承</b> ProviderConfig 配置的 `filter` 和 `listener` 。
     * 所以这个属性，就是用于，像 ServiceConfig 的这种情况，从 ProviderConfig 读取父属性。
     *
     * 举个例子，如果 `ProviderConfig.filter=aaaFilter` ，`ServiceConfig.filter=bbbFilter` ，最终暴露到 Dubbo URL 时，参数为 `service.filter=aaaFilter,bbbFilter` 。
     */
    boolean append() default false;
```

# API 配置（二）之服务提供者

## 1. 概述

本文接 [《API 配置（一）之应用》](http://svip.iocoder.cn/Dubbo/configuration-api-1/) ，分享**服务提供者**相关的配置：包括 provider-config 和 sub-config 部分。

[![配置类关系](assets/03.png)](http://static.iocoder.cn/images/Dubbo/2018_01_07/03.png)配置类关系

- **黄框**部分，provider-side
- **其他**部分，sub-config

------

还是老样子，我们先来看一段 [《Dubbo 用户指南 —— API 配置》](http://dubbo.apache.org/zh-cn/docs/user/configuration/api.html) ，服务提供者的初始化代码：

```
// 服务实现
XxxService xxxService = new XxxServiceImpl();

// 当前应用配置
ApplicationConfig application = new ApplicationConfig();
application.setName("xxx");

// 连接注册中心配置
RegistryConfig registry = new RegistryConfig();
registry.setAddress("10.20.130.230:9090");
registry.setUsername("aaa");
registry.setPassword("bbb");

// 服务提供者协议配置
ProtocolConfig protocol = new ProtocolConfig();
protocol.setName("dubbo");
protocol.setPort(12345);
protocol.setThreads(200);

// 注意：ServiceConfig为重对象，内部封装了与注册中心的连接，以及开启服务端口

// 服务提供者暴露服务配置
ServiceConfig<XxxService> service = new ServiceConfig<XxxService>(); // 此实例很重，封装了与注册中心的连接，请自行缓存，否则可能造成内存和连接泄漏
service.setApplication(application);
service.setRegistry(registry); // 多个注册中心可以用setRegistries()
service.setProtocol(protocol); // 多个协议可以用setProtocols()
service.setInterface(XxxService.class);
service.setRef(xxxService);
service.setVersion("1.0.0");

// 暴露及注册服务
service.export();
```

- 相比 ReferenceConfig 的初始化，会多创建 ProtocolConfig 对象，设置到 ServiceConfig 对象中。

> 友情提示：本文前面部分会比较琐碎，重点在 [「8. ServiceConfig」](http://svip.iocoder.cn/Dubbo/configuration-api-2/#) 部分。

## 2. ProtocolConfig

[`com.alibaba.dubbo.config.ProtocolConfig`](https://github.com/YunaiV/dubbo/blob/9d38b6f9f95798755141d6140e311e8fd51fecc1/dubbo-config/dubbo-config-api/src/main/java/com/alibaba/dubbo/config/ProtocolConfig.java) ，服务提供者协议配置。

- 具体属性的解释，参见 [《Dubbo 用户指南 —— dubbo:protocol》](http://dubbo.apache.org/zh-cn/docs/user/references/xml/dubbo-protocol.html) 文档。

## 3. AbstractMethodConfig

[`com.alibaba.dubbo.config.AbstractMethodConfig`](https://github.com/YunaiV/dubbo/blob/9d38b6f9f95798755141d6140e311e8fd51fecc1/dubbo-config/dubbo-config-api/src/main/java/com/alibaba/dubbo/config/AbstractMethodConfig.java) ，方法级配置的抽象类。

- **部分**属性的解释，参见 [《Dubbo 用户指南 —— dubbo:method》](http://dubbo.apache.org/zh-cn/docs/user/references/xml/dubbo-method.html) 文档。

## 4. MethodConfig

[`com.alibaba.dubbo.config.MethodConfig`](https://github.com/YunaiV/dubbo/blob/9d38b6f9f95798755141d6140e311e8fd51fecc1/dubbo-config/dubbo-config-api/src/main/java/com/alibaba/dubbo/config/AbstractMethodConfig.java) ，继承 AbstractMethodConfig ，方法级配置。

- 具体属性的解释，[《Dubbo 用户指南 —— dubbo:method》](http://dubbo.apache.org/zh-cn/docs/user/references/xml/dubbo-method.html) 文档。

## 5. AbstractInterfaceConfig

[`com.alibaba.dubbo.config.AbstractInterfaceConfig`](https://github.com/YunaiV/dubbo/blob/9d38b6f9f95798755141d6140e311e8fd51fecc1/dubbo-config/dubbo-config-api/src/main/java/com/alibaba/dubbo/config/AbstractInterfaceConfig.java) ，继承 AbstractMethodConfig ，抽象接口配置类。

- 具体属性的解释，**需要寻找**在 [《Dubbo 用户指南 —— dubbo:service》](http://dubbo.apache.org/zh-cn/docs/user/references/xml/dubbo-service.html) 或 [《Dubbo 用户指南 —— dubbo:reference》](http://dubbo.apache.org/zh-cn/docs/user/references/xml/dubbo-reference.html) 文档。

- 下面的方法，会在 [「8. ServiceConfig」](http://svip.iocoder.cn/Dubbo/configuration-api-2/#) 的初始化被调用，胖友可需要的时候，点击查看。

- `#checkApplication()`

   

  方法，校验 ApplicationConfig 配置。

  实际上

  ，该方法会初始化 ApplicationConfig 的配置属性。

  - 🙂 直接点击方法查看，较为简单，已经添加详细注释。

- `#checkRegistry()`

   

  方法，校验 RegistryConfig 配置。

  实际上

  ，该方法会初始化 RegistryConfig 的配置属性。

  - 🙂 直接点击方法查看，较为简单，已经添加详细注释。

- `#checkInterfaceAndMethods(interfaceClass, methods)`

   

  方法，校验接口和方法。主要是两方面：

  - 1、 接口类非空，并是接口
  - 2、 方法在接口中已定义
  - 🙂 直接点击方法查看，较为简单，已经添加详细注释。

- [`#checkStubAndMock(interfaceClass)`](https://github.com/YunaiV/dubbo/blob/9d38b6f9f95798755141d6140e311e8fd51fecc1/dubbo-config/dubbo-config-api/src/main/java/com/alibaba/dubbo/config/AbstractInterfaceConfig.java#L319-L368) 方法，校验 Stub 和 Mock 相关的配置。

- 以上未列举的 [`#loadRegistries(provider)`](https://github.com/YunaiV/dubbo/blob/9d38b6f9f95798755141d6140e311e8fd51fecc1/dubbo-config/dubbo-config-api/src/main/java/com/alibaba/dubbo/config/AbstractInterfaceConfig.java#L189-L233) 和 [`#loadMonitor(registryURL)`](https://github.com/YunaiV/dubbo/blob/9d38b6f9f95798755141d6140e311e8fd51fecc1/dubbo-config/dubbo-config-api/src/main/java/com/alibaba/dubbo/config/AbstractInterfaceConfig.java#L235-L278) 方法，在**后续文章**需要使用到时，在详细分享。

## 6. AbstractServiceConfig

[`com.alibaba.dubbo.config.AbstractServiceConfig`](https://github.com/YunaiV/dubbo/blob/9d38b6f9f95798755141d6140e311e8fd51fecc1/dubbo-config/dubbo-config-api/src/main/java/com/alibaba/dubbo/config/AbstractServiceConfig.java) ，实现 AbstractInterfaceConfig ，抽象服务配置类。

- 具体属性的解释，**需要寻找**在 [《Dubbo 用户指南 —— dubbo:service》](http://dubbo.apache.org/zh-cn/docs/user/references/xml/dubbo-service.html) 或 [《Dubbo 用户指南 —— dubbo:provider》](http://dubbo.apache.org/zh-cn/docs/user/references/xml/dubbo-provider.html) 文档。

## 7. ProviderConfig

[`com.alibaba.dubbo.config.ProviderConfig`](https://github.com/YunaiV/dubbo/blob/9d38b6f9f95798755141d6140e311e8fd51fecc1/dubbo-config/dubbo-config-api/src/main/java/com/alibaba/dubbo/config/ProviderConfig.java) ，实现 AbstractServiceConfig ，服务提供者缺省值配置。

- 具体属性的解释，参见 [《Dubbo 用户指南 —— dubbo:provider》](http://dubbo.apache.org/zh-cn/docs/user/references/xml/dubbo-provider.html) 文档。

## 8. ServiceConfig

[`com.alibaba.dubbo.config.ServiceConfig`](https://github.com/YunaiV/dubbo/blob/9d38b6f9f95798755141d6140e311e8fd51fecc1/dubbo-config/dubbo-config-api/src/main/java/com/alibaba/dubbo/config/ServiceConfig.java) ，服务提供者暴露**服务配置类**。

- 具体属性的解释，参见 [《Dubbo 用户指南 —— dubbo:service》](http://dubbo.apache.org/zh-cn/docs/user/references/xml/dubbo-service.html) 文档。

下面，我们进入**正戏**。

在文初的 ServiceConfig 的初始化示例代码中，最后调用的是 [`ServiceConfig#export()`](https://github.com/YunaiV/dubbo/blob/9d38b6f9f95798755141d6140e311e8fd51fecc1/dubbo-config/dubbo-config-api/src/main/java/com/alibaba/dubbo/config/ServiceConfig.java#L239) 方法。从方法的命名，我们可以看出，**暴露服务**。该方法主要做了如下几件事情：

1. **进一步初始化** ServiceConfig 对象。
2. **校验** ServiceConfig 对象的配置项。
3. 使用 ServiceConfig 对象，**生成** Dubbo URL 对象**数组**。
4. 使用 Dubbo URL 对象，**暴露服务**。

😈 本文重点在服务提供者相关的配置，因此只解析 **1+2+3** 部分( 不包括 4 )。代码如下：

```
 1: public synchronized void export() {
 2:     // 当 export 或者 delay 未配置，从 ProviderConfig 对象读取。
 3:     if (provider != null) {
 4:         if (export == null) {
 5:             export = provider.getExport();
 6:         }
 7:         if (delay == null) {
 8:             delay = provider.getDelay();
 9:         }
10:     }
11:     // 不暴露服务( export = false ) ，则不进行暴露服务逻辑。
12:     if (export != null && !export) {
13:         return;
14:     }
15: 
16:     // 延迟暴露
17:     if (delay != null && delay > 0) {
18:         delayExportExecutor.schedule(new Runnable() {
19:             public void run() {
20:                 doExport();
21:             }
22:         }, delay, TimeUnit.MILLISECONDS);
23:     // 立即暴露
24:     } else {
25:         doExport();
26:     }
27: }
```

- 第 2 至 10 行：当 [`export`](https://github.com/YunaiV/dubbo/blob/9d38b6f9f95798755141d6140e311e8fd51fecc1/dubbo-config/dubbo-config-api/src/main/java/com/alibaba/dubbo/config/AbstractServiceConfig.java#L48) 或 [`delay`](https://github.com/YunaiV/dubbo/blob/9d38b6f9f95798755141d6140e311e8fd51fecc1/dubbo-config/dubbo-config-api/src/main/java/com/alibaba/dubbo/config/AbstractServiceConfig.java#L45) 未配置时，从 ProviderConfig 对象读取。

- 第 11 至 14 行：当配置不需要暴露服务( `export = false` )时，直接返回。

- 第 17 至 22 行：当配置延迟暴露(

   

  ```
  delay > 0
  ```

   

  )时，使用

   

  `delayExportExecutor`

   

  延迟

  调度，调用

   

  ```
  #doExport()
  ```

   

  方法。

  - [《Dubbo 用户指南 —— 延迟暴露》](http://dubbo.apache.org/zh-cn/docs/user/demos/delay-publish.html)

- 第 23 至 26 行：立即暴露，调用 `#doExport()` 方法。

------

[`#doExport()`](https://github.com/YunaiV/dubbo/blob/9d38b6f9f95798755141d6140e311e8fd51fecc1/dubbo-config/dubbo-config-api/src/main/java/com/alibaba/dubbo/config/ServiceConfig.java#L267-L391) 方法，代码如下：

```
  1: protected synchronized void doExport() {
  2:     // 检查是否可以暴露，若可以，标记已经暴露。
  3:     if (unexported) {
  4:         throw new IllegalStateException("Already unexported!");
  5:     }
  6:     if (exported) {
  7:         return;
  8:     }
  9:     exported = true;
 10:     // 校验接口名非空
 11:     if (interfaceName == null || interfaceName.length() == 0) {
 12:         throw new IllegalStateException("<dubbo:service interface=\"\" /> interface not allow null!");
 13:     }
 14:     // 拼接属性配置（环境变量 + properties 属性）到 ProviderConfig 对象
 15:     checkDefault();
 16:     // 从 ProviderConfig 对象中，读取 application、module、registries、monitor、protocols 配置对象。
 17:     if (provider != null) {
 18:         if (application == null) {
 19:             application = provider.getApplication();
 20:         }
 21:         if (module == null) {
 22:             module = provider.getModule();
 23:         }
 24:         if (registries == null) {
 25:             registries = provider.getRegistries();
 26:         }
 27:         if (monitor == null) {
 28:             monitor = provider.getMonitor();
 29:         }
 30:         if (protocols == null) {
 31:             protocols = provider.getProtocols();
 32:         }
 33:     }
 34:     // 从 ModuleConfig 对象中，读取 registries、monitor 配置对象。
 35:     if (module != null) {
 36:         if (registries == null) {
 37:             registries = module.getRegistries();
 38:         }
 39:         if (monitor == null) {
 40:             monitor = module.getMonitor();
 41:         }
 42:     }
 43:     // 从 ApplicationConfig 对象中，读取 registries、monitor 配置对象。
 44:     if (application != null) {
 45:         if (registries == null) {
 46:             registries = application.getRegistries();
 47:         }
 48:         if (monitor == null) {
 49:             monitor = application.getMonitor();
 50:         }
 51:     }
 52:     // 泛化接口的实现
 53:     if (ref instanceof GenericService) {
 54:         interfaceClass = GenericService.class;
 55:         if (StringUtils.isEmpty(generic)) {
 56:             generic = Boolean.TRUE.toString();
 57:         }
 58:     // 普通接口的实现
 59:     } else {
 60:         try {
 61:             interfaceClass = Class.forName(interfaceName, true, Thread.currentThread().getContextClassLoader());
 62:         } catch (ClassNotFoundException e) {
 63:             throw new IllegalStateException(e.getMessage(), e);
 64:         }
 65:         // 校验接口和方法
 66:         checkInterfaceAndMethods(interfaceClass, methods);
 67:         // 校验指向的 service 对象
 68:         checkRef();
 69:         generic = Boolean.FALSE.toString();
 70:     }
 71:     // 处理服务接口客户端本地代理( `local` )相关。实际目前已经废弃，使用 `stub` 属性，参见 `AbstractInterfaceConfig#setLocal` 方法。
 72:     if (local != null) {
 73:         // 设为 true，表示使用缺省代理类名，即：接口名 + Local 后缀
 74:         if ("true".equals(local)) {
 75:             local = interfaceName + "Local";
 76:         }
 77:         Class<?> localClass;
 78:         try {
 79:             localClass = ClassHelper.forNameWithThreadContextClassLoader(local);
 80:         } catch (ClassNotFoundException e) {
 81:             throw new IllegalStateException(e.getMessage(), e);
 82:         }
 83:         if (!interfaceClass.isAssignableFrom(localClass)) {
 84:             throw new IllegalStateException("The local implementation class " + localClass.getName() + " not implement interface " + interfaceName);
 85:         }
 86:     }
 87:     // 处理服务接口客户端本地代理( `stub` )相关
 88:     if (stub != null) {
 89:         // 设为 true，表示使用缺省代理类名，即：接口名 + Stub 后缀
 90:         if ("true".equals(stub)) {
 91:             stub = interfaceName + "Stub";
 92:         }
 93:         Class<?> stubClass;
 94:         try {
 95:             stubClass = ClassHelper.forNameWithThreadContextClassLoader(stub);
 96:         } catch (ClassNotFoundException e) {
 97:             throw new IllegalStateException(e.getMessage(), e);
 98:         }
 99:         if (!interfaceClass.isAssignableFrom(stubClass)) {
100:             throw new IllegalStateException("The stub implementation class " + stubClass.getName() + " not implement interface " + interfaceName);
101:         }
102:     }
103:     // 校验 ApplicationConfig 配置。
104:     checkApplication();
105:     // 校验 RegistryConfig 配置。
106:     checkRegistry();
107:     // 校验 ProtocolConfig 配置数组。
108:     checkProtocol();
109:     // 读取环境变量和 properties 配置到 ServiceConfig 对象。
110:     appendProperties(this);
111:     // 校验 Stub 和 Mock 相关的配置
112:     checkStubAndMock(interfaceClass);
113:     // 服务路径，缺省为接口名
114:     if (path == null || path.length() == 0) {
115:         path = interfaceName;
116:     }
117:     // 暴露服务
118:     doExportUrls();
119:     // TODO 芋艿，等待 qos
120:     ProviderModel providerModel = new ProviderModel(getUniqueServiceName(), this, ref);
121:     ApplicationModel.initProviderModel(getUniqueServiceName(), providerModel);
122: }
```

- 第 2 至 9 行：检查是否可以暴露。若可以，标记已经暴露( `exported = true` )。

- 第 10 至 13 行：校验接口名 [`interfaceName`](https://github.com/YunaiV/dubbo/blob/9d38b6f9f95798755141d6140e311e8fd51fecc1/dubbo-config/dubbo-config-api/src/main/java/com/alibaba/dubbo/config/ServiceConfig.java#L111) 非空。

- 第 15 行：调用

   

  `#checkDefault()`

   

  方法，读取

  属性配置

  ( 环境变量 + properties 属性 )到 ProviderConfig 对象。

  - 关于“**属性配置**” ，在 [《精尽 Dubbo 源码解析 —— 属性配置》](http://svip.iocoder.cn/Dubbo/configuration-properties/?self) 详细解析。
  - 🙂 直接点击方法查看，较为简单，已经添加详细注释。

- 第 16 至 33 行：从 ProviderConfig 对象中，读取 [`application`](https://github.com/YunaiV/dubbo/blob/9d38b6f9f95798755141d6140e311e8fd51fecc1/dubbo-config/dubbo-config-api/src/main/java/com/alibaba/dubbo/config/AbstractInterfaceConfig.java#L102)、[`module`](https://github.com/YunaiV/dubbo/blob/9d38b6f9f95798755141d6140e311e8fd51fecc1/dubbo-config/dubbo-config-api/src/main/java/com/alibaba/dubbo/config/AbstractInterfaceConfig.java#L105)、[`registries`](https://github.com/YunaiV/dubbo/blob/9d38b6f9f95798755141d6140e311e8fd51fecc1/dubbo-config/dubbo-config-api/src/main/java/com/alibaba/dubbo/config/AbstractInterfaceConfig.java#L108)、[`monitor`](https://github.com/YunaiV/dubbo/blob/9d38b6f9f95798755141d6140e311e8fd51fecc1/dubbo-config/dubbo-config-api/src/main/java/com/alibaba/dubbo/config/AbstractInterfaceConfig.java#L77)、[`protocols`](https://github.com/YunaiV/dubbo/blob/9d38b6f9f95798755141d6140e311e8fd51fecc1/dubbo-config/dubbo-config-api/src/main/java/com/alibaba/dubbo/config/AbstractServiceConfig.java#L64) 对象。

- 第 34 至 42 行：从 ModuleConfig 对象中，读取 [`registries`](https://github.com/YunaiV/dubbo/blob/9d38b6f9f95798755141d6140e311e8fd51fecc1/dubbo-config/dubbo-config-api/src/main/java/com/alibaba/dubbo/config/AbstractInterfaceConfig.java#L108)、[`monitor`](https://github.com/YunaiV/dubbo/blob/9d38b6f9f95798755141d6140e311e8fd51fecc1/dubbo-config/dubbo-config-api/src/main/java/com/alibaba/dubbo/config/AbstractInterfaceConfig.java#L77) 对象。

- 第 43 至 51 行：从 ApplicationConfig 对象中，读取 [`registries`](https://github.com/YunaiV/dubbo/blob/9d38b6f9f95798755141d6140e311e8fd51fecc1/dubbo-config/dubbo-config-api/src/main/java/com/alibaba/dubbo/config/AbstractInterfaceConfig.java#L108)、[`monitor`](https://github.com/YunaiV/dubbo/blob/9d38b6f9f95798755141d6140e311e8fd51fecc1/dubbo-config/dubbo-config-api/src/main/java/com/alibaba/dubbo/config/AbstractInterfaceConfig.java#L77) 对象。

- 第 52 至 57 行：泛化接口的实现。

  - [《Dubbo 用户指南 —— 泛化接口》](http://dubbo.apache.org/zh-cn/docs/user/demos/generic-service.html)

- 第 58 至 70 行：普通接口的实现。

  - 第 60 至 64 行：根据 [`interfaceName`](https://github.com/YunaiV/dubbo/blob/9d38b6f9f95798755141d6140e311e8fd51fecc1/dubbo-config/dubbo-config-api/src/main/java/com/alibaba/dubbo/config/ServiceConfig.java#L105) ，获得对应的**接口类**，并赋值给 [`interfaceClass`](https://github.com/YunaiV/dubbo/blob/9d38b6f9f95798755141d6140e311e8fd51fecc1/dubbo-config/dubbo-config-api/src/main/java/com/alibaba/dubbo/config/ServiceConfig.java#L111)。

  - 第 66 行：调用

     

    ```
    #checkInterfaceAndMethods(interfaceClass, methods)
    ```

     

    方法，检查接口和方法。

    - 🙂 本文有已经有这个方法的解析。

  - 第 68 行：调用 [`#checkRef()`](https://github.com/YunaiV/dubbo/blob/9d38b6f9f95798755141d6140e311e8fd51fecc1/dubbo-config/dubbo-config-api/src/main/java/com/alibaba/dubbo/config/ServiceConfig.java#L393-L408) 方法，校验指向的 Service 对象。

  - 第 69 行：标记 [`generic`](https://github.com/YunaiV/dubbo/blob/9d38b6f9f95798755141d6140e311e8fd51fecc1/dubbo-config/dubbo-config-api/src/main/java/com/alibaba/dubbo/config/ServiceConfig.java#L138) 为**非**泛化实现。

- 第 71 至 86 行：处理服务接口客户端本地代理( [`local`](https://github.com/YunaiV/dubbo/blob/9d38b6f9f95798755141d6140e311e8fd51fecc1/dubbo-config/dubbo-config-api/src/main/java/com/alibaba/dubbo/config/AbstractInterfaceConfig.java#L55-L63) )相关。**实际目前已经废弃，此处主要用于兼容**，使用 [`stub`](https://github.com/YunaiV/dubbo/blob/9d38b6f9f95798755141d6140e311e8fd51fecc1/dubbo-config/dubbo-config-api/src/main/java/com/alibaba/dubbo/config/AbstractInterfaceConfig.java#L64-L74) 属性，参见 [`AbstractInterfaceConfig#setLocal(local)`](https://github.com/YunaiV/dubbo/blob/9d38b6f9f95798755141d6140e311e8fd51fecc1/dubbo-config/dubbo-config-api/src/main/java/com/alibaba/dubbo/config/AbstractInterfaceConfig.java#L379-L390) 方法的**注释说明**。

- 第 87 至 102 行：处理服务接口客户端本地代理( [`stub`](https://github.com/YunaiV/dubbo/blob/9d38b6f9f95798755141d6140e311e8fd51fecc1/dubbo-config/dubbo-config-api/src/main/java/com/alibaba/dubbo/config/AbstractInterfaceConfig.java#L64-L74) 属性，参见 [`AbstractInterfaceConfig#setLocal(local)`](https://github.com/YunaiV/dubbo/blob/9d38b6f9f95798755141d6140e311e8fd51fecc1/dubbo-config/dubbo-config-api/src/main/java/com/alibaba/dubbo/config/AbstractInterfaceConfig.java#L379-L390) )相关。

- 第 104 行：调用

   

  `#checkApplication()`

   

  方法，校验 ApplicationConfig 配置。

  - 🙂 直接点击方法查看，较为简单，已经添加详细注释。

- 第 106 行：调用

   

  `#checkRegistry()`

   

  方法，校验 RegistryConfig 配置。

  - 🙂 直接点击方法查看，较为简单，已经添加详细注释。

- 第 108 行：调用

   

  `#checkProtocol()`

   

  方法，校验 ProtocolConfig 配置数组。

  - 🙂 直接点击方法查看，较为简单，已经添加详细注释。

- 第 110 行：调用 [`#appendProperties(config)`](https://github.com/YunaiV/dubbo/blob/9d38b6f9f95798755141d6140e311e8fd51fecc1/dubbo-config/dubbo-config-api/src/main/java/com/alibaba/dubbo/config/AbstractConfig.java#L132-L212) 方法，读取**属性配置**( 环境变量 + properties 属性 )到 ServiceConfig 对象（**自己**）。

- 第 112 行：调用 [`#checkStubAndMock(interfaceClass)`](https://github.com/YunaiV/dubbo/blob/9d38b6f9f95798755141d6140e311e8fd51fecc1/dubbo-config/dubbo-config-api/src/main/java/com/alibaba/dubbo/config/AbstractInterfaceConfig.java#L319-L368) 方法，校验 Stub 和 Mock 相关的配置。

- 第 113 至 116 行：服务路径 [`path`](https://github.com/YunaiV/dubbo/blob/9d38b6f9f95798755141d6140e311e8fd51fecc1/dubbo-config/dubbo-config-api/src/main/java/com/alibaba/dubbo/config/ServiceConfig.java#L115) 为空时，缺省为接口名。

- 第 118 行：调用 [`#doExportUrls()`](https://github.com/YunaiV/dubbo/blob/9d38b6f9f95798755141d6140e311e8fd51fecc1/dubbo-config/dubbo-config-api/src/main/java/com/alibaba/dubbo/config/ServiceConfig.java#L430-L439) 方法，**暴露服务**。此方法包含了我们上述的 **3+4** 部分。

- 第 119 至 121 行：// TODO 芋艿，等待 qos

------

因为本文不分享 **4** 部分，所以下面我们只看 [`#doExportUrls()`](https://github.com/YunaiV/dubbo/blob/9d38b6f9f95798755141d6140e311e8fd51fecc1/dubbo-config/dubbo-config-api/src/main/java/com/alibaba/dubbo/config/ServiceConfig.java#L430-L439) 方法中，调用 [`#doExportUrlsFor1Protocol(protocolConfig, registryURLs)`](https://github.com/YunaiV/dubbo/blob/9d38b6f9f95798755141d6140e311e8fd51fecc1/dubbo-config/dubbo-config-api/src/main/java/com/alibaba/dubbo/config/ServiceConfig.java#L441-L621) 方法，和 **3** 有关的部分。代码如下：

```
  1: private void doExportUrlsFor1Protocol(ProtocolConfig protocolConfig, List<URL> registryURLs) {
  2:     // 协议名
  3:     String name = protocolConfig.getName();
  4:     if (name == null || name.length() == 0) {
  5:         name = "dubbo";
  6:     }
  7: 
  8:     // 将 `side`，`dubbo`，`timestamp`，`pid` 参数，添加到 `map` 集合中。
  9:     Map<String, String> map = new HashMap<String, String>();
 10:     map.put(Constants.SIDE_KEY, Constants.PROVIDER_SIDE);
 11:     map.put(Constants.DUBBO_VERSION_KEY, Version.getVersion());
 12:     map.put(Constants.TIMESTAMP_KEY, String.valueOf(System.currentTimeMillis()));
 13:     if (ConfigUtils.getPid() > 0) {
 14:         map.put(Constants.PID_KEY, String.valueOf(ConfigUtils.getPid()));
 15:     }
 16:     // 将各种配置对象，添加到 `map` 集合中。
 17:     appendParameters(map, application);
 18:     appendParameters(map, module);
 19:     appendParameters(map, provider, Constants.DEFAULT_KEY); // ProviderConfig ，为 ServiceConfig 的默认属性，因此添加 `default` 属性前缀。
 20:     appendParameters(map, protocolConfig);
 21:     appendParameters(map, this);
 22:     // 将 MethodConfig 对象数组，添加到 `map` 集合中。
 23:     if (methods != null && !methods.isEmpty()) {
 24:         for (MethodConfig method : methods) {
 25:             // 将 MethodConfig 对象，添加到 `map` 集合中。
 26:             appendParameters(map, method, method.getName());
 27:             // 当 配置了 `MethodConfig.retry = false` 时，强制禁用重试
 28:             String retryKey = method.getName() + ".retry";
 29:             if (map.containsKey(retryKey)) {
 30:                 String retryValue = map.remove(retryKey);
 31:                 if ("false".equals(retryValue)) {
 32:                     map.put(method.getName() + ".retries", "0");
 33:                 }
 34:             }
 35:             // 将 ArgumentConfig 对象数组，添加到 `map` 集合中。
 36:             List<ArgumentConfig> arguments = method.getArguments();
 37:             if (arguments != null && !arguments.isEmpty()) {
 38:                 for (ArgumentConfig argument : arguments) {
 39:                     // convert argument type
 40:                     if (argument.getType() != null && argument.getType().length() > 0) { // 指定了类型
 41:                         Method[] methods = interfaceClass.getMethods();
 42:                         // visit all methods
 43:                         if (methods != null && methods.length > 0) {
 44:                             for (int i = 0; i < methods.length; i++) {
 45:                                 String methodName = methods[i].getName();
 46:                                 // target the method, and get its signature
 47:                                 if (methodName.equals(method.getName())) { // 找到指定方法
 48:                                     Class<?>[] argTypes = methods[i].getParameterTypes();
 49:                                     // one callback in the method
 50:                                     if (argument.getIndex() != -1) { // 指定单个参数的位置 + 类型
 51:                                         if (argTypes[argument.getIndex()].getName().equals(argument.getType())) {
 52:                                             // 将 ArgumentConfig 对象，添加到 `map` 集合中。
 53:                                             appendParameters(map, argument, method.getName() + "." + argument.getIndex()); // `${methodName}.${index}`
 54:                                         } else {
 55:                                             throw new IllegalArgumentException("argument config error : the index attribute and type attribute not match :index :" + argument.getIndex() + ", type:" + argument.getType());
 56:                                         }
 57:                                     } else {
 58:                                         // multiple callbacks in the method
 59:                                         for (int j = 0; j < argTypes.length; j++) {
 60:                                             Class<?> argClazz = argTypes[j];
 61:                                             if (argClazz.getName().equals(argument.getType())) {
 62:                                                 // 将 ArgumentConfig 对象，添加到 `map` 集合中。
 63:                                                 appendParameters(map, argument, method.getName() + "." + j); // `${methodName}.${index}`
 64:                                                 if (argument.getIndex() != -1 && argument.getIndex() != j) { // 多余的判断，因为 `argument.getIndex() == -1` 。
 65:                                                     throw new IllegalArgumentException("argument config error : the index attribute and type attribute not match :index :" + argument.getIndex() + ", type:" + argument.getType());
 66:                                                 }
 67:                                             }
 68:                                         }
 69:                                     }
 70:                                 }
 71:                             }
 72:                         }
 73:                     } else if (argument.getIndex() != -1) { // 指定单个参数的位置
 74:                         // 将 ArgumentConfig 对象，添加到 `map` 集合中。
 75:                         appendParameters(map, argument, method.getName() + "." + argument.getIndex()); // `${methodName}.${index}`
 76:                     } else {
 77:                         throw new IllegalArgumentException("argument config must set index or type attribute.eg: <dubbo:argument index='0' .../> or <dubbo:argument type=xxx .../>");
 78:                     }
 79: 
 80:                 }
 81:             }
 82:         } // end of methods for
 83:     }
 84: 
 85:     // generic、methods、revision
 86:     if (ProtocolUtils.isGeneric(generic)) {
 87:         map.put("generic", generic);
 88:         map.put("methods", Constants.ANY_VALUE);
 89:     } else {
 90:         String revision = Version.getVersion(interfaceClass, version);
 91:         if (revision != null && revision.length() > 0) {
 92:             map.put("revision", revision); // 修订号
 93:         }
 94: 
 95:         String[] methods = Wrapper.getWrapper(interfaceClass).getMethodNames(); // 获得方法数组
 96:         if (methods.length == 0) {
 97:             logger.warn("NO method found in service interface " + interfaceClass.getName());
 98:             map.put("methods", Constants.ANY_VALUE);
 99:         } else {
100:             map.put("methods", StringUtils.join(new HashSet<String>(Arrays.asList(methods)), ","));
101:         }
102:     }
103:     // token ，参见《令牌校验》http://dubbo.apache.org/zh-cn/docs/user/demos/token-authorization.html
104:     if (!ConfigUtils.isEmpty(token)) {
105:         if (ConfigUtils.isDefault(token)) {
106:             map.put("token", UUID.randomUUID().toString());
107:         } else {
108:             map.put("token", token);
109:         }
110:     }
111:     // 协议为 injvm 时，不注册，不通知。
112:     if ("injvm".equals(protocolConfig.getName())) {
113:         protocolConfig.setRegister(false);
114:         map.put("notify", "false");
115:     }
116:     // export service
117:     String contextPath = protocolConfig.getContextpath();
118:     if ((contextPath == null || contextPath.length() == 0) && provider != null) {
119:         contextPath = provider.getContextpath();
120:     }
121: 
122:     // host、port
123:     String host = this.findConfigedHosts(protocolConfig, registryURLs, map);
124:     Integer port = this.findConfigedPorts(protocolConfig, name, map);
125: 
126:     // 创建 Dubbo URL 对象
127:     URL url = new URL(name, host, port, (contextPath == null || contextPath.length() == 0 ? "" : contextPath + "/") + path, map);
128: 
129:     // 配置规则，参见《配置规则》http://dubbo.apache.org/zh-cn/docs/user/demos/config-rule.html
130:     if (ExtensionLoader.getExtensionLoader(ConfiguratorFactory.class)
131:             .hasExtension(url.getProtocol())) {
132:         url = ExtensionLoader.getExtensionLoader(ConfiguratorFactory.class)
133:                 .getExtension(url.getProtocol()).getConfigurator(url).configure(url);
134:     }
135: 
136:     // 省略【服务暴露】逻辑
137: }
```

- 第 2 至 6 行：协议名空时，缺省 `"dubbo"` 。

- 第 9 行：创建参数集合 `map` ，用于下面创建 Dubbo URL 的 [`parameters`](https://github.com/YunaiV/dubbo/blob/9d38b6f9f95798755141d6140e311e8fd51fecc1/dubbo-common/src/main/java/com/alibaba/dubbo/common/URL.java#L109) 属性。

- 第 10 至 15 行：将 `side` `dubbo` `timestamp` `timestamp` `pid` 添加到 `map` 中。

- 第 16 至 21 行：调用

   

  ```
  #appendParameters(map, config)
  ```

   

  方法，将各种配置对象添加到

   

  ```
  map
  ```

   

  中。

  - 🙂 `#appendParameters(map, config)` 方法，在 [《API 配置（一）之应用》](http://svip.iocoder.cn/Dubbo/configuration-api-1/?self)

- 第 22 至 83 行：调用 MethodConfig 对象

  数组

  ，添加到

   

  ```
  map
  ```

   

  中。

  - 目的是将**每个** MethodConfig 和其对应的 ArgumentConfig 对象数组，添加到 `map` 中。
  - 🙂 代码比较冗长，胖友耐心看注释，建议进行调试每种情况。

- 第 85 至 102 行：将

   

  ```
  generic
  ```

   

  ```
  methods
  ```

   

  ```
  revision
  ```

   

  到

   

  ```
  map
  ```

   

  中。

  - `revision` ，可能比较难理解，在 [「10. Version」](http://svip.iocoder.cn/Dubbo/configuration-api-2/#) 详细解析。

- 第 103 至 110 行：将

   

  ```
  token
  ```

   

  添加到

   

  ```
  map
  ```

   

  中。

  - [《Dubbo 用户指南 —— 令牌验证》](http://dubbo.apache.org/zh-cn/docs/user/demos/token-authorization.html)

- 第 111 至 115 行：当协议为

   

  ```
  injvm
  ```

   

  时，添加

   

  ```
  notify = false
  ```

   

  到

   

  ```
  map
  ```

   

  中，表示不注册，不通知。

  - [《Dubbo 用户指南 —— 本地调用》](http://dubbo.apache.org/zh-cn/docs/user/demos/local-call.html)

- 第 116 至 120 行：获得 `contextPath` ，基础路径，即java web应用中常说的context path 。

- 第 123 行：调用

   

  `#this.findConfigedHosts(protocolConfig, registryURLs, map)`

   

  方法，获得注册到注册中心的服务提供者 Host 。

  - [《Dubbo 用户指南 —— 主机绑定》](http://dubbo.apache.org/zh-cn/docs/user/demos/hostname-binding.html)
  - [《dubbo注册服务IP解析异常及IP解析源码分析》](https://segmentfault.com/a/1190000010550512)
  - 指定服务注册地址，参见 [dubbo-docker-sample](https://github.com/dubbo/dubbo-docker-sample) 示例项目。
  - 🙂 代码比较冗长，胖友耐心看注释，建议进行调试每种情况。

- 第 124 行：调用

   

  `#findConfigedHosts(protocolConfig, name, map)`

   

  方法，获得注册到注册中心的服务提供者 Port 。

  - 🙂 代码比较冗长，胖友耐心看注释，建议进行调试每种情况。

- 第 127 行：创建 Dubbo URL 。

- 第 129 至 134 行：配置规则，后续详细解析。

  - [《Dubbo 用户指南 —— 配置规则》](http://dubbo.apache.org/zh-cn/docs/user/demos/config-rule.html)

- 第 136 行：**省略**【服务暴露】逻辑。

## 9. 为什么继承？？？

我们以 ServiceConfig 和 ProviderConfig 来举例子，两者都继承 AbstractServiceConfig。
从属性上，两者有相同的属性，例如 `group` / `version` 。
同时，也存在着一些差异，例如 `ServiceConfig.interfaceName` / `ProviderConfig.host` 。

另外，我们在看看 ServiceConfig 和 MethodConfig ，两者都继承 AbstractMethodConfig。
在 ServiceConfig 中，可以配置下属所有方法的 `retries` 次数，也可以在 MethodConfig 中**自定义** `retries` 次数。

通过继承，获得相同的属性。

## 10. Version

[`Version#getVersion(cls, defaultVersion)`](https://github.com/alibaba/dubbo/blob/0423219d839404186b8a5ec7dec37f6addeb58d9/dubbo-common/src/main/java/com/alibaba/dubbo/common/Version.java) 方法，获得版本号。代码如下：

```
 1: public static String getVersion(Class<?> cls, String defaultVersion) {
 2:     try {
 3:         // find version info from MANIFEST.MF first
 4:         String version = cls.getPackage().getImplementationVersion();
 5:         if (version == null || version.length() == 0) {
 6:             version = cls.getPackage().getSpecificationVersion();
 7:         }
 8:         if (version == null || version.length() == 0) {
 9:             // guess version fro jar file name if nothing's found from MANIFEST.MF
10:             CodeSource codeSource = cls.getProtectionDomain().getCodeSource();
11:             if (codeSource == null) {
12:                 logger.info("No codeSource for class " + cls.getName() + " when getVersion, use default version " + defaultVersion);
13:             } else {
14:                 String file = codeSource.getLocation().getFile();
15:                 if (file != null && file.length() > 0 && file.endsWith(".jar")) {
16:                     file = file.substring(0, file.length() - 4);
17:                     int i = file.lastIndexOf('/');
18:                     if (i >= 0) {
19:                         file = file.substring(i + 1);
20:                     }
21:                     i = file.indexOf("-");
22:                     if (i >= 0) {
23:                         file = file.substring(i + 1);
24:                     }
25:                     while (file.length() > 0 && !Character.isDigit(file.charAt(0))) {
26:                         i = file.indexOf("-");
27:                         if (i >= 0) {
28:                             file = file.substring(i + 1);
29:                         } else {
30:                             break;
31:                         }
32:                     }
33:                     version = file;
34:                 }
35:             }
36:         }
37:         // return default version if no version info is found
38:         return version == null || version.length() == 0 ? defaultVersion : version;
39:     } catch (Throwable e) {
40:         // return default version when any exception is thrown
41:         logger.error("return default version, ignore exception " + e.getMessage(), e);
42:         return defaultVersion;
43:     }
44: }
```

- 第 3 至 7 行：从 `MAINFEST.MF` 中获得版本号。以 [spring-boot-starter-1.5.10.RELEASE.jar](http://central.maven.org/maven2/org/springframework/boot/spring-boot-starter/1.5.10.RELEASE/spring-boot-starter-1.5.10.RELEASE.jar) 举例子：[![MAINFEST.MF](assets/06)](http://static.iocoder.cn/images/Dubbo/2018_01_10/01.png)MAINFEST.MF
- 第 8 至 36 行：若获取不到，从 jar 包**命名**中**可能**带的版本号作为结果。例如上面的例子，`1.5.10.RELEASE` 。
- 第 38 行：返回版本号。若不存在，返回默认版本号。

# API 配置（三）之服务消费者

## 1. 概述

本文接 [《API 配置（二）之服务提供者》](http://svip.iocoder.cn/Dubbo/configuration-api-2/) ，分享**服务消费者**相关的配置。

[![配置类关系](assets/03.png)](http://static.iocoder.cn/images/Dubbo/2018_01_07/03.png)配置类关系

- **红框**部分，consumer-side

------

还是老样子，我们先来看一段 [《Dubbo 用户指南 —— API 配置》](http://dubbo.apache.org/zh-cn/docs/user/configuration/api.html) ，服务消费者的初始化代码：

```
// 当前应用配置
ApplicationConfig application = new ApplicationConfig();
application.setName("yyy");

// 连接注册中心配置
RegistryConfig registry = new RegistryConfig();
registry.setAddress("10.20.130.230:9090");
registry.setUsername("aaa");
registry.setPassword("bbb");

// 注意：ReferenceConfig为重对象，内部封装了与注册中心的连接，以及与服务提供方的连接

// 引用远程服务
ReferenceConfig<XxxService> reference = new ReferenceConfig<XxxService>(); // 此实例很重，封装了与注册中心的连接以及与提供者的连接，请自行缓存，否则可能造成内存和连接泄漏
reference.setApplication(application);
reference.setRegistry(registry); // 多个注册中心可以用setRegistries()
reference.setInterface(XxxService.class);
reference.setVersion("1.0.0");

// 和本地bean一样使用xxxService
XxxService xxxService = reference.get(); // 注意：此代理对象内部封装了所有通讯细节，对象较重，请缓存复用
```

## 2. AbstractReferenceConfig

[`com.alibaba.dubbo.config.AbstractReferenceConfig`](https://github.com/YunaiV/dubbo/blob/9d38b6f9f95798755141d6140e311e8fd51fecc1/dubbo-config/dubbo-config-api/src/main/java/com/alibaba/dubbo/config/AbstractReferenceConfig.java) ，实现 AbstractInterfaceConfig ，抽象引用配置类。

- 具体属性的解释，**需要寻找**在 [《Dubbo 用户指南 —— dubbo:reference》](http://dubbo.apache.org/zh-cn/docs/user/references/xml/dubbo-reference.html) 或 [《Dubbo 用户指南 —— dubbo:consumer》](http://dubbo.apache.org/zh-cn/docs/user/references/xml/dubbo-consumer.html) 文档。

## 3. ConsumerConfig

[`com.alibaba.dubbo.config.ConsumerConfig`](https://github.com/YunaiV/dubbo/blob/9d38b6f9f95798755141d6140e311e8fd51fecc1/dubbo-config/dubbo-config-api/src/main/java/com/alibaba/dubbo/config/ConsumerConfig.java) ，实现 AbstractReferenceConfig ，服务消费者缺省值配置。

- 具体属性的解释，参见 [《Dubbo 用户指南 —— dubbo:consumer》](http://dubbo.apache.org/zh-cn/docs/user/references/xml/dubbo-consumer.html) 文档。

## 4. ReferenceConfig

[`com.alibaba.dubbo.config.ReferenceConfig`](https://github.com/YunaiV/dubbo/blob/9d38b6f9f95798755141d6140e311e8fd51fecc1/dubbo-config/dubbo-config-api/src/main/java/com/alibaba/dubbo/config/ReferenceConfig.java) ，服务消费者引用**服务配置类**。

- 具体属性的解释，参见 [《Dubbo 用户指南 —— dubbo:consumer》](http://dubbo.apache.org/zh-cn/docs/user/references/xml/dubbo-consumer.html) 文档。

下面，我们进入**正戏**。

在文初的 ReferenceConfig 的初始化示例代码中，最后调用的是 [`ServiceConfig#get()`](https://github.com/YunaiV/dubbo/blob/d3c3975f320c78452f96098b04441fed4c00ab70/dubbo-config/dubbo-config-api/src/main/java/com/alibaba/dubbo/config/ReferenceConfig.java#L185-L195) 方法。从方法的命名，我们可以看出，获取**引用服务**。该方法主要做了如下几件事情：

1. **进一步初始化** ReferenceConfig 对象。
2. **校验** ReferenceConfig 对象的配置项。
3. 使用 ReferenceConfig 对象，**生成** Dubbo URL 对象**数组**。
4. 使用 Dubbo URL 对象，**应用服务**。

😈 本文重点在服务提供者相关的配置，因此只解析 **1+2+3** 部分( 不包括 4 )。代码如下：

```
 1: public synchronized T get() {
 2:     // 已销毁，不可获得
 3:     if (destroyed) {
 4:         throw new IllegalStateException("Already destroyed!");
 5:     }
 6:     // 初始化
 7:     if (ref == null) {
 8:         init();
 9:     }
10:     return ref;
11: }
```

- 第 2 至 5 行：若已经销毁( `destroyed = true` )，抛出异常。
- 第 7 至 9 行：若未初始化，调用 `#init()` 方法，进行初始化。
- 第 10 行：返回引用服务。

------

[`#init()`](http://svip.iocoder.cn/Dubbo/configuration-api-3/) 方法，代码如下：

> 友情提示，该方法并未拆分更多的小方法，所以超级长，近 200+ 行。

```
  1: private void init() {
  2:     // 已经初始化，直接返回
  3:     if (initialized) {
  4:         return;
  5:     }
  6:     initialized = true;
  7:     // 校验接口名非空
  8:     if (interfaceName == null || interfaceName.length() == 0) {
  9:         throw new IllegalStateException("<dubbo:reference interface=\"\" /> interface not allow null!");
 10:     }
 11:     // 拼接属性配置（环境变量 + properties 属性）到 ConsumerConfig 对象
 12:     // get consumer's global configuration
 13:     checkDefault();
 14:     // 拼接属性配置（环境变量 + properties 属性）到 ReferenceConfig 对象
 15:     appendProperties(this);
 16:     // 若未设置 `generic` 属性，使用 `ConsumerConfig.generic` 属性。
 17:     if (getGeneric() == null && getConsumer() != null) {
 18:         setGeneric(getConsumer().getGeneric());
 19:     }
 20:     // 泛化接口的实现
 21:     if (ProtocolUtils.isGeneric(getGeneric())) {
 22:         interfaceClass = GenericService.class;
 23:     // 普通接口的实现
 24:     } else {
 25:         try {
 26:             interfaceClass = Class.forName(interfaceName, true, Thread.currentThread().getContextClassLoader());
 27:         } catch (ClassNotFoundException e) {
 28:             throw new IllegalStateException(e.getMessage(), e);
 29:         }
 30:         // 校验接口和方法
 31:         checkInterfaceAndMethods(interfaceClass, methods);
 32:     }
 33:     // 直连提供者，参见文档《直连提供者》http://dubbo.apache.org/zh-cn/docs/user/demos/explicit-target.html
 34:     // 【直连提供者】第一优先级，通过 -D 参数指定 ，例如 java -Dcom.alibaba.xxx.XxxService=dubbo://localhost:20890
 35:     String resolve = System.getProperty(interfaceName);
 36:     String resolveFile = null;
 37:     // 【直连提供者】第二优先级，通过文件映射，例如 com.alibaba.xxx.XxxService=dubbo://localhost:20890
 38:     if (resolve == null || resolve.length() == 0) {
 39:         // 默认先加载，`${user.home}/dubbo-resolve.properties` 文件 ，无需配置
 40:         resolveFile = System.getProperty("dubbo.resolve.file");
 41:         if (resolveFile == null || resolveFile.length() == 0) {
 42:             File userResolveFile = new File(new File(System.getProperty("user.home")), "dubbo-resolve.properties");
 43:             if (userResolveFile.exists()) {
 44:                 resolveFile = userResolveFile.getAbsolutePath();
 45:             }
 46:         }
 47:         // 存在 resolveFile ，则进行文件读取加载。
 48:         if (resolveFile != null && resolveFile.length() > 0) {
 49:             Properties properties = new Properties();
 50:             FileInputStream fis = null;
 51:             try {
 52:                 fis = new FileInputStream(new File(resolveFile));
 53:                 properties.load(fis);
 54:             } catch (IOException e) {
 55:                 throw new IllegalStateException("Unload " + resolveFile + ", cause: " + e.getMessage(), e);
 56:             } finally {
 57:                 try {
 58:                     if (null != fis) fis.close();
 59:                 } catch (IOException e) {
 60:                     logger.warn(e.getMessage(), e);
 61:                 }
 62:             }
 63:             resolve = properties.getProperty(interfaceName);
 64:         }
 65:     }
 66:     // 设置直连提供者的 url
 67:     if (resolve != null && resolve.length() > 0) {
 68:         url = resolve;
 69:         if (logger.isWarnEnabled()) {
 70:             if (resolveFile != null && resolveFile.length() > 0) {
 71:                 logger.warn("Using default dubbo resolve file " + resolveFile + " replace " + interfaceName + "" + resolve + " to p2p invoke remote service.");
 72:             } else {
 73:                 logger.warn("Using -D" + interfaceName + "=" + resolve + " to p2p invoke remote service.");
 74:             }
 75:         }
 76:     }
 77:     // 从 ConsumerConfig 对象中，读取 application、module、registries、monitor 配置对象。
 78:     if (consumer != null) {
 79:         if (application == null) {
 80:             application = consumer.getApplication();
 81:         }
 82:         if (module == null) {
 83:             module = consumer.getModule();
 84:         }
 85:         if (registries == null) {
 86:             registries = consumer.getRegistries();
 87:         }
 88:         if (monitor == null) {
 89:             monitor = consumer.getMonitor();
 90:         }
 91:     }
 92:     // 从 ModuleConfig 对象中，读取 registries、monitor 配置对象。
 93:     if (module != null) {
 94:         if (registries == null) {
 95:             registries = module.getRegistries();
 96:         }
 97:         if (monitor == null) {
 98:             monitor = module.getMonitor();
 99:         }
100:     }
101:     // 从 ApplicationConfig 对象中，读取 registries、monitor 配置对象。
102:     if (application != null) {
103:         if (registries == null) {
104:             registries = application.getRegistries();
105:         }
106:         if (monitor == null) {
107:             monitor = application.getMonitor();
108:         }
109:     }
110:     // 校验 ApplicationConfig 配置。
111:     checkApplication();
112:     // 校验 Stub 和 Mock 相关的配置
113:     checkStubAndMock(interfaceClass);
114:     // 将 `side`，`dubbo`，`timestamp`，`pid` 参数，添加到 `map` 集合中。
115:     Map<String, String> map = new HashMap<String, String>();
116:     Map<Object, Object> attributes = new HashMap<Object, Object>();
117:     map.put(Constants.SIDE_KEY, Constants.CONSUMER_SIDE);
118:     map.put(Constants.DUBBO_VERSION_KEY, Version.getVersion());
119:     map.put(Constants.TIMESTAMP_KEY, String.valueOf(System.currentTimeMillis()));
120:     if (ConfigUtils.getPid() > 0) {
121:         map.put(Constants.PID_KEY, String.valueOf(ConfigUtils.getPid()));
122:     }
123:     // methods、revision、interface
124:     if (!isGeneric()) {
125:         String revision = Version.getVersion(interfaceClass, version);
126:         if (revision != null && revision.length() > 0) {
127:             map.put("revision", revision);
128:         }
129: 
130:         String[] methods = Wrapper.getWrapper(interfaceClass).getMethodNames(); // 获得方法数组
131:         if (methods.length == 0) {
132:             logger.warn("NO method found in service interface " + interfaceClass.getName());
133:             map.put("methods", Constants.ANY_VALUE);
134:         } else {
135:             map.put("methods", StringUtils.join(new HashSet<String>(Arrays.asList(methods)), ","));
136:         }
137:     }
138:     map.put(Constants.INTERFACE_KEY, interfaceName);
139:     // 将各种配置对象，添加到 `map` 集合中。
140:     appendParameters(map, application);
141:     appendParameters(map, module);
142:     appendParameters(map, consumer, Constants.DEFAULT_KEY);
143:     appendParameters(map, this);
144:     // 获得服务键，作为前缀
145:     String prefix = StringUtils.getServiceKey(map);
146:     // 将 MethodConfig 对象数组，添加到 `map` 集合中。
147:     if (methods != null && !methods.isEmpty()) {
148:         for (MethodConfig method : methods) {
149:             // 将 MethodConfig 对象，添加到 `map` 集合中。
150:             appendParameters(map, method, method.getName());
151:             // 当 配置了 `MethodConfig.retry = false` 时，强制禁用重试
152:             String retryKey = method.getName() + ".retry";
153:             if (map.containsKey(retryKey)) {
154:                 String retryValue = map.remove(retryKey);
155:                 if ("false".equals(retryValue)) {
156:                     map.put(method.getName() + ".retries", "0");
157:                 }
158:             }
159:             // 将带有 @Parameter(attribute = true) 配置对象的属性，添加到参数集合。参见《事件通知》http://dubbo.apache.org/zh-cn/docs/user/demos/events-notify.html
160:             appendAttributes(attributes, method, prefix + "." + method.getName());
161:             // 检查属性集合中的事件通知方法是否正确。若正确，进行转换。
162:             checkAndConvertImplicitConfig(method, map, attributes);
163:         }
164:     }
165: 
166:     // 以系统环境变量( DUBBO_IP_TO_REGISTRY ) 作为服务注册地址，参见 https://github.com/dubbo/dubbo-docker-sample 项目。
167:     String hostToRegistry = ConfigUtils.getSystemProperty(Constants.DUBBO_IP_TO_REGISTRY);
168:     if (hostToRegistry == null || hostToRegistry.length() == 0) {
169:         hostToRegistry = NetUtils.getLocalHost();
170:     } else if (isInvalidLocalHost(hostToRegistry)) {
171:         throw new IllegalArgumentException("Specified invalid registry ip from property:" + Constants.DUBBO_IP_TO_REGISTRY + ", value:" + hostToRegistry);
172:     }
173:     map.put(Constants.REGISTER_IP_KEY, hostToRegistry);
174: 
175:     // 添加到 StaticContext 进行缓存
176:     //attributes are stored by system context.
177:     StaticContext.getSystemContext().putAll(attributes);
178: 
179:     // 省略【引用服务】
185: }
```

- 第 2 至 6 行：若已经初始化( `initialized = true` ) 时，直接返回。否则，标记已经初始化。

- 第 7 至 10 行：校验接口名 [`interfaceName`](https://github.com/YunaiV/dubbo/blob/d3c3975f320c78452f96098b04441fed4c00ab70/dubbo-config/dubbo-config-api/src/main/java/com/alibaba/dubbo/config/ReferenceConfig.java#L78) 非空。

- 第 13 行：调用

   

  `#checkDefault()`

   

  方法，读取

  属性配置

  ( 环境变量 + properties 属性 )到 ConsumerConfig 对象。

  - 关于“**属性配置**” ，在 [《精尽 Dubbo 源码解析 —— 属性配置》](http://svip.iocoder.cn/Dubbo/configuration-properties/?self?self) 详细解析。
  - 🙂 直接点击方法查看，较为简单，已经添加详细注释。

- 第 15 行：调用 [`#appendProperties(config)`](https://github.com/YunaiV/dubbo/blob/9d38b6f9f95798755141d6140e311e8fd51fecc1/dubbo-config/dubbo-config-api/src/main/java/com/alibaba/dubbo/config/AbstractConfig.java#L132-L212) 方法，读取**属性配置**( 环境变量 + properties 属性 )到 ReferenceConfig 对象（**自己**）

- 第 16 至 19 行：若未设置 [`generic`](https://github.com/YunaiV/dubbo/blob/d3c3975f320c78452f96098b04441fed4c00ab70/dubbo-config/dubbo-config-api/src/main/java/com/alibaba/dubbo/config/AbstractReferenceConfig.java#L43) 属性，使用 [`ConsumerConfig.generic`](https://github.com/YunaiV/dubbo/blob/d3c3975f320c78452f96098b04441fed4c00ab70/dubbo-config/dubbo-config-api/src/main/java/com/alibaba/dubbo/config/AbstractReferenceConfig.java#L43) 属性。

- 第 20 至 22 行：泛化接口的实现。

  - [《Dubbo 用户指南 —— 泛化引用》](http://dubbo.apache.org/zh-cn/docs/user/demos/generic-reference.html)

- 第 23 至 32 行：普通接口的实现。

  - 第 60 至 64 行：根据 [`interfaceName`](https://github.com/YunaiV/dubbo/blob/d3c3975f320c78452f96098b04441fed4c00ab70/dubbo-config/dubbo-config-api/src/main/java/com/alibaba/dubbo/config/ReferenceConfig.java#L78) ，获得对应的**接口类**，并赋值给 [`interfaceClass`](https://github.com/YunaiV/dubbo/blob/d3c3975f320c78452f96098b04441fed4c00ab70/dubbo-config/dubbo-config-api/src/main/java/com/alibaba/dubbo/config/ReferenceConfig.java#L79)。

  - 第 31 行：调用

     

    `#checkInterfaceAndMethods(interfaceClass, methods)`

     

    方法，检查接口和方法。

    - 🙂 直接点击方法查看，较为简单，已经添加详细注释。

- 第 33 至 76 行：直连提供者。

  - [《Dubbo 用户指南 —— 直连提供者》](http://dubbo.apache.org/zh-cn/docs/user/demos/explicit-target.html)
  - 🙂 中间有一些逻辑处理，胖友看下代码的注释。结合文档。

- 第 77 至 109 行：从 ConsumerConfig、ModuleConfig、ApplicationConfig 配置对象，复制 `application` `module` `registries` `monitor` 给 ReferenceConfig ( **自己** )。

- 第 111 行：调用

   

  `#checkApplication()`

   

  方法，校验 ApplicationConfig 配置。

  - 🙂 直接点击方法查看，较为简单，已经添加详细注释。

- 第 113 行：调用 [`#checkStubAndMock(interfaceClass)`](https://github.com/YunaiV/dubbo/blob/9d38b6f9f95798755141d6140e311e8fd51fecc1/dubbo-config/dubbo-config-api/src/main/java/com/alibaba/dubbo/config/AbstractInterfaceConfig.java#L319-L368) 方法，校验 Stub 和 Mock 相关的配置。

- **第 115 行：创建参数集合 `map` ，用于下面创建 Dubbo URL 的 [`parameters`](https://github.com/YunaiV/dubbo/blob/9d38b6f9f95798755141d6140e311e8fd51fecc1/dubbo-common/src/main/java/com/alibaba/dubbo/common/URL.java#L109) 属性**。

- 第 116 至 122 行：将 `side` `dubbo` `timestamp` `timestamp` `pid` 添加到 `map` 中。

- 第 123 至 137 行：将 `interface` `methods` `revision` 到 `map` 中。

- 第 139 至 143 行：调用

   

  ```
  #appendParameters(map, config)
  ```

   

  方法，将各种配置对象添加到

   

  ```
  map
  ```

   

  中。

  - 🙂 `#appendParameters(map, config)` 方法，在 [《API 配置（一）之应用》](http://svip.iocoder.cn/Dubbo/configuration-api-1/?self) 有详细解析。

- 第 146 至 164 行：调用 MethodConfig 对象

  数组

  ，添加到

   

  ```
  map
  ```

   

  中。

  - 目的是将**每个** MethodConfig 和其对应的 ArgumentConfig 对象数组，添加到 `map` 中。

  - 第 160 行：调用

     

    `#appendAttributes(parameters, config, prefix)`

     

    方法，将

     

    ```
    @Parameter(attribute = true)
    ```

     

    配置对象的属性，添加到参数集合。在

     

    《API 配置（一）之应用》

     

    有详细解析。

    - [《Dubbo 用户指南 —— 事件通知》](http://dubbo.apache.org/zh-cn/docs/user/demos/events-notify.html)

  - 第 162 行：调用

     

    `#checkAndConvertImplicitConfig(method, map, attributes)`

     

    方法，检查属性集合中的事件通知方法是否正确。若正确，进行转换。

    - 🙂 直接点击方法查看，较为简单，已经添加详细注释。

- 第 166 至 173 行：以系统换将变量 ( DUBBO_IP_TO_REGISTRY ) 作为服务注册地址，参见 [dubbo-docker-sample](https://github.com/dubbo/dubbo-docker-sample) 项目。

- 第 177 行：添加到

  `com.alibaba.dubbo.rpc.StaticContext`

  进行缓存。

  - 目的是 [《Dubbo 用户指南 —— 事件通知》](http://dubbo.apache.org/zh-cn/docs/user/demos/events-notify.html) 。

- 第 179 行：**省略**【服务引用】逻辑。