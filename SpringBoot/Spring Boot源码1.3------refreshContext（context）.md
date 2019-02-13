# Spring Boot源码1.3------refreshContext（context）

* 这个方法是整个spring boot启动过程中最重要的方法，没有之一。
* 也是原生Spring中的必经之路
* 做法就是把当前的AnnotationConfigEmbeddedWebApplicationContext强制调用其父类AbstractApplicationContext的refresh方法



这里面的非常严谨，带有注释的主流程方法有12步。

1. prepareRefresh

   ```java
   // Prepare this context for refreshing.
   prepareRefresh();
   ```

   * 首先会去initPropertySources
     * 调用了GenericWebApplicationContext的initPropertySources
     * 转至调用StandardServletEnvironment的initPropertySources
     * 实际做了件事就是重新覆盖默认的propertySourceList，但是由于ServletContext和ServletConfig都是null，因此没有覆盖什么配置。
   * 接着会去getEnvironment().validateRequiredProperties()
     * 发现StandardServletEnvironment的自带的属性解析器（PropertyResolver）没有存任何的requiredProperties，所以也没有校验什么
   * 弄了个earlyApplicationEvents空在那里
   * 结束prepareRefresh
     * 几乎什么也没干成

2. obtainFreshBeanFactory()

   * 就set一下SerializationId，然后return了这个DefaultListableBeanFactory
   * 果然，到这里beanFactory只有那六个注解处理器和主类的bean

3. prepareBeanFactory(beanFactory)

   * 听上去就吊吊的，应该是一个浩大的工程

   * ```java
     protected void prepareBeanFactory(ConfigurableListableBeanFactory beanFactory) {
       // Tell the internal bean factory to use the context's class loader etc.
       beanFactory.setBeanClassLoader(getClassLoader());
       beanFactory.setBeanExpressionResolver(new StandardBeanExpressionResolver(beanFactory.getBeanClassLoader()));
       beanFactory.addPropertyEditorRegistrar(new ResourceEditorRegistrar(this, getEnvironment()));
     
       // Configure the bean factory with context callbacks.
       // 添加bean后置处理器ApplicationContextAwareProcessor,负责给那些aware塞入BeanFactory让它们感知。
       //就是ignore的这六个
       beanFactory.addBeanPostProcessor(new ApplicationContextAwareProcessor(this));
       beanFactory.ignoreDependencyInterface(EnvironmentAware.class);
       beanFactory.ignoreDependencyInterface(EmbeddedValueResolverAware.class);
       beanFactory.ignoreDependencyInterface(ResourceLoaderAware.class);
       beanFactory.ignoreDependencyInterface(ApplicationEventPublisherAware.class);
       beanFactory.ignoreDependencyInterface(MessageSourceAware.class);
       beanFactory.ignoreDependencyInterface(ApplicationContextAware.class);
     
       // BeanFactory interface not registered as resolvable type in a plain factory.
       // MessageSource registered (and found for autowiring) as a bean.
       //注册可溶解的、可分解的依赖？？
       beanFactory.registerResolvableDependency(BeanFactory.class, beanFactory);
       beanFactory.registerResolvableDependency(ResourceLoader.class, this);
       beanFactory.registerResolvableDependency(ApplicationEventPublisher.class, this);
       beanFactory.registerResolvableDependency(ApplicationContext.class, this);
     
       // Register early post-processor for detecting inner beans as ApplicationListeners.
       // 又添加了一个Bean后置处理器，负责把那些实现了ApplicationListener的bean添加到context的applicationListeners中。  
       beanFactory.addBeanPostProcessor(new ApplicationListenerDetector(this));
     
       // Detect a LoadTimeWeaver and prepare for weaving, if found.
       //检查是不是有个叫loadTimeWeaver的bean
       //这个bean是aop的bean  
       if (beanFactory.containsBean(LOAD_TIME_WEAVER_BEAN_NAME)) {
         beanFactory.addBeanPostProcessor(new LoadTimeWeaverAwareProcessor(beanFactory));
     	// Set a temporary ClassLoader for type matching.
     	beanFactory.setTempClassLoader(new ContextTypeMatchClassLoader(beanFactory.getBeanClassLoader()));
       }
     
       // Register default environment beans.
       //往beanFactory中注册和environment相关的singleton  
       if (!beanFactory.containsLocalBean(ENVIRONMENT_BEAN_NAME)) {
     	beanFactory.registerSingleton(ENVIRONMENT_BEAN_NAME, getEnvironment());
       }
       if (!beanFactory.containsLocalBean(SYSTEM_PROPERTIES_BEAN_NAME)) {
         beanFactory.registerSingleton(SYSTEM_PROPERTIES_BEAN_NAME, getEnvironment().getSystemProperties());
       }
       if (!beanFactory.containsLocalBean(SYSTEM_ENVIRONMENT_BEAN_NAME)) {
     	 beanFactory.registerSingleton(SYSTEM_ENVIRONMENT_BEAN_NAME, getEnvironment().getSystemEnvironment());
       }
     }
     ```

   【从这儿往下会有异常出现，看上去直接要把程序干死】

