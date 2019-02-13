# Spring源码---- IOC容器的初始化过程

* 简单来说，是通过调用ConfigurableApplicationContext接口中声明的refresh()方法。
* 一个常见的refresh()方法实现是在AbstractApplicationContext抽象类中实现
* 具体来说，初始化过程包含了3个步骤
  * Resource的定位。它由ResourceLoader通过统一的Resource接口来完成。
  * BeanDefinition的载入。把用户定义好的Bean通过Resource上来，表示成IOC容器内部的数据结构，即BeanDefinition。
  * BeanDefinition向BeanFactory注册的过程。

------

1. Resource定位
   1. 像DefaultListableBeanFactory这种IOC容器不能够直接使用Resource，Spring使用BeanDefinitionReader来对这些Resource进行处理。
   2. ApplicationContext们，比如FileSystemXmlApplicationContext、ClassPathXmlApplicationContext、XmlWebApplicationContext。它们都是AbstractApplicationContext，本质上它们都是DefaultResourceLoader。
   3. ResouceLoader具备读入以Resource定义的BeanDefinition的能力。

2. BeanDefinition载入

   1. 在BeanFactory更新的过程中最终会调用applicationContext.loadBeanDefinition()方法
   2. 这个方法里会用BeanDefinitionReader去实现，比如XmlWebApplicationContext中会用临时变量refreshBeanFactory()中的XmlBeanDefinitionReader去loadBeanDefinitions(String config).
      * loadBeanDefinitions过程1，拿下Resource的InputStream
      * 准备数据（XmlBeanDefinitionReader中弄了DefaultBeanDefinitionDocumentReader和XmlReaderContext）
      * 这个DefaultBeanDefinitionDocumentReader中使用了BeanDefinitionReaderUtils.registerBeanDefinition()方法。
      * 最终回调了DefaultListableBeanFactory.registerBeanDefinition()。
3. 最终回调了DefaultListableBeanFactory.registerBeanDefinition()。似乎就是第三步了。

--------------

-------------------

那么再看一下Spring Boot有什么特色？

* 首先它的默认的ApplicationContext是AnnotationConfigEmbeddedWebApplicationContext
* 这个AnnotationConfigEmbeddeWebApplicationContext里面直接搞了个成员变量AnnotatedBeanDefinitionReader
* 这个AnnotatedBeanDefinitionReader是spring-context包里的，看来是spring3.0的那个产物。看起来它的registerBean方法似乎很有食用价值
  * 成员变量
    * ConditionEvaluator。条件注解的评估器。详情另文展开。
      * 和Condition有关
      * 有很多spring和spring boot中的注解有关
      * 和@Conditional有关
    * AnnotationScopeMetadataResolver
      * 其通过resolveScopeMetadata()方法解析注解Bean定义类的作用域元信息，即判断注册的bean是原型类型(prototype)还是单态类型(singleton)？
      * ScopeMetadata有两个成员变量
        * 一是scopeName(String)，有singleton或propotype两个值
        * 另一个是ScopedProxyMode，作用域代理模式。详情另文展开。
    * AnnotationBeanNameGenerator
      * 这个似乎比较简单，beanName的生成器。
  * 工具类
    * AnnotationConfigUtils
    * 这个类似乎很有意义啊，它里面的registerBeanDefinition()方法也最终调用了registry.registerBeanDefinition()。