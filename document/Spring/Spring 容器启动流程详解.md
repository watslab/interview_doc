# Spring 容器启动流程详解

## 一、概述

Spring 容器的启动流程是理解 Spring 框架的核心。无论是传统的 Spring 应用还是 Spring Boot 应用，容器启动最终都会调用 `AbstractApplicationContext.refresh()` 方法完成初始化。

本文将从传统 Spring 启动流程入手，深入分析 `refresh()` 方法的 12 个核心步骤，再对比 Spring Boot 的启动流程，帮助读者全面理解 Spring 容器的初始化机制。

---

## 二、传统 Spring 容器启动流程

### 2.1 启动入口

传统 Spring 应用通常通过以下方式启动容器：

```java
// XML 配置方式
ApplicationContext context = new ClassPathXmlApplicationContext("applicationContext.xml");

// 注解配置方式
ApplicationContext context = new AnnotationConfigApplicationContext(AppConfig.class);
```

无论哪种方式，最终都会调用 `AbstractApplicationContext.refresh()` 方法。

### 2.2 核心三步骤

以 `AnnotationConfigApplicationContext` 为例，构造方法包含三个核心步骤：

```java
public AnnotationConfigApplicationContext(Class<?>... componentClasses) {
    this();                  // 步骤1：初始化容器基础组件
    register(componentClasses); // 步骤2：注册配置类
    refresh();               // 步骤3：刷新容器（核心）
}
```

#### 步骤1：容器初始化（this()）

完成以下工作：
- 创建 `DefaultListableBeanFactory`（核心 Bean 工厂）
- 创建 `AnnotatedBeanDefinitionReader`（注解解析器）
- 创建 `ClassPathBeanDefinitionScanner`（类路径扫描器）
- 注册 Spring 内置的后置处理器

#### 步骤2：注册配置类（register()）

将配置类解析为 `BeanDefinition` 并注册到容器中。

#### 步骤3：刷新容器（refresh()）

这是容器启动的核心，详见下节。

### 2.3 refresh() 方法详解

`refresh()` 方法定义在 `AbstractApplicationContext` 中，包含 12 个核心步骤：

```java
@Override
public void refresh() throws BeansException, IllegalStateException {
    synchronized (this.startupShutdownMonitor) {
        // 1. 准备刷新
        prepareRefresh();

        // 2. 获取 BeanFactory
        ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();

        // 3. 配置 BeanFactory 标准特性
        prepareBeanFactory(beanFactory);

        try {
            // 4. 子类扩展点
            postProcessBeanFactory(beanFactory);

            // 5. 执行 BeanFactoryPostProcessor
            invokeBeanFactoryPostProcessors(beanFactory);

            // 6. 注册 BeanPostProcessor
            registerBeanPostProcessors(beanFactory);

            // 7. 初始化消息源（国际化）
            initMessageSource();

            // 8. 初始化事件广播器
            initApplicationEventMulticaster();

            // 9. 子类扩展点
            onRefresh();

            // 10. 注册事件监听器
            registerListeners();

            // 11. 实例化所有非懒加载单例 Bean
            finishBeanFactoryInitialization(beanFactory);

            // 12. 完成刷新
            finishRefresh();

        } catch (BeansException ex) {
            destroyBeans();
            cancelRefresh(ex);
            throw ex;
        }
    }
}
```

#### 步骤详解

