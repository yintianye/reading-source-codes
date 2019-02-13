# zipkin如何玩儿

  ## java服务作为zipkin client

1. springboot需要配spring-cloud-starter-zipkin的maven依赖。

2. 需要配置application.yaml文件

   ```yaml
   application:
   	zipkin:
   		base-url: http://localhost:9411
   ```

   用于向zipkin发送消息。

## 用java来起一个zipkin server

也可以使用spring boot直接起zipkin server，做法如下。

1. 添加maven的依赖

   ```xml
   <dependency>
       <groupId>io.zipkin.java</groupId>
       <artifactId>zipkin-server</artifactId>
   </dependency>
   <dependency>
       <groupId>io.zipkin.java</groupId>
       <artifactId>zipkin-autoconfigure-ui</artifactId>
   </dependency>
   
   ```

   

2. 添加注解

   ```
   @EnableZipkinServer
   @SpringBootApplication
   public class MyZipkinServer {
   	...
   }
   ```

3. 运行，并访问localhost:port，应该就能够看到zipkin的server了。

4. 这个@EnableZipkinServer，所在的包没有spring.factories文件，不知道咋起来的，而且这里面有个main函数。我怀疑是SpringApplication.run()函数里对main重定向了。

   * 验证了一下，没有起这个main函数。

   * 没有spring.factories，走的不是autoConfiguration，而是Import了class然后spring bean。

     ```java
     
     ```

     import的类是@Configuration，里面包着很多bean。 

     ```java
     
     ```

     

   