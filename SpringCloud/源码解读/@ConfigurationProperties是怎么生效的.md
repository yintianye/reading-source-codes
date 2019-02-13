# @ConfigurationProperties是怎么生效的

在spring boot中，有一些配置项放在了application.properties或者是application.yaml中，这些配置是可以被反序列化成bean的。spring是怎么把这些东西反序列化的叻？研究一下。

* 首先举个例子

  ```java
  bee:
    enabled: true
    application-name: ${spring.application.name}
    appenders:
     #
    skiplist:
      #
    message:
      defaults:
        #
      custom:
        #
  ```

  这个玩意儿想要被反序列化成这么个类的实例

  ```
  @ConfigurationProperties("bee")
  public class BeeConfig implements BeanNameAware {
      ...
      private AppendersConfig appenders;
      private List<SkiplistRule> skiplist = new ArrayList();
      ...
  }
  ```

  

* spring3.0之后，用注解进行java bean也可以，要挂@Bean @Component等。

  使@ConfigurationProperties生效的方式有这么几种

  * 在某个bean类上，使用@EnableConfigurationProperties(BeeConfig.class)
  * 对于BeeConfig对应的@Bean方法
    * 如果类上没写@ConfigurationProperties，添加在@Bean方法上也生效
    * 否则类上写了@ConfigurationProperties，就能生成配置好的bean。

* 咋搞定的呢？

  当这个类被注入到其他类时，会触发该类的类加载，在加载过程中，会调用AbstractAutowireCapableBeanFactory#applyBeanPostProcessorsBeforeInitialization方法。

  ```java
  public Object applyBeanPostProcessorsBeforeInitialization(Object existingBean, String beanName) throws BeansException {
          Object result = existingBean;
          Iterator var4 = this.getBeanPostProcessors().iterator();
  
          do {
              if (!var4.hasNext()) {
                  return result;
              }
  
              BeanPostProcessor beanProcessor = (BeanPostProcessor)var4.next();
              result = beanProcessor.postProcessBeforeInitialization(result, beanName);
          } while(result != null);
  
          return result;
      }
  ```

  这会导致触发CongratulationPropertiesBindingPostProcessor#postProcessBeforeInitialization。

  ```java
      public Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException {
          //先看这个类上有没有@ConfigurationProperties注解
          ConfigurationProperties annotation = (ConfigurationProperties)AnnotationUtils.findAnnotation(bean.getClass(), ConfigurationProperties.class);
          if (annotation != null) {
              this.postProcessBeforeInitialization(bean, beanName, annotation);
          }
  		//再看这个bean的工厂方法上有没有@ConfigurationProperties注解
          annotation = (ConfigurationProperties)this.beans.findFactoryAnnotation(beanName, ConfigurationProperties.class);
          if (annotation != null) {
              this.postProcessBeforeInitialization(bean, beanName, annotation);
          }
  
          return bean;
      }
  ```

  如果有@ConfigurationProperties注解，最终会调用三个参数的postProcessBeforeInitialization方法

  ```java
      private void postProcessBeforeInitialization(Object bean, String beanName, ConfigurationProperties annotation) {
          //把我们的bean挂在factory里
          PropertiesConfigurationFactory<Object> factory = new PropertiesConfigurationFactory(bean);
          //把property源交给factory
          factory.setPropertySources(this.propertySources);
          factory.setValidator(this.determineValidator(bean));
          factory.setConversionService(this.conversionService == null ? this.getDefaultConversionService() : this.conversionService);
          if (annotation != null) {
              factory.setIgnoreInvalidFields(annotation.ignoreInvalidFields());
              factory.setIgnoreUnknownFields(annotation.ignoreUnknownFields());
              factory.setExceptionIfInvalid(annotation.exceptionIfInvalid());
              factory.setIgnoreNestedProperties(annotation.ignoreNestedProperties());
              //从annotation中获取bean对应的property字段
              if (StringUtils.hasLength(annotation.prefix())) {
                  factory.setTargetName(annotation.prefix());
              }
          }
  
          try {
              //把property反序列化给bean
              factory.bindPropertiesToTarget();
          } catch (Exception var8) {
              String targetClass = ClassUtils.getShortName(bean.getClass());
              throw new BeanCreationException(beanName, "Could not bind properties to " + targetClass + " (" + this.getAnnotationDetails(annotation) + ")", var8);
          }
      }
  ```

  