| 步骤 | 方法名 | 核心职责 |
|------|--------|----------|
| 1 | prepareRefresh() | 初始化启动时间戳、设置 active 标志、验证必需属性 |
| 2 | obtainFreshBeanFactory() | 创建或获取 BeanFactory、加载 BeanDefinition |
| 3 | prepareBeanFactory() | 配置 ClassLoader、注册 Aware 接口处理器、设置忽略依赖接口 |
| 4 | postProcessBeanFactory() | 子类扩展点，允许修改 BeanFactory |
| 5 | invokeBeanFactoryPostProcessors() | 执行 BeanFactoryPostProcessor，解析配置类、扫描包 |
| 6 | registerBeanPostProcessors() | 注册 BeanPostProcessor，用于拦截 Bean 创建过程 |
| 7 | initMessageSource() | 初始化国际化消息源 |
| 8 | initApplicationEventMulticaster() | 初始化事件广播器 |
| 9 | onRefresh() | 子类扩展点，Spring Boot 在此启动内嵌服务器 |
| 10 | registerListeners() | 注册 ApplicationListener |
| 11 | finishBeanFactoryInitialization() | 实例化所有非懒加载单例 Bean（核心） |
| 12 | finishRefresh() | 发布 ContextRefreshedEvent 事件、初始化生命周期处理器 |

### 2.4 关键步骤深入分析

#### 2.4.1 步骤5：invokeBeanFactoryPostProcessors()

这是配置类解析的核心阶段，主要执行 `ConfigurationClassPostProcessor`：

```
ConfigurationClassPostProcessor 执行流程
┌─────────────────────────────────────────────────────────────┐
│  解析 @Configuration 配置类                                   │
├─────────────────────────────────────────────────────────────┤
│  处理 @ComponentScan → 扫描指定包下的组件                       │
├─────────────────────────────────────────────────────────────┤
│  处理 @Import → 导入额外的配置类或 ImportSelector              │
├─────────────────────────────────────────────────────────────┤
│  处理 @ImportResource → 导入 XML 配置文件                      │
├─────────────────────────────────────────────────────────────┤
│  处理 @Bean → 注册方法返回值为 BeanDefinition                  │
└─────────────────────────────────────────────────────────────┘
```

#### 2.4.2 步骤11：finishBeanFactoryInitialization()

这是 Bean 实例化的核心阶段：

```java
protected void finishBeanFactoryInitialization(ConfigurableListableBeanFactory beanFactory) {
    // 初始化转换服务
    if (beanFactory.containsBean(CONVERSION_SERVICE_BEAN_NAME)) {
        beanFactory.setConversionService(
            beanFactory.getBean(CONVERSION_SERVICE_BEAN_NAME, ConversionService.class));
    }

    // 注册默认值解析器（处理 ${} 占位符）
    if (!beanFactory.hasEmbeddedValueResolver()) {
        beanFactory.addEmbeddedValueResolver(strVal -> getEnvironment().resolvePlaceholders(strVal));
    }

    // 实例化所有非懒加载单例 Bean
    beanFactory.preInstantiateSingletons();
}
```

#### 2.4.3 Bean 创建流程

单个 Bean 的创建流程：

```
getBean() → doGetBean() → createBean() → doCreateBean()
                                              │
                    ┌─────────────────────────┴─────────────────────────┐
                    │                                                   │
             createBeanInstance()                              populateBean()
                    │                                                   │
            实例化 Bean 对象                              属性注入（@Autowired 等）
                    │                                                   │
                    └─────────────────────────┬─────────────────────────┘
                                              │
                                     initializeBean()
                                              │
                    ┌─────────────────────────┼─────────────────────────┐
                    │                         │                         │
             invokeAwareMethods()    applyBeanPostProcessorsBeforeInitialization()
            (设置 Aware 接口)              (前置处理)                    │
                                              │                         │
                                      invokeInitMethods()    applyBeanPostProcessorsAfterInitialization()
                                      (执行初始化方法)            (后置处理，AOP 代理在此生成)
```

---

## 三、Spring Boot 启动流程

### 3.1 启动入口

Spring Boot 应用的启动入口：

```java
@SpringBootApplication
public class Application {
    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }
}
```

### 3.2 SpringApplication.run() 核心流程

