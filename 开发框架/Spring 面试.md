[TOC]

# Spring 面试

## SpringMVC 的理解？

SpringMVC 可以理解为对 Servlet 的封装，屏蔽掉 Servlet 很多的细节，刚学 Servlet 的时候，要获取参数需要不断的 `getParameter`。现在只要在 SpringMVC 方法定义对应的 JavaBean，只要属性名与参数名一致，SpringMVC 就可以帮我们实现将参数封装到 JavaBean 上了。

## SpringMVC 流程？

总体：

1. 首先有个统一处理请求的入口
2. 随后根据请求路径找到对应的映射器
3. 找到处理请求的适配器
4. 拦截器前置处理
5. 处理请求（调用 `Controller`）
6. 视图解析器处理
7. 拦截器后置处理

细节：

1. SpringMVC 有一个核心 Servlet —— `DispatcherServlet`。它处理所有的请求，然后通过内部的 `doDispatch` 方法将请求分发给各个 `Controller`。

2. `doDispatch` 中，有 `getHandler` 方法返回一个 `HandlerExecutionChain` 对象。

   1. 本质上是通过遍历 `HandlerMapping` 找到合适的 hm，然后调用它的 `getHandler` 方法。

      `HandlerMapping` 保存了每一个 URL 及其对应的 `Controller` 的具体方法。

   2. hm 的 `getHandler` 内部又调用了一个 `getHandlerInternal` 方法，它通过 URL 获取到请求路径，然后调用 `lookupHandler` 方法。

   3. `lookupHandler` 首先在一个缓存 Handler `Map` 中查看此请求映射是否已被缓存过，缓存过直接拿出来，返回；否则会根据 URL 匹配，找出最合适的。

3. 之后，调用 `getHandlerAdapter` 方法获取 `HandlerAdapter`，`HandlerAdpater` 会对 `Handler`（也就是 `Controller`）做适配，然后返回一个 `ModelAndView`。

   寻找 ha 的过程也是一个遍历过程，一般而言，获取到的 `HandlerAdapter` 是 `RequestMappingHandlerAdapter`。

4. 如果是 `GET` 和 `HEAD` 请求，则还会检查 `Last-Modified`，判断是否修改，也就是所谓的 HTTP 协商缓存机制。

5. 获取到 `HandlerAdapter` 之后，就会真正调用 `handle` 方法，来处理请求。

   * `handle` 方法中又调用 `invokeHandlerMethod`

   * `invokeHandlerMethod` 中又调用 `invokeAndHandle`

     * `invokeAndHandle` 又调用 `invokeForRequest`

     * `invokeForRequest` 首先调用 `getMethodArgumentValues` 从前端解析参数

       `HandlerMethodArgumentResolverComposite` 是所有 `HandlerMethodArgumentResolver` 的组合，用于找到合适的参数解析器进行解析。这里依然是遍历，然后找到合适的 `Resolver`。

     * 之后又调用 `doInvoke`，`doInvoke` 是通过反射调用方法

     * `invokeAndHandle` 调用 `invokeForRequest` 之后，就会获得返回值，然后调用  `HandlerMethodReturnValueHandlerComposite` 的`handleReturnValue` 方法

   通常来说，我们一般会获取到 `@ResponseBody` 注解的参数和返回值处理器，这里内部又会调用 `HttpMessageConverter` 进行参数的读写。

6. 接下来就是 `processDispatchResult`，这里进行视图解析，调用 `render` 方法。

   * `render` 会调用 `resolveViewName` 方法
   * `resolveViewName` 中会遍历 `ViewReslover`，找到合适的视图解析器

整体流程：

