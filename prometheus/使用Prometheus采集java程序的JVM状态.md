# 使用Prometheus采集java程序的JVM状态
时间：2018-07-17

一个java进程的JVM状态使用jvm自带工具可以查到，但是其结果持久化出来给Prometheus用会比较麻烦，最好的做法是在进程内直接开exporter。现在有对于spring boot程序的办法。详解一下：

* Prometheus对spring boot有插件实现对本spring boot进程的exporter


  ```xml
  <!-- Exposition spring_boot -->
  <dependency>
      <groupId>io.prometheus</groupId>
      <artifactId>simpleclient_spring_boot</artifactId>
      <version>0.4.0</version>
  </dependency>
  
  ```

  

* 在该SpringBoot的入口类加上两个Prometheus的注解

  ```java
  @SpringBootApplication
  @EnablePrometheusEndpoint
  @EnableSpringBootMetricsCollector
  public class MyApp {
      public static void main(String[] args) {
          SpringApplication.run(MyApp.class);
      }
  }
  ```

* 在ApplicationContext里要有Prometheus需要的bean，此处写一个starter，maven引进来就行了。

  * 其中starter需要的依赖是

    ```xml
    <!-- Exposition spring_boot -->
    <dependency>
        <groupId>io.prometheus</groupId>
        <artifactId>simpleclient_spring_boot</artifactId>
        <version>0.4.0</version>
    </dependency>
    <!-- Hotspot JVM metrics -->
    <dependency>
        <groupId>io.prometheus</groupId>
        <artifactId>simpleclient_hotspot</artifactId>
        <version>0.4.0</version>
    </dependency>
    <!-- Exposition servlet -->
    <dependency>
        <groupId>io.prometheus</groupId>
        <artifactId>simpleclient_servlet</artifactId>
        <version>0.4.0</version>
    </dependency>
    ```

  * 看了一下，最少需要两个bean，一个用于构造collector，一个用于提供exporter接口。几行代码给个例子。

    ```java
    @Bean
    SpringBootMetricsCollector springBootMetricsCollector(Collection<PublicMetrics> publicMetrics) {
        SpringBootMetricsCollector springBootMetricsCollector = new SpringBootMetricsCollector(publicMetrics);
        springBootMetricsCollector.register();
        return springBootMetricsCollector;
    }
    
    @Bean
    ServletRegistrationBean servletRegistrationBean() {
        DefaultExports.initialize();
        return new ServletRegistrationBean(new MetricsServlet(), "/prometheus");
    }
    ```

    这两个bean构造的拿到的数据也不知道对不对。。。

    另外暴露的exporter的名字啥的好像也比较萎靡。

    后续继续优化重写一下。

* 我想要试一下能否给exporter自定义名字。

* D:\dev\repo\org\springframework\boot\spring-boot-actuator\1.5.13.RELEASE\spring-boot-actuator-1.5.13.RELEASE.jar!\org\springframework\boot\actuate\metrics\export\MetricExportProperties.class