```java
public ConfigurableApplicationContext run(String... args) {
    // 1. 记录启动时间
    StopWatch stopWatch = new StopWatch();
    stopWatch.start();

    // 2. 获取 SpringApplicationRunListeners
    SpringApplicationRunListeners listeners = getRunListeners(args);

    // 3. 发布 starting 事件
    listeners.starting();

    try {
        // 4. 准备环境
        ApplicationArguments applicationArguments = new DefaultApplicationArguments(args);
        ConfigurableEnvironment environment = prepareEnvironment(listeners, applicationArguments);

        // 5. 打印 Banner
        printBanner(environment);

        // 6. 创建 ApplicationContext
        context = createApplicationContext();

        // 7. 准备上下文
        prepareContext(context, environment, listeners, applicationArguments, printedBanner);

        // 8. 刷新上下文（核心）
        refreshContext(context);

        // 9. 刷新后处理
        afterRefresh(context, applicationArguments);

        // 10. 发布 started 事件
        listeners.started(context);

        // 11. 执行 ApplicationRunner 和 CommandLineRunner
        callRunners(context, applicationArguments);

    } catch (Throwable ex) {
        handleRunFailure(context, ex, listeners);
        throw new IllegalStateException(ex);
    }

    return context;
}
```

### 3.3 Spring Boot 启动流程图

```
SpringApplication.run()
         │
         ▼
┌─────────────────────────────────────────────────────────────────┐
│  1. getRunListeners() - 获取启动监听器                            │
│     从 META-INF/spring.factories 加载 SpringApplicationRunListener │
└─────────────────────────────────────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────────────────────────────────┐
│  2. prepareEnvironment() - 准备环境                              │
│     创建 StandardServletEnvironment                              │
│     加载 application.yml/properties 配置                          │
│     发布 ApplicationEnvironmentPreparedEvent                     │
└─────────────────────────────────────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────────────────────────────────┐
│  3. createApplicationContext() - 创建上下文                       │
│     Web 应用：AnnotationConfigServletWebServerApplicationContext │
│     非 Web：AnnotationConfigApplicationContext                    │
└─────────────────────────────────────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────────────────────────────────┐
│  4. prepareContext() - 准备上下文                                 │
│     设置 Environment                                             │
│     注册主配置类为 BeanDefinition                                  │
│     执行 ApplicationContextInitializer                           │
│     发布 contextPrepared 事件                                     │
└─────────────────────────────────────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────────────────────────────────┐
│  5. refreshContext() - 刷新上下文（核心）                          │
│     调用 AbstractApplicationContext.refresh()                    │
│     在 onRefresh() 中启动内嵌 Tomcat/Jetty                        │
│     执行自动配置（@EnableAutoConfiguration）                       │
└─────────────────────────────────────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────────────────────────────────┐
│  6. afterRefresh() - 刷新后处理                                   │
│     空实现，留给子类扩展                                           │
└─────────────────────────────────────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────────────────────────────────┐
│  7. callRunners() - 执行 Runner                                   │
│     执行所有 ApplicationRunner                                    │
│     执行所有 CommandLineRunner                                    │
└─────────────────────────────────────────────────────────────────┘
```

### 3.4 自动配置原理

Spring Boot 的核心特性之一是自动配置，通过 `@EnableAutoConfiguration` 实现：

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
@AutoConfigurationPackage
@Import(AutoConfigurationImportSelector.class)
public @interface EnableAutoConfiguration {
    // ...
}
```

自动配置流程：

```
@EnableAutoConfiguration
         │
         ▼
AutoConfigurationImportSelector
         │
         ▼