4. postProcessBeanFactory，后置处理BeanFactory

   ```java
   //AnnotationConfigEmbeddedWebApplicationContext	
   protected void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) {
       super.postProcessBeanFactory(beanFactory);
       if (this.basePackages != null && this.basePackages.length > 0) {
           this.scanner.scan(this.basePackages);
       }
       if (this.annotatedClasses != null && this.annotatedClasses.length > 0) {
           this.reader.register(this.annotatedClasses);
       }
   }
   ```

   ```java
   //EmbeddedWebApplicationContext，也就是super
   protected void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) {
       beanFactory.addBeanPostProcessor(
           new WebApplicationContextServletContextAwareProcessor(this));
       beanFactory.ignoreDependencyInterface(ServletContextAware.class);
   }
   ```

   * 又添加了一个bean后置处理器WebApplicationContextServletContextAwareProcessor
     * 这个处理器给那些ServletContextAware和ServletConfigAware塞对应的东西
     * 根据逻辑，先从WebApplicationContextServletContextAwareProcessor中的context拿，没有再从父类ServletContextAwareProcessor中找。
     * 因为已经处理了，所以ignore掉ServletContextAware。
   * 此时basePackages和annotatedClasses还都是null，所以它们还不能scan和register

5. invokeBeanFactoryPostProcessors

   ```java
   protected void invokeBeanFactoryPostProcessors(ConfigurableListableBeanFactory beanFactory) {
   	PostProcessorRegistrationDelegate.invokeBeanFactoryPostProcessors(beanFactory, getBeanFactoryPostProcessors());
   
   	// Detect a LoadTimeWeaver and prepare for weaving, if found in the meantime
   	// (e.g. through an @Bean method registered by ConfigurationClassPostProcessor)
   	if (beanFactory.getTempClassLoader() == null && beanFactory.containsBean(LOAD_TIME_WEAVER_BEAN_NAME)) {
   		beanFactory.addBeanPostProcessor(new LoadTimeWeaverAwareProcessor(beanFactory));
   		beanFactory.setTempClassLoader(new ContextTypeMatchClassLoader(beanFactory.getBeanClassLoader()));
   		}
   	}
   ```

   * 先去获取beanFactory后置处理器们，默认有三个
     * ConfigurationWarningsApplicationContextInitializer，没什么鸟用
     * SharedMetadataReaderFactoryContextInitializer
     * ConfigFileApplicationListener

   * 这三个beanFactory如果是BeanDefinitionRegistryPostProcessor,那么就要去执行这个的postProcessBeanDefinitionRegistry()方法。

   * 先考虑BeanDefinitionRegistryPostProcessors

     【在beanFactory是BeanDefinitionRegistry的前提下】然后去获取currentRegistryProcessors。currentRegistryProcessors是个很重要的东西。

     * 当前里面只有一个叫ConfigurationClassPostProcessor的后置处理器，这个ConfigurationClassPostProcessor有个大名鼎鼎的public方法叫processConfigBeanDefinations()，用来处理@Configuration的。
       * 先生成一个ConfigurationClassParser
         * 这个ConfigurationClassParser的parse方法，会调用processConfigurationClass()方法。
       * 然后从parser中获取到ConfigurationClasses。{可能只有ResourcePath,而beanName是null}。
       * 然后new一个ConfigurationClassBeanDefinitionReader
         * 用它去loadBeanDefinition！！！！！！！！
         * load的过程中带着很多onCondition的东西。

   * 然后考虑BeanFactoryPostProcessor

     获取BeanFactoryPostProcessor们的名字，尝试往priorityOrderedPostProcessors、orderedPostProcessorNames或者nonOrderedPostProcessorNames中添加这些名字。

