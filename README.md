# Spring Boot

### 启动流程

#### 1.SpringApplication实例化

```java
 new SpringApplication(primarySources)
```

1. 配置`resourceLoader`
2. 获取当前应用程序的类型`WebApplicationType.SERVLET`
3.  获取初始化容器的实例对象

    ```java
    setInitializers((Collection)getSpringFactoriesInstances(ApplicationContextInitializer.class));
    ```

    > Spi加载org.springframework.context.ApplicationContextInitializer
    >
    > CloudFoundryVcapEnvironmentPostProcessor ConfigFileApplicationListener AnsiOutputApplicationListener LoggingApplicationListener BackgroundPreinitializer ClasspathLoggingApplicationListener ParentContextCloserApplicationListener ClearCachesApplicationListener FileEncodingApplicationListener LiquibaseServiceLocatorApplicationListener
4.  获取监听器的实例对象并设置

    ```java
    setListeners((Collection)getSpringFactoriesInstances(ApplicationListener.class));
    ```
5. 找到应用程序的主类，开始执行

#### 2.执行run方法

**1 获取SpringApplicationRunListeners监听器**

```java
SpringApplicationRunListeners listeners = getRunListeners(args);`
```

> org.springframework.boot.SpringApplicationRunListener=\
> org.springframework.boot.context.event.EventPublishingRunListener

**1.1 反射实例化EventPublishingRunListener**

实例化时加入第一步创建的监听器对象

```java
this.initialMulticaster.addApplicationListener(listener);
```

**2 启动监听器**

调用SimpleApplicationEventMulticaster（事件发布器）去发布对应类型的事件

```java
listeners.starting();
```

1.  循环启动`SpringApplicationRunListener`

    ```java
    for (SpringApplicationRunListener listener : this.listeners) {
       listener.starting();
    }
    ```
2.  获取`ApplicationListener`

    ```java
    getApplicationListeners(event, type)
    ```

    筛选出符合条件的监听器

    ```java
    supportsEvent(listener, eventType, sourceType)
    ```
3.  监听器的事件发布

    ```java
    listener.onApplicationEvent(event);
    ```

* LoggingApplicationListener
* BackgroundPreinitializer
* DelegatingApplicationListener
* LiquibaseServiceLocatorApplicationListener

**3 准备上下文环境信息**

```java
ConfigurableEnvironment environment = prepareEnvironment(listeners, applicationArguments);
```

**3.1 创建环境**

```java
ConfigurableEnvironment environment = getOrCreateEnvironment();
```

创建Servelt环境对象

1. 装载`servletConfigInitParams`
2. 装载`servletContextInitParams`
3. 装载系统变量（`systemProperties`）

```java
propertySources.addLast(new PropertiesPropertySource(SYSTEM_PROPERTIES_PROPERTY_SOURCE_NAME, getSystemProperties()));
```

1. 装载系统环境（`systemEnvironment`）

```java
propertySources.addLast(new SystemEnvironmentPropertySource(SYSTEM_ENVIRONMENT_PROPERTY_SOURCE_NAME, getSystemEnvironment()));
```

![image-20220505204754382](http://cdn.blocketh.top/img/image-20220505204754382.png)

**3.2 根据用户配置，配置系统环境**

```java
configureEnvironment(environment, applicationArguments.getSourceArgs());
```

1.  **注册转换器**

    ```java
    ConversionService conversionService = ApplicationConversionService.getSharedInstance();
    ```
2.  **配置属性源**

    ```java
    configurePropertySources(environment, args);
    ```

    1.  默认**Properties**

        ```java
        if (this.defaultProperties != null && !this.defaultProperties.isEmpty()) {
           sources.addLast(new MapPropertySource("defaultProperties", this.defaultProperties));
        }
        ```
    2.  Main方法启动参数源，添加到`MutablePropertySources`First位置

        ```java
        sources.addFirst(new SimpleCommandLinePropertySource(args));
        ```
3.  **激活配置环境**

    ```java
    configureProfiles(environment, args);
    ```

    spring.profiles.active配置生效

    ```java
    protected void configureProfiles(ConfigurableEnvironment environment, String[] args) {
       Set<String> profiles = new LinkedHashSet<>(this.additionalProfiles);
       profiles.addAll(Arrays.asList(environment.getActiveProfiles()));
       environment.setActiveProfiles(StringUtils.toStringArray(profiles));
    }
    ```

**3.3 添加configurationProperties**

```java
ConfigurationPropertySources.attach(environment);
```

`propertySources`中现包含2.3.1创建+configurationProperties（五个）

**3.4 环境事件监听**

```java
listeners.environmentPrepared(environment);
```

1. 发布事件广播，包装成`ApplicationEnvironmentPreparedEvent`
2.  获取`ApplicationListener`

    ```java
    getApplicationListeners(event, type)
    ```

    从1.4扫描的监听器找出符合条件的`ApplicationEnvironmentPreparedEvent`

    ```java
    supportsEvent(listener, eventType, sourceType)
    ```

    > FileEncodingApplicationListener DelegatingApplicationListener ClasspathLoggingApplicationListener BackgroundPreinitializer LoggingApplicationListener AnsiOutputApplicationListener ConfigFileApplicationListener
3.  监听器的事件发布

    ```java
    listener.onApplicationEvent(event);
    ```

    > 最重要的是ConfigFileApplicationListener，加载项目的配置文件

    基于环境的事件发布

    ```java
    onApplicationEnvironmentPreparedEvent((ApplicationEnvironmentPreparedEvent) event);
    ```

    获取环境事件处理器

    ```java
    List<EnvironmentPostProcessor> postProcessors = loadPostProcessors();
    ```

    > **org.springframework.boot.env.EnvironmentPostProcessor=/**
    >
    > SystemEnvironmentPropertySourceEnvironmentPostProcessor
    >
    > SpringApplicationJsonEnvironmentPostProcessor
    >
    > CloudFoundryVcapEnvironmentPostProcessor
    >
    > DebugAgentEnvironmentPostProcessor
    >
    > postProcessors.add(this)；--> ConfigFileApplicationListener implements EnvironmentPostProcessor

    ```java
    postProcessor.postProcessEnvironment(event.getEnvironment(), event.getSpringApplication());
    ```

    **ConfigFileApplicationListener**

    ```java
    @Override
    public void postProcessEnvironment(ConfigurableEnvironment environment, SpringApplication application) {
       addPropertySources(environment, application.getResourceLoader());
    }
    ```

    ```java
    protected void addPropertySources(ConfigurableEnvironment environment, ResourceLoader resourceLoader) {
       RandomValuePropertySource.addToEnvironment(environment);
       new Loader(environment, resourceLoader).load();
    }
    ```

    1.  添加随机数

        ```java
        RandomValuePropertySource.addToEnvironment(environment);
        ```
    2.  load资源加载

        ```java
         new Loader(environment, resourceLoader).load();
        ```

​ ConfigFileApplicationListener资源加载时有默认的优先级

​ ![img](http://cdn.blocketh.top/img/20210712192629529.png)

> 最先加载的配置文件会生效，同时可以通过**spring.conﬁg.location**指定配置文件位置

根据2.1.1注册的监听器筛选出符合条件的监听器（七个）

循环执行七个监听器

```java
postProcessEnvironment(ConfigurableEnvironment environment, SpringApplication application)
```

**执行ConfigFileApplicationListener**

1. 根据SPI机制加载进行实例化

```
org.springframework.boot.env.PropertySourceLoader=\
org.springframework.boot.env.PropertiesPropertySourceLoader,\
org.springframework.boot.env.YamlPropertySourceLoader
```

1. 加载`application`配置文件

​ propertySources中新增`applicationConfig`

**4 配置忽略的Bean**

```java
configureIgnoreBeanInfo(environment);
```

**5 打印Banner图**

```java
Banner printedBanner = printBanner(environment);
```

**6 初始化SpringBoot上下文**

```java
context = createApplicationContext();
```

反射生成`AnnotationConfigServletWebServerApplicationContext`，父类也进行初始化操作，完成IOC容器的创建。

```java
public AnnotationConfigServletWebServerApplicationContext() {
		this.reader = new AnnotatedBeanDefinitionReader(this);
		this.scanner = new ClassPathBeanDefinitionScanner(this);
}
```

![image-20220421231027138](http://cdn.blocketh.top/img/image-20220421231027138.png)

**AnnotatedBeanDefinitionReader**

注册条件判断

```java
this.conditionEvaluator = new ConditionEvaluator(registry, environment, null);
```

* `@ConditionalOnJava`：系统的Java是否符合要求
* `@ConditionalOnBean`：容器中存在指定的Bean
* `@ConditionalOnMissingBean`：容器中不存在指定的Bean
* `@ConditionalOnExpression`：满足SpEL表达式
* `@ConditionalOnClass`：系统中存在指定的类
* `@ConditionalOnMissingClass`：系统中不存在指定的类
* `@ConditionalOnSingleCandidate`：容器中只有一个指定的Bean，或者是首选Bean
* `@ConditionalOnProperty`：系统中指定的属性是否有指定的值
* `@ConditionalOnResource`：类路径下是否存在指定资源文件
* `@ConditionalOnWebApplication`：当前是web环境
* `@ConditionalOnNotWebApplication`：当前不是web环境
* `@ConditionalOnJndi`：JNDI存在指定项

**ClassPathBeanDefinitionScanner**

扫描指定注解`@Compoent`、`@Repository`、`@Service`、`@Controller`

**7 准备上下文**

```java
prepareContext(context, environment, listeners, applicationArguments, printedBanner);
```

**1. 上下文中设置环境**

```java
context.setEnvironment(environment);
```

**2. 上下文中的IOC容器设置2.3.2中的转换器**

```java
context.getBeanFactory().setConversionService(ApplicationConversionService.getSharedInstance());
```

**3. 把1.3中的ApplicationInitializer（初始化器）在上下文刷新之前进行初始化操作**

```java
applyInitializers(context);
```

在context中注册成`BeanFactoryPostProcessor`或`Listener`

```java
this.beanFactoryPostProcessors.add(postProcessor);
//在AbstractApplicationContextzh
```

**4. 向各个监听器发送容器已经准备好的事件**

```java
listeners.contextPrepared(context);
```

同样筛选出符合条件的监听器处理

**5. 打印启动日志**

```java
logStartupInfo(context.getParent() == null);
logStartupProfileInfo(context);
```

**6. 把启动类加载到IOC容器**

```java
load(context, sources.toArray(new Object[0]));
```

1. 创建`BeanDefinitionLoader`
   1.  注解Bean读取

       ```java
       new AnnotatedBeanDefinitionReader(registry)；
       ```
   2.  xml bean读取

       ```java
       new XmlBeanDefinitionReader(registry)；
       ```
   3.  类路径扫描

       ```java
       new ClassPathBeanDefinitionScanner(registry)；
       ```
   4.  扫描添加排除过滤器

       ```java
       addExcludeFilter(new ClassExcludeFilter(sources))；
       ```
2.  加载Spring Boot启动类

    ```java
    loader.load();
    ```

    利用`BeanDefinitionRegistry`向IOC容器注册SpringBootApplication启动类
3.  发布容器已加载事件

    ```java
    listeners.contextLoaded(context);
    ```

**8 刷新上下文**

```java
refreshContext(context);
```

最终调用到Spring中`AbstractApplicationContext.refresh()`操作

**invokeBeanFactoryPostProcessors(beanFactory);**

1. `ConfigurationClassParser.parse(candidates);`
   1.  获取Spring Boot`BeanDefinition`信息，递归解析`configuration`及其超类层次结构。

       ```java
       SourceClass sourceClass = asSourceClass(configClass, filter);
       do {
          sourceClass = doProcessConfigurationClass(configClass, sourceClass, filter);
       }
       while (sourceClass != null);
       ```

       ```java
       doProcessConfigurationClass(
       			ConfigurationClass configClass, SourceClass sourceClass, Predicate<String> filter)
       ```

       ​

       1. 扫描`@Compoend`注解
       2. `@PropertySource`
       3.  获取`@CompoentScans`注解信息

           1. `this.componentScanParser.parse(componentScan, sourceClass.getMetadata().getClassName())`
              1. 初始化`ClassPathBeanDefinitionScanner`，如果`basePackages`为空时赋值启动类所在路径
              2.  排除SpringBootApplication启动类

                  ```
                  scanner.addExcludeFilter(new AbstractTypeHierarchyTraversingFilter(false, false) {
                     @Override
                     protected boolean matchClassName(String className) {
                        return declaringClass.equals(className);
                     }
                  });
                  ```
              3. `scanner.doScan()`
                 1. 获取`classpath*:路径/**/*.class`
                 2.  \`\`isCandidateComponent(metadataReader)\`

                     判断@Compoent`修饰并且排除过滤的类，会回调进上面3中，Sping Boot启动类会被排除。`
                 3.  添加至IOC容器`beanDefinitionMap`中

                     ```java
                     registerBeanDefinition(definitionHolder, this.registry);
                     ```

           > 第一次在doProcessConfigurationClass中@扫描@SpringBootApplication中的basePackages下的@Controller、@Service、@Compoent等组件所在的类封装成BeanDefinition加入DefaultListableBeanFactory的BeanDefinitionMap中

           1.  再次调用parse进行解析，上面扫描出的`BeanDefinition`可能类中存在注解引用

               ```java
               parse(bdCand.getBeanClassName(), holder.getBeanName());
               ```
       4.  扫描`@Import`注解，Starter组件的扫描在这一步完成

           1. `getImports(sourceClass)`获取启动类`@Import`注解
              1.  递归`@SpringBootApplicaion`进行注解收集

                  ```java
                  collectImports(annotation, imports, visited);
                  ```
           2. `processImports(ConfigurationClass configClass, SourceClass currentSourceClass, Collection<SourceClass> importCandidates, Predicate<String> exclusionFilter, boolean checkForCircularImports)`

           ```
            1. `@Import`注解上对应的类
           ```

           ````
               ```java
               Class<?> candidateClass = candidate.loadClass();
               ```
           ````
       5.  实例化类所对应的`ImportSelector`

           会判断目标类是实现`ImportSelector`还是`ImportBeanDefinitionRegistrar`

           *   **`AutoConfigurationPackages$Registrar implements ImportBeanDefinitionRegistrar`**

               添加至SpringBoot Configuration类中

               ```java
               this.importBeanDefinitionRegistrars.put(registrar, importingClassMetadata);
               ```

               > 整合Mybatis时，SpringBoot@MapperScan导入@Import(MapperScannerRegistrar.class)
               >
               > MapperScannerRegistrar implements ImportBeanDefinitionRegistrar也会加入ConfigurationClass中的**importBeanDefinitionRegistrars**
           *   **`AutoConfigurationImportSelector implements DeferredImportSelector`**

               ```java
               this.deferredImportSelectorHandler.handle(configClass, (DeferredImportSelector) selector);
               ```

               第一次`handle`不存在，则加入

               ```java
               this.deferredImportSelectors.add(holder);
               ```

               1.  `this.deferredImportSelectorHandler.process();`自动配置的入口，

                   1.  `handler.processGroupImports();`

                       把上一步注入的`deferredImportSelectors`循环注入`DeferredImportSelectorGroupingHandler`中
                   2. `processGroupImports()`
                      1.  `grouping.getImports()`

                          自动装配的执行，执行到`AutoConfigurationImportSelector.AutoConfigurationGroup`类中

                          1.  进行`DeferredImportSelector.process()`

                              > Spring Boot在这一步进行自动装配，加载META-INF/spring.factories/org.springframework.boot.autoconfigure.EnableAutoConfiguration。
                              >
                              > 内部通过META-INF/spring.factories下AutoConfigurationImportFilter来过滤不需要的bean，通过OnBeanCondition，OnClassCondition，OnWebApplicationCondition来过滤不需要的bean。
                          2.  进行`DeferredImportSelector.selectImports()`

                              返回要装配的类
                   3. `processImports(ConfigurationClass configClass, SourceClass currentSourceClass,Collection<SourceClass> importCandidates, Predicate<String> exclusionFilter,boolean checkForCircularImports)`
                   4.

                   循环遍历自动装配类中的`Import`处理
   2.  循环解析每个`ConfigurationClass`

       `ConfigurationClass = new ConfigurationClass(metadata, beanName)`

       ```
       this.reader.loadBeanDefinitions(configClasses);
       ```

       1.  如果是导入的配置类，把当前类的`BeanDefinition`注入IOC

           ```java
           if (configClass.isImported()) {
           	registerBeanDefinitionForImportedConfigurationClass(configClass);
           }
           ```
       2.  配置类的各种属性进行处理，处理方法上的bean注解，同样封装成`BeanDefiniton`注入IOC

           ```java
           for (BeanMethod beanMethod : configClass.getBeanMethods()) {
              loadBeanDefinitionsForBeanMethod(beanMethod);
           }
           ```
       3.  把扫描`@Import`中实现`ImportBeanDefinitionRegistrar`的类进行循环调用`registerBeanDefinitions(AnnotationMetadata importingClassMetadata, BeanDefinitionRegistry registry)`注册自定义`BeanDefinition`信息到IOC

           ```java
           registrars.forEach((registrar, metadata) ->
                 registrar.registerBeanDefinitions(metadata, this.registry, this.importBeanNameGenerator));
           ```

           > `ServletWebServerFactoryAutoConfiguration`内部类`BeanPostProcessorsRegistrar implements ImportBeanDefinitionRegistrar`会进行`registerBeanDefinitions()`注册，向`BeanDefinitionRegistry`注册`WebServerFactoryCustomizerBeanPostProcessor`

           > 继承Mybatis时，@MapperScam可以集成在SpringBoot启动类上，在SpringBoot的ConfigurationClass配置类`importBeanDefinitionRegistrars`包含MapperScannerRegistrar，执行`registerBeanDefinitions(AnnotationMetadata importingClassMetadata, BeanDefinitionRegistry registry)`时，封装了自定义的`BeanDefinition`信息，生成的是`MapperScannerConfigurer implements BeanDefinitionRegistryPostProcessor`的BeanDefinition，在后续的`invokeBeanFactoryPostprocessors()`时会执行`postProcessBeanDefinitionRegistry()`，会调用到MapperScannerConfigurer，里面定义了针对Mapper扫描的ClassPathMapperScanner，把basePackage下的Mapper定义为BeanDefinition，其中的beanClass定义为MapperFactoryBean，注入IOC容器

**onRefresh()**

1.  从IOC容器中获取WebServerFactory

    ```java
    ServletWebServerFactory factory = getWebServerFactory();
    ```
2.  启动Tomcat

    ```java
    factory.getWebServer(getSelfInitializer());
    ```

**9 运行所有监听器对象**

```java
listeners.started(context)；
```

**callRunners**

从`ApplicationContext`上下文中获取`ApplicationRunner.class`和`CommandLineRunner.clas`类型的类，循环执行`run()`

***

### Spring Boot Actuator

#### 监控检查

/actuator/health

**展示详情**

```yaml
management:
  endpoints:
    health:
      show-details: always