┌─────────────────────────────────────────────────────────────────┐
│  加载 META-INF/spring.factories 中的自动配置类                     │
│  如：WebMvcAutoConfiguration、DataSourceAutoConfiguration 等      │
└─────────────────────────────────────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────────────────────────────────┐
│  条件过滤（@Conditional 注解）                                     │
│  @ConditionalOnClass - 类路径存在指定类                            │
│  @ConditionalOnBean - 容器中存在指定 Bean                          │
│  @ConditionalOnProperty - 配置属性满足条件                         │
│  @ConditionalOnMissingBean - 容器中不存在指定 Bean                 │
└─────────────────────────────────────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────────────────────────────────┐
│  注册满足条件的自动配置 Bean                                        │
└─────────────────────────────────────────────────────────────────┘
```

---

## 四、传统 Spring 与 Spring Boot 启动对比

### 4.1 启动流程对比

| 对比维度 | 传统 Spring | Spring Boot |
|----------|-------------|-------------|
| 启动入口 | `new ClassPathXmlApplicationContext()` 或 `new AnnotationConfigApplicationContext()` | `SpringApplication.run()` |
| 配置加载 | 手动指定 XML 或配置类 | 自动扫描 + 自动配置 |
| Bean 注册 | 手动配置或 `@ComponentScan` 指定包 | 自动配置 + 条件装配 |
| Web 容器 | 需要外部 Tomcat/Jetty | 内嵌 Tomcat/Jetty/Undertow |
| 依赖管理 | 手动声明所有依赖及版本 | Starter 封装，自动版本管理 |
| 配置文件 | XML 或 JavaConfig | application.yml/properties + 自动配置 |

### 4.2 启动流程差异图

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                          传统 Spring 启动流程                                 │
├─────────────────────────────────────────────────────────────────────────────┤
│  main()                                                                     │
│    │                                                                        │
│    ▼                                                                        │
│  new ClassPathXmlApplicationContext("applicationContext.xml")               │
│    │                                                                        │
│    ▼                                                                        │
│  加载 XML 配置 → 注册 BeanDefinition                                         │
│    │                                                                        │
│    ▼                                                                        │
│  refresh() → 初始化容器                                                      │
│    │                                                                        │
│    ▼                                                                        │
│  部署 WAR 包到外部 Tomcat                                                    │
└─────────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────────┐
│                          Spring Boot 启动流程                                │
├─────────────────────────────────────────────────────────────────────────────┤
│  main()                                                                     │
│    │                                                                        │
│    ▼                                                                        │
│  SpringApplication.run(Application.class, args)                             │
│    │                                                                        │
│    ├─→ 创建 SpringApplication                                                │
│    │     ├─→ 推断应用类型（Web/Reactive/None）                               │
│    │     ├─→ 加载 ApplicationContextInitializer                             │
│    │     └─→ 加载 ApplicationListener                                       │
│    │                                                                        │
│    ├─→ 准备 Environment（加载 application.yml）                              │
│    │                                                                        │
│    ├─→ 创建 ApplicationContext                                               │
│    │                                                                        │
│    ├─→ prepareContext()（注册主配置类、执行 Initializer）                     │
│    │                                                                        │
│    ├─→ refreshContext()                                                     │
│    │     ├─→ refresh()（传统 Spring 的 12 步骤）                             │
│    │     ├─→ onRefresh() 启动内嵌 Tomcat                                     │
│    │     └─→ 执行自动配置（@EnableAutoConfiguration）                         │
│    │                                                                        │
│    └─→ callRunners()（执行 ApplicationRunner/CommandLineRunner）             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 4.3 核心差异总结

#### 4.3.1 配置方式

**传统 Spring：**
```xml
<!-- applicationContext.xml -->
<beans>
    <bean id="userService" class="com.example.UserService"/>
    <context:component-scan base-package="com.example"/>
</beans>
```

或

```java
@Configuration
@ComponentScan("com.example")
public class AppConfig {
    @Bean
    public UserService userService() {
        return new UserService();
    }
}
```

**Spring Boot：**
```java
@SpringBootApplication // 包含 @Configuration + @ComponentScan + @EnableAutoConfiguration
public class Application {
    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }
}
```

#### 4.3.2 依赖管理

**传统 Spring：**
```xml
<dependencies>
    <dependency>
        <groupId>org.springframework</groupId>
        <artifactId>spring-webmvc</artifactId>
        <version>5.3.20</version>
    </dependency>
    <dependency>
        <groupId>com.fasterxml.jackson.core</groupId>
        <artifactId>jackson-databind</artifactId>
        <version>2.13.0</version>
    </dependency>
    <!-- 需要手动管理所有依赖及版本 -->
