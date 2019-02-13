# Prometheus的一些问题和坑和经验

* 首先我发现prometheus所配置文件有点搞笑。在配置一个job的时候，发现label等不是【过滤条件】而是【添加或替换标签】：
  举个例子：

  ```yaml
    - job_name: 'testClient2'
      metrics_path: '/prometheus'
      static_configs:
      - targets: ['localhost:8012']
        labels:
          whatlabel: 'testPrometheus'
          area: 'aaaaa'
  ```

  如果不做任何labels设置，那么原来从localhost:8012/prometheus带来的数据只有这么一个带标签的metric

  ```
  jvm_memory_bytes_used{area="heap",instance="localhost:8012",job="testClient2"}
  ```

  现在变成了下面这样：

  ```
  jvm_memory_bytes_used{area="aaaaa",exported_area="heap",instance="localhost:8012",job="testClient2",whatlabel="testPrometheus"}
  
  和
  
  gc_g1_young_generation_time{area="aaaaa",instance="localhost:8012",job="testClient2",whatlabel="testPrometheus"}
  （对比一下其他的job）
  gc_g1_young_generation_time{instance="localhost:8998",job="testPrometheus"}
  ```

  你懂的，把从localhost:8012/prometheus拿到的所有metric的坐标全改飞了。