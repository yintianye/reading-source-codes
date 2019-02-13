# Spring Boot源码1.2------prepareContext()

首先方法参数列表为

```java
private void prepareContext(
    ConfigurableApplicationContext context,
    ConfigurableEnvironment environment, 
    SpringApplicationRunListeners listeners,
    ApplicationArguments applicationArguments, 
    Banner printedBanner
) {}
```

能够看出来参数主要是这些。那我们分步骤看看prepareContext都干了点儿啥。

* 1是context.setEnvironment(environment)。

  * 即把StandardServletEnvironment给AnnotationConfigEmbeddedWebApplicationContext

* 2是postProcessApplicationContext(context)

  * 由于默认SpringApplication的beanNameGenerator和resourceloader都是null，所以这一步是直接跳过的。

* 3是applyInitializers(context)

  * 处理所有的ApplicationContextInitializer，调用每个initializer.initialize(context)方法。
  * 默认的initializer有6个，都是spring-boot和spring-boot-autoconfigure包里的
    * org.springframework.boot.context.config.DelegatingApplicationContextInitializer
    * org.springframework.boot.context.ContextIdApplicationContextInitializer
    * org.springframework.boot.context.ConfigurationWarningsApplicationContextInitializer
    * org.springframework.boot.context.embedded.ServerPortInfoApplicationContextInitializer
    * org.springframework.boot.autoconfigure.SharedMetadataReaderFactoryContextInitializer
    * org.springframework.boot.autoconfigure.logging.AutoConfigurationReportLoggingInitializer
  * 有时间可以看看这些Initializer都干了些啥。

* 4是执行所有listeners.contextPrepared(context)方法

* 5是打startupInfo的日志

  * ```java
    if (this.logStartupInfo) {
    	logStartupInfo(context.getParent() == null);
    	logStartupProfileInfo(context);
    }
    
    //打出如下两句。
    //2018-11-25 01:31:28.217  INFO 3656 --- [           main] t.t.AlertManagerMain                     : Starting AlertManagerMain on htf-yintianye-nb with PID 3656 (D:\IdeaProjects\pureSpringBoot\alert-manager\target\classes started by yintianye in D:\IdeaProjects\pureSpringBoot)
    //2018-11-25 01:31:28.229  INFO 3656 --- [           main] t.t.AlertManagerMain                     : No active profile set, falling back to default profiles: default
    ```

* 6是注册两个单例：SpringApplicationArguments和SpringBootBanner

* 7是Load the sources

  * 默认拿出来的Source是主类的class

  * 生成了一个BeanDefinitionLoader！！这是Spring Boot的BeanDefinitionLoader

    ```java
    //这里的设计很冗余，你会发现BeanDefinitionLoader里面也有成员变量
    //AnnotatedBeanDefinitionReader和ClassPathBeanDefinitionScanner
    //而这个registry是AnnotationConfigEmbeddedWebApplicationContext，也有这些玩意儿
    
    BeanDefinitionLoader(BeanDefinitionRegistry registry, Object... sources) {
    	Assert.notNull(registry, "Registry must not be null");
    	Assert.notEmpty(sources, "Sources must not be empty");
    	this.sources = sources;
    	this.annotatedReader = new AnnotatedBeanDefinitionReader(registry);
    	this.xmlReader = new XmlBeanDefinitionReader(registry);
    	if (isGroovyPresent()) {
    		this.groovyReader = new GroovyBeanDefinitionReader(registry);
    	}
    	this.scanner = new ClassPathBeanDefinitionScanner(registry);
    	this.scanner.addExcludeFilter(new ClassExcludeFilter(sources));
    }
    ```

    

  * 执行这个loader.load();

    * 由于默认的情况这里传入的sources是主类，所以load的时候就是判断了一下是不是@Component，然后使用AnnotatedBeanDefinitionReader.registerBean

-------------------------------------------------

总结一下，prepareContext似乎只加了一个Bean上去，也就是主类的Bean。