</dependencies>
```

**Spring Boot：**
```xml
<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
    <!-- Starter 自动引入所有相关依赖，版本由 spring-boot-starter-parent 管理 -->
</dependencies>
```

#### 4.3.3 Web 容器启动

**传统 Spring：**
- 打包为 WAR 文件
- 部署到外部 Tomcat/Jetty
- 需要配置 web.xml 或 ServletContainerInitializer

**Spring Boot：**
```java
// 在 onRefresh() 阶段启动内嵌服务器
protected void onRefresh() {
    super.onRefresh();
    createWebServer(); // 创建并启动 Tomcat/Jetty/Undertow
}
```

#### 4.3.4 条件装配

Spring Boot 通过条件注解实现按需加载：

```java
@Configuration
@ConditionalOnClass({ Servlet.class, DispatcherServlet.class })
@ConditionalOnWebApplication(type = Type.SERVLET)
@AutoConfigureAfter(ServiceConnectionsAutoConfiguration.class)
public class WebMvcAutoConfiguration {

    @Bean
    @ConditionalOnMissingBean
    public InternalResourceViewResolver defaultViewResolver() {
        return new InternalResourceViewResolver();
    }

    @Bean
    @ConditionalOnMissingBean
    @ConditionalOnProperty(prefix = "spring.mvc.contentnegotiation", 
                           name = "favor-parameter", havingValue = "true")
    public ParameterContentNegotiationStrategy parameterContentNegotiationStrategy() {
        return new ParameterContentNegotiationStrategy(getMediaTypes());
    }
}
```

---

## 五、关键组件说明

### 5.1 BeanDefinition

BeanDefinition 是 Bean 的元数据描述，包含：

| 属性 | 说明 |
|------|------|
| beanClassName | Bean 的全限定类名 |
| scope | 作用域（singleton/prototype） |
| lazyInit | 是否懒加载 |
| dependsOn | 依赖的其他 Bean |
| initMethodName | 初始化方法名 |
| destroyMethodName | 销毁方法名 |
| propertyValues | 属性值集合 |

### 5.2 BeanFactoryPostProcessor

在 Bean 实例化之前执行，用于修改 BeanDefinition：

```java
@FunctionalInterface
public interface BeanFactoryPostProcessor {
    void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) throws BeansException;
}
```

典型实现：
- `ConfigurationClassPostProcessor`：解析配置类、扫描包
- `PropertySourcesPlaceholderConfigurer`：解析 `${}` 占位符

### 5.3 BeanPostProcessor

在 Bean 实例化后执行，用于拦截 Bean 的初始化过程：

```java
public interface BeanPostProcessor {
    default Object postProcessBeforeInitialization(Object bean, String beanName) {
        return bean;
    }

