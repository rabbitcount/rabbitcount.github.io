---
title: Spring Boot 启动函数
date: 2019-01-11 15:16:12
tags:
- spring boot
---
Spring Boot 初始化函数概述
> 基于 VERSION = 2.1.1.RELEASE 
<!-- more --> 
--- 

# SpringApplication 实例化

```java
public static void main(String[] args) {
  SpringApplication.run(DemoApplication.class, args);
}
```

调用 `SpringApplication` 的重载 run 方法（args参数类型不同）；

```java
public static ConfigurableApplicationContext run(Class<?> primarySource,
    String... args) {
  return run(new Class<?>[] { primarySource }, args);
}

public static ConfigurableApplicationContext run(Class<?>[] primarySources,
    String[] args) {
  return new SpringApplication(primarySources).run(args);
}
```

并创建 `SpringApplication` 对象；

```java
public SpringApplication(ResourceLoader resourceLoader, Class<?>... primarySources) {
  this.resourceLoader = resourceLoader;
  Assert.notNull(primarySources, "PrimarySources must not be null");
  this.primarySources = new LinkedHashSet<>(Arrays.asList(primarySources));
  this.webApplicationType = WebApplicationType.deduceFromClasspath();
  setInitializers((Collection) getSpringFactoriesInstances(
    ApplicationContextInitializer.class));
  setListeners((Collection) getSpringFactoriesInstances(ApplicationListener.class));
  this.mainApplicationClass = deduceMainApplicationClass();
}
```

{% note info %} 公有依赖方法 `getSpringFactoriesInstances`：根据指定的类型查询到全路径类名称并初始化；{% endnote %}

`SpringApplication` 构造函数执行以下四个步骤：

