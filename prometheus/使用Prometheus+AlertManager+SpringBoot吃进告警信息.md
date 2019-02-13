# Prometheus+AlertManager+SpringBoot吃进告警信息


Prometheus可以向AlertManamger发消息，然后把告警信息送出去，送出的方式有很多种。有直接调smtp、发http的post请求等。

* prometheus首先要配置其alertmessage实例，顶级缩进下，声明alerting。

  ```yaml
  alerting:
    alertmanagers:
    - static_configs:
      - targets:
        - localhost:9093
  ```

* 然后prometheus要确认其告警规则文件的位置，顶格缩进。

  ```yaml
  rule_files:
    # - "first_rules.yml"
    # - "second_rules.yml"
    - myrules.yml
  ```
  这里面会对警告规则进行配置。简单看一下myrules.yml：

  ```yaml
  groups:
    - name: testrule
      rules:
      - alert: "AppMemoryUsage"
        expr: test_summary_count > 1750
        for: 5s
        labels:
          team: mynode
        annotations:
          summary: "High Memory usage detected"
          description: "Memory usage is above 100M"
  ```

  * groups里面可以定义很多group，这种group是给alertmanager的一个参数，代表哪个规则组。
  * 里面的rules可以定义多个单独的规则，这个东西导致alternamager往外报告警的时候，是以group粒度进行的。
  * 一个rule：
    * alert 是名字
    * expr 是真正的规则，PromQL型表达式。 例子中的test_summary_count 是个Counter型的metric
    * for  延迟触发alert，默认值是0。有何意义？？
    * labels 里面定义个label算是【路径标签】，alertmanager里面的一个分发route里面要match这些label
    * annotations 是告警消息体，其实是可以有{{$instance.name}}这种类型的变量的。需要详细看看。

* 然后就是alertmanager.yml了。

  ```yaml
  global:
    resolve_timeout: 1m
  
  route:
    receiver: webhook-all
    group_wait: 5s
    group_interval: 1m
    repeat_interval: 2m
    group_by: ['testrule']
    routes:
    - receiver: webhook1
      group_wait: 10s
      match:
        team: mynode
  
  receivers:
    - name: webhook-all
      webhook_configs:
      - url: 'http://localhost:8003/activation'
        send_resolved: true #是否确认response
    - name: webhook1
      webhook_configs:
      - url: 'http://localhost:8003/activation'
        send_resolved: true #是否确认response
  ```

  https://prometheus.io/docs/alerting/configuration/这是它的配置文件说明：

  * global应该是全局设置

  * route 是一个树形结构的根节点。满足match条件，往里走。

  * receivers是具体接受alertmanager的配置。其中这里用的webhook_configs是调http的post请求把数据发给对应的接口。

* 最后是spring boot端，spring boot吃进json格式的内容，可以直接通过如下方式来弄。

  ```java
      @RequestMapping(method = RequestMethod.POST, value = "/activation")
      public void activateAlerting(@RequestBody JSONObject jsonParam) {
          String msg = jsonParam.getJSONObject("commonAnnotations").getString("description");
          LOGGER.info("[Test me here] Message is {}", msg);
          LOGGER.info("[Test me here.] JSON object is {}", jsonParam);
  		
          //Do something for this alert.
      }
  ```

  用fastjson来弄，就可以获取到所有信息，然后做后续处理。