    default Object postProcessAfterInitialization(Object bean, String beanName) {
        return bean;
    }
}
```

典型实现：
- `AutowiredAnnotationBeanPostProcessor`：处理 `@Autowired`
- `CommonAnnotationBeanPostProcessor`：处理 `@Resource`、`@PostConstruct`
- `AnnotationAwareAspectJAutoProxyCreator`：创建 AOP 代理

### 5.4 ApplicationContextInitializer

在 `refresh()` 之前执行，用于编程式地初始化 ApplicationContext：

```java
@FunctionalInterface
public interface ApplicationContextInitializer<C extends ConfigurableApplicationContext> {
    void initialize(C applicationContext);
}
```

注册方式：
```java
// spring.factories
org.springframework.context.ApplicationContextInitializer=\
com.example.MyInitializer
```

### 5.5 ApplicationListener

监听 Spring 事件：

```java
@FunctionalInterface
public interface ApplicationListener<E extends ApplicationEvent> {
    void onApplicationEvent(E event);
}
```

常用事件：
- `ContextRefreshedEvent`：容器刷新完成
- `ContextStartedEvent`：容器启动
- `ContextStoppedEvent`：容器停止
- `ContextClosedEvent`：容器关闭

---

## 六、常见问题

### 6.1 Bean 循环依赖

**问题：** A 依赖 B，B 依赖 A，导致启动失败。

**解决方案：**
- 构造器注入：无法解决，需重构代码
- Setter 注入：Spring 通过三级缓存解决

三级缓存：
```java
// 一级缓存：完整的 Bean
private final Map<String, Object> singletonObjects = new ConcurrentHashMap<>(256);

// 二级缓存：早期暴露的 Bean（已实例化，未填充属性）
private final Map<String, Object> earlySingletonObjects = new HashMap<>(16);

// 三级缓存：Bean 工厂（用于生成早期 Bean）
private final Map<String, ObjectFactory<?>> singletonFactories = new HashMap<>(16);
```

### 6.2 BeanFactoryPostProcessor 执行顺序

通过 `@Order` 或 `PriorityOrdered` 控制顺序：

```java
@Component
@Order(1)
public class MyBeanFactoryPostProcessor implements BeanFactoryPostProcessor, PriorityOrdered {
    @Override
    public int getOrder() {
        return 1;
    }
}
```

执行顺序：
1. 实现 `PriorityOrdered` 接口的
2. 实现 `Ordered` 接口的
3. 未实现排序接口的

### 6.3 如何在容器启动时执行代码

方式一：实现 `ApplicationRunner`

```java
@Component
public class MyApplicationRunner implements ApplicationRunner {
    @Override
    public void run(ApplicationArguments args) {
        // 容器启动后执行
    }
}
```

方式二：实现 `CommandLineRunner`

```java
@Component
public class MyCommandLineRunner implements CommandLineRunner {
    @Override
    public void run(String... args) {
        // 容器启动后执行
    }
}
```

方式三：监听 `ContextRefreshedEvent`

```java
@Component
public class MyListener implements ApplicationListener<ContextRefreshedEvent> {
    @Override
    public void onApplicationEvent(ContextRefreshedEvent event) {
        // 容器刷新完成时执行
    }
}
```

---

## 七、总结

### 7.1 核心要点

1. **统一入口**：无论是传统 Spring 还是 Spring Boot，容器启动最终都调用 `AbstractApplicationContext.refresh()`

2. **refresh() 12 步骤**：
   - 准备阶段（1-4）：环境准备、BeanFactory 创建
   - 扩展阶段（5-6）：执行后置处理器
   - 功能阶段（7-10）：国际化、事件、监听器
   - 实例化阶段（11-12）：Bean 创建、完成通知

3. **Spring Boot 增强**：
   - 自动配置：`@EnableAutoConfiguration` + 条件装配
   - 内嵌服务器：`onRefresh()` 中启动
   - Starter 依赖管理：简化配置

4. **扩展机制**：
   - `BeanFactoryPostProcessor`：修改 BeanDefinition
   - `BeanPostProcessor`：拦截 Bean 创建
   - `ApplicationContextInitializer`：初始化上下文
   - `ApplicationListener`：监听事件

### 7.2 学习建议

1. 从 `AbstractApplicationContext.refresh()` 入手，理解整体流程
2. 关注 `invokeBeanFactoryPostProcessors()` 和 `finishBeanFactoryInitialization()` 两个核心步骤
3. 理解 `BeanPostProcessor` 的作用，这是 Spring 扩展性的基础
4. 对比传统 Spring 和 Spring Boot 的启动差异，理解 Spring Boot 的设计理念
