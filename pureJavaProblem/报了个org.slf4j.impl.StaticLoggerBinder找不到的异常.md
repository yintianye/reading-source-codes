# 报了个org.slf4j.impl.StaticLoggerBinder找不到的异常

时间：2018-07-11



现象：在用java写kafka的producer时，使用LOGGER时包上述问题，导致日志打不出来。

原因：引入了kafka-clients v1.1.0包，顺带着进来了slf4j-api v1.7.25，但没有其他的。这是因为缺少了slf4j-log4j的联合包，也没有log4j这个包。所以只有门面slf4j没有logger的实现类。

解决办法：添加对应的依赖。其中，要添加slf4j-log4j12的依赖，注意是12.

```xml
<dependency>
    <groupId>org.slf4j</groupId>
    <artifactId>slf4j-log4j12</artifactId>
    <version>1.7.25</version>
</dependency>
```

其会将log4j的1.2.17版本的jar包也拉下来。



这还没完，因为kafka不像spring boot嵌入得那么好，所以要配log4j.properties

找了一个配置文件，就可以了。

```properties
log4j.rootLogger=debug, stdout, R

log4j.appender.stdout=org.apache.log4j.ConsoleAppender
log4j.appender.stdout.layout=org.apache.log4j.PatternLayout

log4j.appender.stdout.layout.ConversionPattern=%5p - %m%n

log4j.appender.R=org.apache.log4j.RollingFileAppender
log4j.appender.R.File=firestorm.log

log4j.appender.R.MaxFileSize=100KB
log4j.appender.R.MaxBackupIndex=1

log4j.appender.R.layout=org.apache.log4j.PatternLayout
log4j.appender.R.layout.ConversionPattern=%p %t %c - %m%n

log4j.logger.com.codefutures=DEBUG
```