```

#### Metrics

/actiator/metrics 展示系统指标

* JVM 指标，报告利用率：
  * 各种内存和缓冲池
  * 垃圾回收相关统计
  * 线程利用率
  * 加载/卸载的类数
* CPU 指标
* 文件描述符指标
* Kafka 消费者、生产者和流指标
* Log4j2 指标：记录每个级别记录到 Log4j2 的事件数
* Logback 指标：记录每个级别记录到 Logback 的事件数
* 正常运行时间指标：报告正常运行时间的计量器和表示应用程序绝对启动时间的固定计量器
* Tomcat 指标（`server.tomcat.mbeanregistry.enabled`必须设置`true`为所有要注册的 Tomcat 指标）
* [Spring 集成](https://docs.spring.io/spring-integration/docs/5.3.8.RELEASE/reference/html/system-management.html#micrometer-integration)指标

**集成Prometheus**

```
'io.micrometer:micrometer-registry-prometheus:1.8.5'
```

/actuator/prometheus

**promethues客户端**

prometheus.yml

```yaml
  - job_name: "user-service"
    metrics_path: "/actuator/prometheus"
    static_configs:
      - targets: [ "localhost:9091" ]
```

http://localhost:9090

***

### 整合Spring MVC

Servlet3.0规范的诞生为Spring Boot去除XML（web.xml）提供了理论基础

> 实现Servlet3.0规范的容器（如Tomcat7.0及以上）启动时，通过SPI机制扫描所有jar下的META-INF/javax.servlet.ServletContainerInitiallizer指定的全路径的类，并实例化该类，回调该类的onStartup方法

```java
org.springframework.boot.autoconfigure.web.servlet.DispatcherServletAutoConfiguration
```

Spring Boot自动装配Spring MVC其实就是向ServletContext中加入DispatchServlet，除了可以添加Servlet，还可以动态加入Listener、Filter

### 整合Mybatis

#### 自动装配

**数据源**

```
org.springframework.boot.autoconfigure.jdbc.DataSourceAutoConfiguration
```

自动装配解析`DataSourceAutoConfiguration`的@Bean方法，内部Bean`PooledDataSourceConfiguration` 导入了

```
@Import({ DataSourceConfiguration.Hikari.class, DataSourceConfiguration.Tomcat.class,
      DataSourceConfiguration.Dbcp2.class, DataSourceConfiguration.Generic.class,
      DataSourceJmxConfiguration.class })
