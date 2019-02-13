# 使用Prometheus exporter采集Redis数据

1. 首先下载Redis，如果是在windows底下，则注意Redis官网是默认不提供windows的二进制版本的。
  * windows版本的下载地址为https://github.com/ServiceStack/redis-windows/blob/master/downloads/redis-64.3.0.503.zip ，看起来微软比较懒，都不怎么跟了。
  * 解压后，redis-server.exe就是

2. 然后下载redis的exporter，这是一个三方实现的exporter
  * 下载地址为https://github.com/oliver006/redis_exporter ，这个oliver还挺厉害。有空看看它的源码。
  * 然后go build编译出exporter的执行程序。
  * 执行该redis_exporter程序
      * 其参数列表有很多，常用参数项为
          * -web.listen-address   0.0.0.0:9121，这个代表该exporter提供metrics的端口号。
          * -redis.addr 1.1.1.1:7001, 2.2.2.2:7002，这个代表redisserver的列表。
  * 该redis_exporter的URL是:9121/metrics

3. 然后配Prometheus的prometheus.yml文件
  * 举个例子

    ```yaml
    scrape_configs:
    ...
        - job_name: redis_exporter
        	static_configs:
        	- targets: ['localhost:9121']
    ...
    ```

  * 然后启动Prometheus看看。OK了。