## 推断应用类型
`WebApplicationType.deduceFromClasspath`
判断应用类型是 `REACTIVE SERVLET NONE` 中的哪一种
```java
private static final String WEBFLUX_INDICATOR_CLASS = "org."
    + "springframework.web.reactive.DispatcherHandler";
private static final String WEBMVC_INDICATOR_CLASS = "org.springframework."
    + "web.servlet.DispatcherServlet";
private static final String JERSEY_INDICATOR_CLASS =    
    "org.glassfish.jersey.servlet.ServletContainer";
private static final String[] SERVLET_INDICATOR_CLASSES = { "javax.servlet.Servlet",
    "org.springframework.web.context.ConfigurableWebApplicationContext" };
            
static WebApplicationType deduceFromClasspath() {
  // 如果仅存在 reactive.DispatcherHandler，则认定为REACTIVE的项目
  if (ClassUtils.isPresent(WEBFLUX_INDICATOR_CLASS, null)
      && !ClassUtils.isPresent(WEBMVC_INDICATOR_CLASS, null)
      && !ClassUtils.isPresent(JERSEY_INDICATOR_CLASS, null)) {
    return WebApplicationType.REACTIVE;
  }
  // 如果不存在 Servlet 和 ConfigurableWebApplicationContext 类，则认定不是 SERVLET 工程
  for (String className : SERVLET_INDICATOR_CLASSES) {
    if (!ClassUtils.isPresent(className, null)) {
      return WebApplicationType.NONE;
    }
  }
  return WebApplicationType.SERVLET;
}

public enum WebApplicationType {
  /**
   * The application should not run as a web application and should not start an
   * embedded web server.
   */
  NONE,

  /**
   * The application should run as a servlet-based web application and should start an
   * embedded servlet web server.
   */
  SERVLET,

  /**
   * The application should run as a reactive web application and should start an
   * embedded reactive web server.
   */
  REACTIVE;
  ...
}
```
## 设置初始化器(Initializer)
```java
setInitializers((Collection) getSpringFactoriesInstances(
				ApplicationContextInitializer.class));
```
[通过getSpringFactoriesInstances，查询并实例化key为org.springframework.context.ApplicationContextInitializer的对象](/2019/01/11/spring-boot-run/#getSpringFactoriesInstances)
> Initializer类的功能：
在Spring上下文被刷新之前进行初始化的操作。典型地比如在Web应用中，注册Property Sources或者是激活Profiles。Property Sources比较好理解，就是配置文件。Profiles是Spring为了在不同环境下(如DEV，TEST，PRODUCTION等)，加载不同的配置项而抽象出来的一个实体。

![](https://wiki-1258407249.cos.ap-chengdu.myqcloud.com/2019-01-11-spring-boot-run/spring-boot-run-initializer.png)
![](https://wiki-1258407249.cos.ap-chengdu.myqcloud.com/2019-01-11-spring-boot-run/spring-boot-run-initializer-boot.png)

## 设置监听器(Listener)
```java
setListeners((Collection) getSpringFactoriesInstances(ApplicationListener.class));
```

[通过getSpringFactoriesInstances，查询并实例化key为org.springframework.context.ApplicationListener的对象](/2019/01/11/spring-boot-run/#getSpringFactoriesInstances)

## 推断应用入口类
```java
this.mainApplicationClass = deduceMainApplicationClass();
```
利用了jvm的异常机制，抛出一个异常，并在异常栈中，查询有`main`方法的类；
```java
private Class<?> deduceMainApplicationClass() {
  try {
    StackTraceElement[] stackTrace = new RuntimeException().getStackTrace();
    for (StackTraceElement stackTraceElement : stackTrace) {
      if ("main".equals(stackTraceElement.getMethodName())) {
        return Class.forName(stackTraceElement.getClassName());
      }
    }
  }
  catch (ClassNotFoundException ex) {
    // Swallow and continue
  }
  return null;
}
```

# SpringApplication.run
```java
public ConfigurableApplicationContext run(String... args) {
  StopWatch stopWatch = new StopWatch();
  stopWatch.start();
  ConfigurableApplicationContext context = null;
  Collection<SpringBootExceptionReporter> exceptionReporters = new ArrayList<>();
  configureHeadlessProperty();
  // 获取SpringApplicationRunListeners
  SpringApplicationRunListeners listeners = getRunListeners(args);
  // 发出开始执行的事件
  listeners.starting();
  try {
    ApplicationArguments applicationArguments = new DefaultApplicationArguments(
        args);
    // 根据SpringApplicationRunListeners以及参数来准备环境
    ConfigurableEnvironment environment = prepareEnvironment(listeners,
        applicationArguments);
    configureIgnoreBeanInfo(environment);
    // 准备Banner打印器 - 就是启动Spring Boot的时候打印在console上的ASCII艺术字体
    Banner printedBanner = printBanner(environment);
    // 创建Spring上下文
    context = createApplicationContext();
    // 准备异常报告器
    exceptionReporters = getSpringFactoriesInstances(
        SpringBootExceptionReporter.class,
        new Class[] { ConfigurableApplicationContext.class }, context);
    // Spring上下文前置处理
    prepareContext(context, environment, listeners, applicationArguments,
        printedBanner);
    // Spring上下文刷新
    refreshContext(context);
    // Spring上下文后置处理
    afterRefresh(context, applicationArguments);
    // 发出结束执行的事件
    stopWatch.stop();
    if (this.logStartupInfo) {
      new StartupInfoLogger(this.mainApplicationClass)
          .logStarted(getApplicationLog(), stopWatch);
    }
    listeners.started(context);
    callRunners(context, applicationArguments);
  }
  catch (Throwable ex) {
    handleRunFailure(context, ex, exceptionReporters, listeners);
    throw new IllegalStateException(ex);
  }

  try {
    listeners.running(context);
  }
  catch (Throwable ex) {
    handleRunFailure(context, ex, exceptionReporters, null);
    throw new IllegalStateException(ex);
  }
  return context;
}
```

## 获取SpringApplicationRunListeners

[通过getSpringFactoriesInstances，查询并实例化key为org.springframework.boot.SpringApplicationRunListener的对象](/2019/01/11/spring-boot-run/#getSpringFactoriesInstances)

`EventPublishingRunListener` 负责发布SpringApplicationEvent事件的，它会利用一个内部的`ApplicationEventMulticaster` 在上下文实际被刷新之前对事件进行处理。

```java
private SpringApplicationRunListeners getRunListeners(String[] args) {
  Class<?>[] types = new Class<?>[] { SpringApplication.class, String[].class };
  return new SpringApplicationRunListeners(logger, getSpringFactoriesInstances(
      SpringApplicationRunListener.class, types, this, args));
}
```

## 根据SpringApplicationRunListeners以及参数来准备环境
```java
private ConfigurableEnvironment prepareEnvironment(
    SpringApplicationRunListeners listeners,
    ApplicationArguments applicationArguments) {
  // Create and configure the environment
  // 根据 webApplicationType，创建 StandardServletEnvironment，
  // StandardReactiveWebEnvironment或StandardEnvironment 的环境变量
  ConfigurableEnvironment environment = getOrCreateEnvironment();
  configureEnvironment(environment, applicationArguments.getSourceArgs());
  listeners.environmentPrepared(environment);
  bindToSpringApplication(environment);
  if (!this.isCustomEnvironment) {
    environment = new EnvironmentConverter(getClassLoader())
        .convertEnvironmentIfNecessary(environment, deduceEnvironmentClass());
  }
  ConfigurationPropertySources.attach(environment);
  return environment;
}
```
### getOrCreateEnvironment
根据 webApplicationType，创建对应的环境变量： `StandardServletEnvironment`，`StandardReactiveWebEnvironment` 或 `StandardEnvironment` ；
```java
private ConfigurableEnvironment getOrCreateEnvironment() {
  if (this.environment != null) {
    return this.environment;
  }
  switch (this.webApplicationType) {
  case SERVLET:
    return new StandardServletEnvironment();
  case REACTIVE:
    return new StandardReactiveWebEnvironment();
  default:
    return new StandardEnvironment();
  }
}
```
### configureEnvironment
配置环境变量：
1. 配置 Property Sources
2. 配置 Profiles
```java
protected void configureEnvironment(ConfigurableEnvironment environment,
        String[] args) {
  if (this.addConversionService) {
    ConversionService conversionService = ApplicationConversionService
        .getSharedInstance();
    environment.setConversionService(
        (ConfigurableConversionService) conversionService);
  }
  configurePropertySources(environment, args);
  configureProfiles(environment, args);
}
```
### listeners.environmentPrepared;
在已注册的监听器上注册environment就绪；广播 Event（ApplicationEnvironmentPreparedEvent）
```java
public void environmentPrepared(ConfigurableEnvironment environment) {
  this.initialMulticaster.multicastEvent(new ApplicationEnvironmentPreparedEvent(
      this.application, this.args, environment));
}
```

## 创建Spring上下文
根据 WebApplicactionType的类型，对应创建：
1. `AnnotationConfigApplicationContext`
2. `AnnotationConfigServletWebServerApplicationContext`
3. `AnnotationConfigReactiveWebServerApplicationContext`
```java
public static final String DEFAULT_CONTEXT_CLASS = "org.springframework.context."
      + "annotation.AnnotationConfigApplicationContext";

public static final String DEFAULT_SERVLET_WEB_CONTEXT_CLASS = "org.springframework.boot."
      + "web.servlet.context.AnnotationConfigServletWebServerApplicationContext";

public static final String DEFAULT_REACTIVE_WEB_CONTEXT_CLASS = "org.springframework."
      + "boot.web.reactive.context.AnnotationConfigReactiveWebServerApplicationContext";

protected ConfigurableApplicationContext createApplicationContext() {
  Class<?> contextClass = this.applicationContextClass;
  if (contextClass == null) {
    try {
      switch (this.webApplicationType) {
      case SERVLET:
        contextClass = Class.forName(DEFAULT_SERVLET_WEB_CONTEXT_CLASS);
        break;
      case REACTIVE:
        contextClass = Class.forName(DEFAULT_REACTIVE_WEB_CONTEXT_CLASS);
        break;
      default:
        contextClass = Class.forName(DEFAULT_CONTEXT_CLASS);
      }
    }
    ...
  }
  return (ConfigurableApplicationContext) BeanUtils.instantiateClass(contextClass);
}
```

## Spring上下文前置处理

```java
private void prepareContext(ConfigurableApplicationContext context,
      ConfigurableEnvironment environment, SpringApplicationRunListeners listeners,
      ApplicationArguments applicationArguments, Banner printedBanner) {
  context.setEnvironment(environment);
  // 为上下文配置Bean生成器以及资源加载器(如果它们非空) ？？ 用途待明确
  postProcessApplicationContext(context);
  // 调用初始化器
  applyInitializers(context);
  // 触发Spring Boot启动过程的contextPrepared事件
  listeners.contextPrepared(context);
  if (this.logStartupInfo) {
    logStartupInfo(context.getParent() == null);
    logStartupProfileInfo(context);
  }
  // Add boot specific singleton beans
  // 添加两个Spring Boot中的特殊单例Beans - springApplicationArguments以及springBootBanner
  ConfigurableListableBeanFactory beanFactory = context.getBeanFactory();
  beanFactory.registerSingleton("springApplicationArguments", applicationArguments);
  if (printedBanner != null) {
    beanFactory.registerSingleton("springBootBanner", printedBanner);
  }
  if (beanFactory instanceof DefaultListableBeanFactory) {
    ((DefaultListableBeanFactory) beanFactory)
        .setAllowBeanDefinitionOverriding(this.allowBeanDefinitionOverriding);
  }
  // Load the sources
  // 加载sources - 对于DemoApplication而言，这里的sources集合只包含了它一个class对象
  Set<Object> sources = getAllSources();
  Assert.notEmpty(sources, "Sources must not be empty");
  // 加载动作 - 构造BeanDefinitionLoader并完成Bean定义的加载
  load(context, sources.toArray(new Object[0]));
  // 触发Spring Boot启动过程的contextLoaded事件
  listeners.contextLoaded(context);
}
```

### 执行 applyInitializers(context)
调用 `ApplicationContextInitializer.initialize(..)`
适用于执行`ConfigurableApplicationContext#refresh()`前，需要通过代码执行初始化 `ConfigurableApplicationContext`的场景；

## refreshContext(context)
调用 `AbstractApplicationContext#refresh` 方法刷新applicationContext
```java
// 增加可能的context配置后，需要刷新applicationContext
private void refreshContext(ConfigurableApplicationContext context) {
  refresh(context);
  if (this.registerShutdownHook) {
    try {
      context.registerShutdownHook();
    }
    catch (AccessControlException ex) {
      // Not allowed in some environments.
    }
  }
}

protected void refresh(ApplicationContext applicationContext) {
  Assert.isInstanceOf(AbstractApplicationContext.class, applicationContext);
      ((AbstractApplicationContext) applicationContext).refresh();
}
```

## context刷新后实现
```java
// 刷新 applicationContext
afterRefresh(context, applicationArguments);
// 启动监听器
listeners.started(context);
callRunners(context, applicationArguments);
```
### afterRefresh
默认是一个空实现
```java
protected void afterRefresh(ConfigurableApplicationContext context,
      ApplicationArguments args) {
}
```

### 启动Runner
启动 `ApplicationRunner`，`CommandLineRunner` 类型的Runner；
- `ApplicationRunner`
- `CommandLineRunner`
两个Runner的差别是，一个接收参数是原始的 `String...`，一个是解析后的 `ApplicationArguments`
```java
/**
 * Interface used to indicate that a bean should <em>run</em> when it is contained within
 * a {@link SpringApplication}. Multiple {@link ApplicationRunner} beans can be defined
 * within the same application context and can be ordered using the {@link Ordered}
 * interface or {@link Order @Order} annotation.
 * @see CommandLineRunner
 */
@FunctionalInterface
public interface ApplicationRunner {

  /**
   * Callback used to run the bean.
   * @param args incoming application arguments
   * @throws Exception on error
   */
  void run(ApplicationArguments args) throws Exception;
}

/**
 * Interface used to indicate that a bean should <em>run</em> when it is contained within
 * a {@link SpringApplication}. Multiple {@link CommandLineRunner} beans can be defined
 * within the same application context and can be ordered using the {@link Ordered}
 * interface or {@link Order @Order} annotation.
 * <p>
 * If you need access to {@link ApplicationArguments} instead of the raw String array
 * consider using {@link ApplicationRunner}.
 * @see ApplicationRunner
 */
@FunctionalInterface
public interface CommandLineRunner {
  /**
  * Callback used to run the bean.
   * @param args incoming main method arguments
   * @throws Exception on error
   */
  void run(String... args) throws Exception;
}
```

# Spring启动阶段可执行扩展
- 初始化器(Initializer)
- 监听器(Listener)
- 容器刷新后置Runners(ApplicationRunner或者CommandLineRunner接口的实现类)
- 启动期间在Console打印Banner的具体实现类

# 附加
## getSpringFactoriesInstances
获取配置中指定类型的类名列表，并完成实例化；
方法入口`getSpringFactoriesInstances`，从 `META-INF/spring.factories` 文件中读取 key 为指定类全路径名的配置数据；
```java
private <T> Collection<T> getSpringFactoriesInstances(Class<T> type,
		Class<?>[] parameterTypes, Object... args) {
  ClassLoader classLoader = getClassLoader();
  // Use names and ensure unique to protect against duplicates
  Set<String> names = new LinkedHashSet<>(
    SpringFactoriesLoader.loadFactoryNames(type, classLoader));
  List<T> instances = createSpringFactoriesInstances(type, parameterTypes,
    classLoader, args, names);
  AnnotationAwareOrderComparator.sort(instances);
  return instances;
}

private <T> List<T> createSpringFactoriesInstances(Class<T> type,
    Class<?>[] parameterTypes, ClassLoader classLoader, Object[] args,
    Set<String> names) {
  List<T> instances = new ArrayList<>(names.size());
  for (String name : names) {
    try {
      Class<?> instanceClass = ClassUtils.forName(name, classLoader);
      Assert.isAssignable(type, instanceClass);
      Constructor<?> constructor = instanceClass
        .getDeclaredConstructor(parameterTypes);
      T instance = (T) BeanUtils.instantiateClass(constructor, args);
      instances.add(instance);
    }
    catch (Throwable ex) {
      throw new IllegalArgumentException(
        "Cannot instantiate " + type + " : " + name, ex);
    }
  }
  return instances;
}
```
