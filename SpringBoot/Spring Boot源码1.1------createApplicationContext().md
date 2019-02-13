# Spring Boot源码1.1------createApplicationContext()

首先，如果是web应用，则会生成AnnotationConfigEmbeddedWebApplicationContext。

* AnnotationConfigEmbeddedWebApplicationContext的成员变量有一个reader和一个scanner

  ```java
  public AnnotationConfigEmbeddedWebApplicationContext() {
      this.reader = new AnnotatedBeanDefinitionReader(this);
      this.scanner = new ClassPathBeanDefinitionScanner(this);
  }
  ```

* 这个AnnotationConfigEmbeddedWebApplicationContext的构造函数一路往上，调用了一层层父类的构造方法。

  * AnnotationConfigEmbeddeWebApplication
    * EmbeddedWebApplicationContext
      * GenericWebApplicationContext
        * GenericApplicationContext(这里面挂着DefaultListableBeanFactory，也就是调用完这些构造函数的时候，我们的ApplicationContext里面就已经有了一个空的DefaultListableBeanFactory)

* 然后再去new这两个成员变量

  * 先看AnnotatedBeanDefinitionReader

    ```java
    public class AnnotatedBeanDefinitionReader {
    
    	private final BeanDefinitionRegistry registry;
    	private BeanNameGenerator beanNameGenerator = new AnnotationBeanNameGenerator();
    	private ScopeMetadataResolver scopeMetadataResolver = new AnnotationScopeMetadataResolver();
    	private ConditionEvaluator conditionEvaluator;
        
        public AnnotatedBeanDefinitionReader(BeanDefinitionRegistry registry) {
            //这个getOrCreateEnvironment在执行时会调用所谓AnnotationConfigEmbeddedWebApplicationContext的getEnvironment()方法，
            //一开始调的时候environment是null
            //由中间的GenericWebApplicationContext的createEnvironment()实现去new了一个StandardServletEnvironment
    		this(registry, getOrCreateEnvironment(registry));
    	}
        
        public AnnotatedBeanDefinitionReader(BeanDefinitionRegistry registry, Environment environment) {
    		Assert.notNull(registry, "BeanDefinitionRegistry must not be null");
    		Assert.notNull(environment, "Environment must not be null");
    		this.registry = registry;
    		this.conditionEvaluator = new ConditionEvaluator(registry, environment, null);
            //这儿!!!会在BeanDefinitionMap里导入最初的几个BeanDefinition！
    		AnnotationConfigUtils.registerAnnotationConfigProcessors(this.registry);
    	}
        
    }
    ```

  * 关于这几个最初的BeanDefinition，需要稍微说一说

    * 其实是罗列了7个在Spring-Context4.3.17中

    * 分别是

      * org.springframework.context.annotation.internalConfigurationAnnotationProcessor

        * @Configuration的处理器

        * 真正的类是ConfigurationClassPostProcessor

        * 本质上是个BeanDefinitionRegistryPostProcessor即BeanFactoryPostProcessor

          

      * org.springframework.context.annotation.internalAutowiredAnnotationProcessor

        * @Autowired和@Value的处理器

        * 似乎和@Lookup有点关系？

        * 真正的类是AutowiredAnnotationBeanPostProcessor

        * 这玩意儿出现在spring2.5版本，所以说Autowired的注入方式是比较靠后的，但在3.0注解式之前。

        * 它本质上是一个BeanPostProcessor

          

      * org.springframework.context.annotation.internalRequiredAnnotationProcessor

        * @Required的处理器

        * 真正的类是RequiredAnnotationBeanPostProcessor

        * 这注解出现在2.0版本，说明@Required比@Autowired早啊

        * 它本质上也是一个BeanPostProcessor

          

      * org.springframework.context.annotation.internalCommonAnnotationProcessor

        * 似乎是个通用处理器，处理javax的很多注解

        * 这玩意儿号称JSR-250，需要@Resource

        * 如@PostConstruct、@PreDestroy、@Resource、@WebService等

        * 似乎也关心@Lazy注解？

        * 它本质上也是一个BeanPostProcessor

          

      * org.springframework.orm.jpa.support.PersistenceAnnotationBeanPostProcessor

        * jpa的处理器

        * 需要javax.persistence.EntityManagerFactory和org.springframework.orm.jpa.support.PersistenceAnnotationBeanPostProcessor

        * 显然这个处理JPA的那些注解

          

      * org.springframework.context.event.internalEventListenerProcessor

        * @EventListener的处理
        * 真正的类是EventListenerMethodProcessor
        * 这玩意儿，竟然不实现PostProcessor

      * org.springframework.context.event.internalEventListenerFactory

        * 这玩意儿不是个处理器
        * 只为了实例化个DefaultEventListenerFactory
        * 可能为了盛那些EventListener

  -------------------------------------------------------------------

  * 再看看ClassPathBeanDefinitionScanner

    * 这个scanner的重要作用就是使component-scan生效。

    * 或者@ComponentScan+@Configuration注解，用的ComponentScanAnnotationParser中也有ClassPathBeanDefinitionScanner

    * ```java
      public class ClassPathBeanDefinitionScanner extends ClassPathScanningCandidateComponentProvider {}
      ```

    * 这个类似乎和Bean有巨大的关系，按照注释上说的它当前处理这三类注解

      * @Component及@Service、@Repository、@Controller
      * JSR-250 中的javax.annotation.@ManagedBean
      * JSR-330 中的javax.inject.@Named

    * 别的先暂时放一下，着重看其父类ClassPathScanningCandidateComponentProvider这个类很有作用

      ```java
      public class ClassPathScanningCandidateComponentProvider extends ... {
          private final List<TypeFilter> includeFilters = new LinkedList<TypeFilter>();
          private final List<TypeFilter> excludeFilters = new LinkedList<TypeFilter>();
      }
      ```

      * 父类中有这样两个FilterList，其中includeFilters将处理上述注解的三种AnnotationTypeFilter加进来