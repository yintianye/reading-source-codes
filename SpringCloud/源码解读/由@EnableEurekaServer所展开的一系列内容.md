# 由@EnableEurekaServer所展开的EurekaServer是如何启动的

时间：2018-07-19

有这么一个情况，你要起一个EurekaServer，只要在入口处加一个@EnableEurekaServer

```java
@SpringBootApplication
@EnableEurekaServer
public class EurekaServer {
    public static void main(String[] args) {
        SpringApplication.run(EurekaServer.class);
    }
}
```

然后再在application.json或application.properties中加一些配置数据，比如

```yaml
server:
  port: 8761

eureka:
  instance:
    hostname: localhost
  client:
    registerWithEureka: false
    fetchRegistry: false
    serviceUrl:
      defaultZone: http://${eureka.instance.hostname}:${server.port}/eureka/
```

然后运行起来，EurekaServer就起来了。好随意好简单好牛逼好厉害。

这中间发生了什么？来研究一下

ps@2018-07-18： 先空开spring boot启动时对入口类的反射和遍历注解，SpringApplication.run的堆栈太深，暂时还没找到在啥地方遍历Annotation，但这玩意儿肯定是在java bean扫包之前，更在扫AutoConfiguration之前。



* 首先遍历主类时会load @EnableEurekaServer，看看这个注解：

  ```java
  @EnableDiscoveryClient
  @Target({ElementType.TYPE})
  @Retention(RetentionPolicy.RUNTIME)
  @Documented
  @Import({EurekaServerMarkerConfiguration.class})
  public @interface EnableEurekaServer {
  }
  ```

  这个注解别的东西再说。我们发现有个spring的@Import注解弄进来一个EurekaServerMarkerConfiguration，看完这个类就知道它干了件如下很无聊的事儿。

  ```java
  @Configuration
  public class EurekaServerMarkerConfiguration {
      public EurekaServerMarkerConfiguration() {
      }
  
      @Bean
      public EurekaServerMarkerConfiguration.Marker eurekaServerMarkerBean() {
          return new EurekaServerMarkerConfiguration.Marker();
      }
  
      class Marker {
          Marker() {
          }
      }
  }
  ```

  没了。弄了一顿，就只生成了一个叫Marker的bean。这个bean只有一个功能，就是被用来判断需不需要弄一个EurekaServer：

   * 如果主类上有@EnableEurekaServer，就有这个bean，判断的时候就需要弄一个EurekaServer

咋判断的？

在它的spring.factories里面配了个EnableAutoConfiguration，就是这么个bean

```java
@Configuration
@Import({EurekaServerInitializerConfiguration.class})
@ConditionalOnBean({Marker.class})
@EnableConfigurationProperties({EurekaDashboardProperties.class, InstanceRegistryProperties.class})
@PropertySource({"classpath:/eureka/server.properties"})
public class EurekaServerAutoConfiguration extends WebMvcConfigurerAdapter {
    ...
}
```

它有个@ConditionalOnBean，spring在java bean这个EurekaServerAutoConfiguration的时候，就看一下Marker.class有没有bean，没有就不弄EurekaServer。就这么判断的。



这个EurekaServerAutoConfiguration类了不得啊。

* 首先这个类捣鼓出了很多bean

* 其次最重要的，它import了EurekaServerInitializerConfiguration这个类，这个是起server的最终原因。let me see see.

  * 看看这个类的定义。

    ```java
    @Configuration
    public class EurekaServerInitializerConfiguration implements ServletContextAware, SmartLifecycle, Ordered {
        ...
            public void start() {
            (new Thread(new Runnable() {
                public void run() {
                    try {
                        EurekaServerInitializerConfiguration.
                        	this.
                        	eurekaServerBootstrap.
                        	contextInitialized(
                        		EurekaServerInitializerConfiguration.this.servletContext
                        	);
                        	
                        EurekaServerInitializerConfiguration.
                        	log.info("Started Eureka Server");
                        	
                        EurekaServerInitializerConfiguration.this.publish(
                        	new EurekaRegistryAvailableEvent(
                        		EurekaServerInitializerConfiguration.
                        		this.getEurekaServerConfig()
                        	)
                        );
                        
                        EurekaServerInitializerConfiguration.this.running = true;
                        
                        EurekaServerInitializerConfiguration.this.publish(
                        	new EurekaServerStartedEvent(
                        		EurekaServerInitializerConfiguration.
                        		this.getEurekaServerConfig()
                        	)
                        );
                    } catch (Exception var2) {
                        EurekaServerInitializerConfiguration.log.error(
                        "Could not initialize Eureka servlet context", var2);
                    }
    
                }
            })).start();
        }
        
        public boolean isAutoStartup() {
            return true;
        }
        ...
    }
    ```

  * 这个类首先会参与java bean，然后最重要的，它是一个SmartLifecycle！！！

      <u>*Spring的机制中，当所有的bean加载并初始化完毕，Spring就会回调SmartLifecycle接口的start方法来做处理（不知道是异步的还是同步的）。*</u>

  

  * 由上面的代码我们看到，start方法里面调用的是eurekaServerBootStrap的东西。而这个EurekaServerBootStrap，是spring-cloud直接照抄netflix的EurekaBootStrap！厉害了。

    <u>*这里细说一下，其实Eureka本身也是可以独立于Spring体系使用的。那时候遵循Servlet协议，使用的就是EurekaBootStrap这里面的initializeContext等方法。而这里SpringCloud这里就照抄了一番。*</u>

  

* 剩下的应该就是

  * EurekaServer其实也是一个EurekaClient，一个特殊的client
  * 依托于spring boot起的embed tomcat
  * 然后使用jersey进行http的请求了。

* 具体Eureka的策略，后面可以再详细学习一下。