6. registerBeanPostProcessors，注册bean的所有后置处理器。

   * 也是先从beanFactory中获取BeanPostProcessor的beanNames
   * 然后按照优先级注解去把它们挨个加到beanFactory的beanPostProcessors中
   * 最后，重新new 一个ApplicationListenerDetector，放到beanFactory的beanPostProcessors末尾
   * 这时候，比较重要的一些postProcessor都进来了。列举一下：
     * ApplicationContextAwareProcessor
     * WebApplicationContextServletContextAwareProcessor
     * ConfigurationClassPostProcessor
     * PostProcessorRegistrationDelegate
     * ConfigurationPropertiesBindingPostProcessor
     * MethodValidationPostProcessor
     * EmbeddedServletContainerCustmizerBeanPostProcessor
     * ErrorPageRegistrarBeanPostProcessor
     * CommonAnnotationBeanPostProcessor
     * AutowiredAnnotationBeanPostProcessor
     * RequiredAnnotationBeanPostProcessor
     * ApplicationListenerDetector

7. initMessageSource，初始化MessageSource

   * 这个比较简单，如果beanFactory中没有，则生成一个DelegatingMessageSource
   * 并作为一个单例messageSource，加到beanFactory中。

8. initApplicationEventMulticaster，初始化事件广播器

   * 这个也比较简单，如果beanFacotory中没有，则生成一个SimpleApplicationEventMulticaster
   * 并作为一个单例applicationEventMulticaster，加到beanFactory中。

9. onRefresh，重磅！

   这是重中之重，刚刚搞定所有的bean，要搭建web-server了。

   ```java
   //EmbeddedWebApplicationContext
   protected void onRefresh() {
   	super.onRefresh();
   	try {
   		createEmbeddedServletContainer();
   	}
   	catch (Throwable ex) {
   		throw new ApplicationContextException("Unable to start embedded container",
   				ex);
   	}
   }
   ```

   * super.onRefresh很简单，是GenericWebApplicationContext中的，加了个ThemeSource，用于支持原来的SpringMVC

   * 接下来就是去创建EmbeddedServletContainer了!

     * 一般来讲，会去获取其工厂，里面会尝试去beanFactory中获取EmbeddedServletContainerFactory在beanFactory的beanNames

     * 由于前面在beanNames中加入了EmbeddedServletContainerAutoConfiguration$EmbeddedTomcat

     * 所以你会发现默认启动的是Tomcat的factory。当然，引包后可能是Jetty或Undertow

     * 这里面

       ```java
       //TomcatEmbeddedServletContainerFactory
       public EmbeddedServletContainer getEmbeddedServletContainer(
       			ServletContextInitializer... initializers) {
       	Tomcat tomcat = new Tomcat();
       	File baseDir = (this.baseDirectory != null ? this.baseDirectory
       			: createTempDir("tomcat"));
       	tomcat.setBaseDir(baseDir.getAbsolutePath());
       	Connector connector = new Connector(this.protocol);
       	tomcat.getService().addConnector(connector);
       	customizeConnector(connector);
       	tomcat.setConnector(connector);
       	tomcat.getHost().setAutoDeploy(false);
       	configureEngine(tomcat.getEngine());
       	for (Connector additionalConnector : this.additionalTomcatConnectors) {
       		tomcat.getService().addConnector(additionalConnector);
       	}
       	prepareContext(tomcat.getHost(), initializers);
       	return getTomcatEmbeddedServletContainer(tomcat);
       }
       ```

       

10. registerListeners

    * 这个比较简单，就是之前第8步给beanFactory添加了个SimpleApplicationEventMulticaster，现在要把那些ApplicationListener都注册上去
      * 其中有一部分listener已经在applicationContext里了。
      * 还有一部分是要调beanFactory.getBeanNamesForType(ApplicationListener.class)，然后拿出来这些。
    * 把multicaster和listener都绑定好了，则去处理一把“EarlyEvents”，“告诉它们：我们终于有广播器了！”

11. 啊aadf 