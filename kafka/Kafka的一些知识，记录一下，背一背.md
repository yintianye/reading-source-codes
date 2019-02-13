# Kafka的一些知识，记录一下，背一背

## topic相关

如何查看所有的topics

```shell
.\bin\windows\kafka-topics.bat --zookeeper localhost:2181 --list
```

由命令可知，这里的“所有的topics”的意思是注册在同一个zk体系下的所有kafkaServer所持有的topic。

**** 那么问题来了，如果server1和server2都有一个topic叫做test1，那么会不会有问题？

**** 这个问题暂时留一下。