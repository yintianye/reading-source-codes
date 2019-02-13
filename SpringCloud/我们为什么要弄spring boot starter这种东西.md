# 我们有一些spring boot starter

1. 它本质上像插件那样，是程序中的一块小功能

2. 它是spring boot的约定
3. 原理如下：
  * SpringBoot的AutoConfiguration机制，SpringBoot启动时，会去扫描classpath中所有jar包，看看是否有META-INF/spring.factories文件。读取org.springframework.boot.autoconfigure.EnableAutoConfiguration所制定的Configuration，根据该Configuration上面的@Conditional自动创建Bean。

  * Spring（Spring Boot？）自动配置的starter及自动配置能力由如下包提供：
   ```xml
   <dependency>  
       <groupId>org.springframework.boot</groupId>  
       <artifactId>spring-boot-autoconfigure</artifactId>  
       <version>1.5.9.RELEASE</version>  
   </dependency>  
   ```

4. 需要理解一下Spring的@Configuration。

## 我们的bee-starter：

1. 本质上是给其他spring boot程序配apollo的。
2. 遵循的starter的约定，入口是BeeAutoConfiguration类。
   * 该类里主要定义了4个bean：
     * 一个BeeConfig，从配置文件里反序列化得到的
     * 一个BeeFilter（BeeConfig， Tracer），用BeeConfig去配的BeeFilter
       * 这个Filter里，最主要的持有一个有BeeConfig生成的SkipListApplyController。这个Controller里记录了3个列表，skipListRules、reqBodySkipListRules、respBodySkipListRules，都是直接从BeeConfig里拿的，可见都是配置文件里配好的。
     * 一个SpringBoot的FilterRegistrationBean，看来我们的配置是被spring boot这么吃进去的。
     * 一个ApolloListenerInitializer，是个自己实现的类。在java bean的new完了之后，自己调用了一把所谓的initialize方法，然后把这个instance返回回去。
       * 感觉这玩意儿干了这么个事儿，这个initialize方法里面调用了一下Apollo的Config的addChangeListener，这里在未知位置起了一个循环的线程。如果监控的文件发生了改变，就回调传入的方法，拿取最新配置进行处理。

