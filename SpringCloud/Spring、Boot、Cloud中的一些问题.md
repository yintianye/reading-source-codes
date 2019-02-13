# Spring、Boot、Cloud中的一些问题

1. ConfigurationProperties注解是干嘛用的？
  * 应该是这个类要java Bean的时候，其构造函数的参数从配置文件的这个坐标开始
  * 举个例子

   ```java
   @ConfigurationProperties("bee")
   public class BeeConfig implements BeanNameAware {...}
   ```

   这个的意思就是BeeConfig在javaBean的时候，从配置文件中的bee.xxxxxx开始反序列化，

   得到该bean。
2. EnableConfigurationProperties注解
  * 用法@EnableConfigurationProperties(XXXXX.class)

  * 这个注解顾名思义，enable一个ConfigurationProperties，XXXXX类型的作为参数

  * 本质上会java bean。

  * 举个例子

    ```java
    @Configuration
    @EnableConfigurationProperties(TosserSensitiveHeadersProperties.class)
    @ConditionalOnProperty(prefix = "tosser", value = "enabled", havingValue = "true", matchIfMissing = true)
    public class TosserAutoConfiguration {
        @Autowired
        private TosserSensitiveHeadersProperties tosserSensitiveHeadersProperties;
        
        ...
    }
    ```

	意思很明确，该里面的ioc，就是通过@EnableConfigurationProperties来实现java bean的。应该是在该@Configuration被ComponentScan的时候，记录在BeanFactory的ClassSet里。

