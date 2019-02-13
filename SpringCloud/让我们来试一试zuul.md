# 让我们来试一试zuul
时间：2018-07-19
zuul是spring cloud引入的一个gateway，原本是Netflix自己写的servlet，当前被spring cloud整合了。

zuul号称有这么多功能：

- 认证
- 洞察
- 压力测试
- 金丝雀测试
- 动态路由
- 服务迁移
- 负载脱落
- 安全
- 静态响应处理
- 主动/主动流量管理

## 第0个功能：转发请求



* 一开始的配置如下。

  ```yaml
  spring:
    application:
      name: springcloud-zuul
  
  server:
    port: 8040
  
  eureka:
    client:
      serviceUrl:
        defaultZone: http://localhost:8761/eureka/
  
  zuul:
    routes:
      route1:
        path: /client2/**
        serviceId: client22222
        #url: http://localhost:8012
  ```

  解释一下：
  * 它的名字
  * 它的端口号
  * 它如果想要用服务发现serviceId来访问，挂在eureka或consul上。
  * zuul自己的配置，必不可少的就是routes。

* 运行起来之后，访问localhost:8040/client2/hello时，报了个500错误。一看zuul那里抛了异常。

```
com.netflix.zuul.exception.ZuulException: Forwarding error
......
Caused by: com.netflix.client.ClientException: Load balancer does not have available server for client: client22222
	at com.netflix.loadbalancer.LoadBalancerContext.getServerFromLoadBalancer(LoadBalancerContext.java:483) ~
```

​	大概的意思时，lb不知道client名字到底是哪个服务。愚蠢的默认设计，zuul都知道去问eureka，你下面的lb不知道，，给跪下了。

* 在zuul的application.yaml里加如下的配置

  ```yaml
  ribbon:
    eureka:
      enabled: true
  ```

* 然后再起。再访问报404，

  * 我的rest接口的url是"/client2/hello"
  * 试试"localhost:8040/client2/client2/hello"
  * 成功了！可见zuul转发时会吃掉转发规则的前缀。

* 如此说来，转发报文这个能力大概可以如此实现了。



## 第1个功能：负载均衡

刚刚的那个配置里面，数据转发时做了定向的路由。如果想要load balance呢？试一试。

把zuul的application.yaml改了

```yaml
zuul:
  routes:
    route2:
      path: /testzuul/**
      serviceId: lbusers

ribbon:
  eureka:
    enabled: false

lbusers:
  ribbon:
    listOfServers: localhost:8021, localhost:8022
```

如上，关了ribbon的eureka，让ribbon自己控制serviceId的指向。开两个普通的Springboot进程，都提供"/hello"接口。启动后调用统一网关localhost:8040/testzuul/hello，发现果然load balance了，而且这个负载均衡算法在两个节点时，一人一个来回换。

## 第2个功能：熔断保护

听说你能熔断？

spring cloud中的Zuul已经整合了Hystrix。想要为Zuul添加回退，需要实现ZuulFallbackProvider接口。

* 弄个可以java bean的类，实现这个接口。

  ```java
  @Component
  public class FallBackOnZuul implements ZuulFallbackProvider {
      @Override
      public String getRoute() {
          return "lbusers";
      }
  
      @Override
      public ClientHttpResponse fallbackResponse() {
          return new ClientHttpResponse() {
  
              @Override
              public HttpHeaders getHeaders() {
                  // 设置header
                  HttpHeaders headers = new HttpHeaders();
                  MediaType mediaType = new MediaType("application",
                          "json",Charset.forName("UTF-8"));
                  headers.setContentType(mediaType);
  
                  return headers;
              }
  
              @Override
              public InputStream getBody() throws IOException {
                  // 响应体
                  return new ByteArrayInputStream(
                          "Usermicro-service is unavailable, please try it again later!".getBytes());
              }
  
              @Override
              public String getStatusText() throws IOException {
                  // 返回状态文本
                  return this.getStatusCode().getReasonPhrase();
              }
  
              @Override
              public HttpStatus getStatusCode() throws IOException {
                  return HttpStatus.OK;
              }
  
              @Override
              public int getRawStatusCode() throws IOException {
                  // 返回数字类型的状态码
                  return this.getStatusCode().value();
              }
  
              @Override
              public void close() {
              }
          };
      }
  }
  ```

  最重要的是，要重写其getRoute方法，配置好到底这个fallbackprovider是替配置文件中哪个route对应的微服务做的回调。例子中是用ribbon的名字。然后我ribbon lb的两个节点都不启动，果然就被这里fallback了。

  这是服务不可用时，Zuul给出的默认返回值。

* 如果服务可用，但是接口超时了呢？

  有这么几个参数

    ```java
    ribbon:
      ReadTimeout: 10000
      ConnectTimeout: 10000
  
    hystrix:
      command:
        default:
          execution:
            isolation:
              thread:
                timeoutInMilliseconds: 9000
  
    ```

  试了一下，

  * 我所希望的“处理超时”，对应的是ribbon.ReadTimeout
  * 我猜：如果zuul知道节点alive，但是连接不上，应该是对应的ribbon.ConnectTimeout
  * 至于这个hystrix的配置，暂时还不知道是个啥玩意儿。不好猜。

## 第3个功能：来个高可用Zuul集群

好像这玩意儿挺鸡肋的。要在外面挂nginx或者F5。这么说的话，zuul的作用就是替了一下nginx的lua脚本。