```

如果`Type`类型未设置

```properties
spring.datasource.type=com.zaxxer.hikari.HikariDataSource
```

findType获取第一个HikariDataSource

```java
private static final String[] DATA_SOURCE_TYPE_NAMES = new String[] { "com.zaxxer.hikari.HikariDataSource",
      "org.apache.tomcat.jdbc.pool.DataSource", "org.apache.commons.dbcp2.BasicDataSource" };
```

**Mybatis**

自动装配的类

```
# Auto Configure
org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
org.mybatis.spring.boot.autoconfigure.MybatisLanguageDriverAutoConfiguration,\
org.mybatis.spring.boot.autoconfigure.MybatisAutoConfiguration
```

**MybatisAutoConfiguration**

目的为了实现Spring Starter组件的自动装配，无需Spring整合Mybatis时手动通过`SqlSessionFactoryBean`创建`SqlSessionFactory`

1.  `@EnableConfigurationProperties(MybatisProperties.class)`

    启用`MybatisProperties`配置
2. 包含`sqlSessionFactory`方法Bean
   1. 创建`SqlSessionFactoryBean`
   2. 获取Mybatis全部配置对象`@`
   3.  `SqlSessionFactoryBean.getObject()`返回DefaultSqlSessionFactory

       实现FactoryBean，getObject()中会进行XMLMapperBuilderP解析，封装`Configuration`
3. 包含`sqlSessionTemplate`方法Bean

添加`@Mapper`或者`@MapperScan`

**SpringBoot启动类添加@MapperScan**

> 包含 @Import(MapperScannerRegistrar.class)

**registerBeanDefinitions()**

1. 把`@MapperScan`里面属性构建成`MapperScannerConfigurer`(实现BeanDefinitionRegistryPostProcessor)类型的`BeanDefinition`。
2. 后续执行`invokeBeanFactoryPocessors`会处理新增的`BeanDefinitionRegistryPostProcessor`，调用`MapperScannerConfigurer。postProcessBeanDefinitionRegistry()`
3. 完成`ClassPathMapperScanner`初始化，扫描basePackage下的Mapper，封装成`MethodFactoryBean`类型的`BeanDefinition`

### 缓存

#### JSR107

JSR，Java的规范请求，JSR-107是关于如何使用缓存的规范，类似JDBC规范，没有具体的实现，具体实现就是Reids等缓存

定义了五个核心接口

* CachingProvider（缓存提供者）：创建、配置、获取、管理和控制多个CacheManager
* CacheManager（缓存管理者）：创建、配置、获取、管理和控制多个唯一命名的Cache，一个CacheManager仅对应一个CachingProvider。
* Cache（缓存）：被CacheManager管理，Cache存在与CacheManager的上下文中，类似Map的数据结构，并临时存储以key为所有的值。一个Cache仅被一个CacheManager所拥有
* Entry（缓存键值对）：是一个存储在Cache中的key-value
* Expiry（缓存时效）：每个存储在Cache中的条目都有一个定义的有效期，一旦超过这个时间，条目就自动过期，过期后，条目将不可以访问、更新和删除，缓存有效期可以通过ExpiryPolicy设置。

![img](http://cdn.blocketh.top/img/1429976-20190423143953487-894773441.png)

### 配置

#### HTTPS配置

```yaml
server:
  port: 443
  ssl:
    key-store: classpath:aaaaa.pfx
    key-store-password: yourpassword
    keyStoreType: PKCS12
```
