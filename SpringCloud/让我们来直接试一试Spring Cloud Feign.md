# 让我们来直接试一试Spring Cloud Feign

如果你的spring boot程序要调用别人的REST接口时，需要发HTTP请求。
发HTTP请求的形式有很多种，但是spring cloud feign提供了一种各微服务之间相互调用的简单做法，借助于eureka等服务发现机制，可以把调用封装成接口函数，写代码时就会简单很多。

* 首先，需要如下maven依赖

  ```xml
  <dependency>
      <groupId>org.springframework.cloud</groupId>
      <artifactId>spring-cloud-starter-feign</artifactId>
  </dependency>
  ```

  在需要调HTTP请求的微服务加入该feign-starter，然后在主类添加注解

  ```java
  @EnableFeignClients
  public class FeignProvider {
  	...
  }
  ```

  当然，还要配Eureka那一堆配置，要加eureka的注解。

* 然后，声明远端RestApi对应的接口

  举个例子

  ```java
  @FeignClient("feignproducer")
  public interface FeignProducer {
      @RequestMapping("/name")
      String getProducerName();
  }
  ```

  这里的"feignproducer“就是远端的微服务的application_name，对应的url就被映射成方法。

* 最后，调用的地方注入该接口，调用方法就行。

  ```java
  @Autowired
  FeignProducer feignProducer;
  
  @RequestMapping("/name")
  public String getName() {
      return feignProducer.getProducerName();
  }
  ```

  

* 附录：各种参数如何传递