![image-20220425123732105](https://fastly.jsdelivr.net/gh/Faraway002/typora/images/image-20220425123732105.png)

## Spring IoC AOP？

### IoC

IoC 即控制反转（Inversion of Control），是指**创建对象的控制权的转移**，以前创建对象的主动权和时机是由自己把控的，而现在这种权力转移到 Spring 容器中，并由容器根据配置文件去创建实例和管理各个实例之间的依赖关系，对象与对象之间松散耦合，也利于功能的复用。

依赖注入（Dependency Injection，DI），**和控制反转是同一个概念的不同角度的描述**，即应用程序在运行时依赖 IoC 容器来动态注入对象需要的外部资源。

注入的方式有三种：构造方法的注入、setter 的注入和注解注入。

常用的就是注解注入，有 `@Autowired` 以及 `@Resource`：

* `@Autowired` 是 Spring 的注解，默认 byName 注入
* `@Resource` 是 JDK 的拓展注解，默认 byType 注入

**作用**：

- 管理对象的创建和依赖关系的维护。对象的创建并不是一件简单的事，在对象关系比较复杂时，如果依赖关系需要程序员来维护的话，那是相当头疼的
- 解耦，由容器去维护具体的对象
- 托管了类的产生过程，比如我们需要在类的产生过程中做一些处理，最直接的例子就是代理，如果有容器程序可以把这部分处理交给容器，应用程序则无需去关心类是如何完成代理的

**优点**：

- IoC 或 依赖注入把应用的代码量降到最低。
- 它使应用容易测试，单元测试不再需要单例和 JNDI 查找机制。
- 最小的代价和最小的侵入性使松散耦合得以实现。
- IoC 容器支持加载服务时的饿汉式初始化和懒加载。

#### 原理？

Spring 内部有一个 `ApplicationContext`，意思是应用程序上下文，实际上就是 IoC 容器。容器的根本接口是 `BeanFactory`，Spring 把自己管理的所有对象都叫做 Bean。

#### Bean 的生命周期

![img](https://fastly.jsdelivr.net/gh/Faraway002/typora/images/abde7edc90734009864eee7e12aa986d~tplv-k3u1fbpfcp-zoom-in-crop-mark:1304:0:0:0.awebp)

1. 首先，通过 `BeanDefinitionReader` 读取指定的配置文件生成 bean 的定义信息，然后到完整的 bean 定义信息( `BeanDefinition` 对象)，注意这里只是存储 bean 的定义信息，还没有实例化 bean 对象；
2. 在 `BeanDefinition` 和 完整 `BeanDefinition` 中间通过一个后置增强器，可以对bean的定义信息进行统一修改，只需要实现 `BeanFactoryPostProcessor` 接口即可，这个后置增强器是可以有多个的，你只要在不同的类实现多个 `BeanFactoryPostProcessor` 接口就会执行多次
3. 得到完整 `BeanDefinition` 之后就可以进行创建对象了，这整个过程被称为 bean 的生命周期，也就是从实例化到销毁的过程

![image.png](https://fastly.jsdelivr.net/gh/Faraway002/typora/images/b99901f4ba6f45159c32946d3fb31536~tplv-k3u1fbpfcp-zoom-in-crop-mark:1304:0:0:0.awebp)

这场图是简化版本的，但其实，它的内部蕴含了很多东西，让我们看看细化后的流程图：

![](https://fastly.jsdelivr.net/gh/Faraway002/typora/images/72677c123f5e41b3b8498654acac8fe0~tplv-k3u1fbpfcp-zoom-in-crop-mark:1304:0:0:0.awebp)

### 循环依赖怎么解决的？

* 首先 A 对象实例化，然后对属性进行注入，发现依赖 B 对象
* B 对象此时还没创建出来，所以转头去实例化 B 对象
* B 对象实例化之后，发现需要依赖 A 对象，那 A 对象已经实例化了嘛，所以 B 对象最终能完成创建
* B 对象返回到 A 对象的属性注入的方法上，A 对象最终完成创建

三级缓存：

三个Map，singletonObjects（一级，日常实际获取Bean的地方），earlySingletonObjects（二级，还没进行属性注入，由三级缓存放进来），singletonFactories（三级，value 是一个对象工厂）

![image-20220425162422576](https://fastly.jsdelivr.net/gh/Faraway002/typora/images/image-20220425162422576.png)

* A 对象实例化之后，属性注入之前，其实会把 A 对象放入三级缓存中
* 等到 A 对象属性注入时，发现依赖 B，又去实例化 B 时，B 属性注入需要去获取 A 对象，这里就是从三级缓存里拿出 ObjectFactory，从ObjectFactor y得到对应的 Bean（就是对象 A），然后把 A 从三级缓存放到二级缓存中
* 等到完全初始化之后，就会把二级缓存给 remove 掉，塞到一级缓存中
* 我们自己去 getBean 的时候，实际上拿到的是一级缓存的

## SpringBoot 自动配置原理？

### 加载自动配置类

主要是 Spring Boot 的启动类上的核心注解 `@SpringBootApplication` 注解主配置类，它可以被视作：

* `@EnableAutoConfiguration` 启用 SpringBoot 的自动配置机制。
* `@ComponentScan`：扫描被 `@Component`、`@Service`、`@Controller`、`@Repository` 注解的类。
* `@Configuration`：允许上下文注册额外的 bean 或导入其他配置类。

重点是 `@EnableAutoConfiguration`：

```java
@Target({ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
@AutoConfigurationPackage
@Import({AutoConfigurationImportSelector.class})
public @interface EnableAutoConfiguration {
    String ENABLED_OVERRIDE_PROPERTY = "spring.boot.enableautoconfiguration";

    Class<?>[] exclude() default {};

    String[] excludeName() default {};
}
```

这里通过 `@Import` 注解导入了 `AutoConfigurationImportSelector` 类，该类本质上是一个 `ImportSelector`，它有一个方法 `selectImports`，返回一个 `String[]`，Spring 会把这个数组里的所有类名加载到容器中，使得 Spring 可以动态从其他地方加载 Bean。

这个实现类中的 `selectImports` 会去调用 `getAutoConfigurationEntry`，而这个方法又调用 `getCandidateConfigurations` 方法，这个方法会把所有自动配置类的信息以 `List` 的形式返回：

```java
protected List<String> getCandidateConfigurations(AnnotationMetadata metadata, AnnotationAttributes attributes) {
    List<String> configurations = SpringFactoriesLoader.loadFactoryNames(getSpringFactoriesLoaderFactoryClass(),
                                                                         getBeanClassLoader());
    Assert.notEmpty(configurations, "No auto configuration classes found in META-INF/spring.factories. If you are using a custom packaging, make sure that file is correct.");
    return configurations;
}
```

所以，一切都是从这个方法开始的，那么它又是从哪里加载的呢？我们看一下 `loadFactoryNames()`：

```java
public static List<String> loadFactoryNames(Class<?> factoryType, @Nullable ClassLoader classLoader) {
    ClassLoader classLoaderToUse = classLoader;
    if (classLoaderToUse == null) {
        classLoaderToUse = SpringFactoriesLoader.class.getClassLoader();
    }
    String factoryTypeName = factoryType.getName();
    return loadSpringFactories(classLoaderToUse).getOrDefault(factoryTypeName, Collections.emptyList());
}
```

这个类好像没有做什么实际的事，实际上是调用了 `loadSpringFactories`，继续深入：

```java
private static Map<String, List<String>> loadSpringFactories(ClassLoader classLoader) {
    // 缓存机制
    Map<String, List<String>> result = cache.get(classLoader);
    if (result != null) {
        return result;
    }

    result = new HashMap<>();
    try {
        // 重点
        Enumeration<URL> urls = classLoader.getResources(FACTORIES_RESOURCE_LOCATION);
        while (urls.hasMoreElements()) {
            URL url = urls.nextElement();
            UrlResource resource = new UrlResource(url);
            Properties properties = PropertiesLoaderUtils.loadProperties(resource);
            for (Map.Entry<?, ?> entry : properties.entrySet()) {
                String factoryTypeName = ((String) entry.getKey()).trim();
                String[] factoryImplementationNames =
                    StringUtils.commaDelimitedListToStringArray((String) entry.getValue());
                for (String factoryImplementationName : factoryImplementationNames) {
                    result.computeIfAbsent(factoryTypeName, key -> new ArrayList<>())
                        .add(factoryImplementationName.trim());
                }
            }
        }
        // Replace all lists with unmodifiable lists containing unique elements
        result.replaceAll((factoryType, implementations) -> implementations.stream().distinct()
                          .collect(Collectors.collectingAndThen(Collectors.toList(), Collections::unmodifiableList)));
        cache.put(classLoader, result);
    } catch (IOException ex) {
        throw new IllegalArgumentException("Unable to load factories from location [" +
                                           FACTORIES_RESOURCE_LOCATION + "]", ex);
    }
    return result;
}
```

我们发现，一切都是从 `classLoader.getResources(FACTORIES_RESOURCE_LOCATION)` 这里开始的，这个路径是什么？我们看一下：

![image-20220720082849281](https://cdn.jsdelivr.net/gh/Faraway002/typora/images/image-20220720082849281.png)

原来如此，一切都在这个目录下，随便找一个和 SpringBoot 整合的包，我们看一下：

![image-20220720083121338](https://cdn.jsdelivr.net/gh/Faraway002/typora/images/image-20220720083121338.png)

就是这个文件，看一下里面的内容：

![image-20220720083146402](https://cdn.jsdelivr.net/gh/Faraway002/typora/images/image-20220720083146402.png)

是类名，我们点一个进去看看：

![image-20220720083208916](https://cdn.jsdelivr.net/gh/Faraway002/typora/images/image-20220720083208916.png)

这就是 MyBatis 的自动配置类，这下就串起来了。

### 条件装配

有了上面讲述的这一机制能够加载自动配置类，那么是否就万事大吉了呢？其实不是，SpringBoot 的自动装配不可能把所有的配置类都加载进去，原因有两个：

1. 依赖包可能不全。比如你想使用 MyBatis，但是你忘记创建一个数据源了，这时候把 MyBatis 进行自动装配的话会导致出现问题。
2. 环境。SpringBoot 针对不同的开发环境可以制定不同的配置文件，如果环境不同，可能导致使用到的包也不同。比如开发环境下你可能只想用内存数据库 H2，但是生产环境下你想换成 MySQL，这时就完全没有必要加载 H2 相关的配置。

因此，SpringBoot 推出了条件装配机制，利用条件装配注解，当一个自动配置类的条件装配注解不能全部满足时，就不装配。

比如上面的 MyBatis 自动配置类，注意这一行 `@ConditionalOnClass({ SqlSessionFactory.class, SqlSessionFactoryBean.class })`，它要求你的类路径里有 `SqlSessionFactory` 以及 `SqlSessionFactoryBean`，也就是 MyBatis 基本包和 MyBatis-Spring 的整合包，如果你没有这两个类，说明你都没导入 MyBatis，更别提自动装配了。

## SpringBoot 自定义 Starter