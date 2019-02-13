# 使用Prometheus exporter采集Kafka数据

1. 下载并安装zk和kafka，详细步骤略
2. 使用开源exporter，挂在github上，地址是https://github.com/danielqsj/kafka_exporter 
3. 下载代码至本地，注意放在GOPATH之下，使用go build进行编译。
4. 运行 .\kafka_exporter-master.exe  --kafka.server=localhost:9092
5. 搞定。