# Spring Cloud在用jdk10会有JAXBContext问题

时间：20180710

## 问题现象：
配好Eureka Server时启动抛异常，尽管启动最终仍然能够完成，但实际上该spring boot进程没有监听对应的端口。

```java
java.lang.TypeNotPresentException: Type javax.xml.bind.JAXBContext not present
    at java.base/sun.reflect.generics.factory.CoreReflectionFactory.makeNamedType(CoreReflectionFactory.java:117) ~[na:na]
    at java.base/sun.reflect.generics.visitor.Reifier.visitClassTypeSignature(Reifier.java:125) ~[na:na]
    at java.base/sun.reflect.generics.tree.ClassTypeSignature.accept(ClassTypeSignature.java:49) ~[na:na]
    at java.base/sun.reflect.generics.visitor.Reifier.reifyTypeArguments(Reifier.java:68) ~[na:na]
    at java.base/sun.reflect.generics.visitor.Reifier.visitClassTypeSignature(Reifier.java:138) ~[na:na]
    at java.base/sun.reflect.generics.tree.ClassTypeSignature.accept(ClassTypeSignature.java:49) ~[na:na]
    at java.base/sun.reflect.generics.repository.ClassRepository.computeSuperInterfaces(ClassRepository.java:117) ~[na:na]
    at java.base/sun.reflect.generics.repository.ClassRepository.getSuperInterfaces(ClassRepository.java:95) ~[na:na]
    at java.base/java.lang.Class.getGenericInterfaces(Class.java:1112) ~[na:na]
    at com.sun.jersey.core.reflection.ReflectionHelper.getClass(ReflectionHelper.java:629) ~[jersey-core-1.19.1.jar:1.19.1]
    at com.sun.jersey.core.reflection.ReflectionHelper.getClass(ReflectionHelper.java:625) ~[jersey-core-1.19.1.jar:1.19.1]
    at com.sun.jersey.core.spi.factory.ContextResolverFactory.getParameterizedType(ContextResolverFactory.java:202) ~[jersey-core-1.19.1.jar:1.19.1]
    at com.sun.jersey.core.spi.factory.ContextResolverFactory.init(ContextResolverFactory.java:89) ~[jersey-core-1.19.1.jar:1.19.1]
    at com.sun.jersey.server.impl.application.WebApplicationImpl._initiate(WebApplicationImpl.java:1332) ~[jersey-server-1.19.1.jar:1.19.1]
    at com.sun.jersey.server.impl.application.WebApplicationImpl.access$700(WebApplicationImpl.java:180) ~[jersey-server-1.19.1.jar:1.19.1]
    at com.sun.jersey.server.impl.application.WebApplicationImpl$13.f(WebApplicationImpl.java:799) ~[jersey-server-1.19.1.jar:1.19.1]
    at com.sun.jersey.server.impl.application.WebApplicationImpl$13.f(WebApplicationImpl.java:795) ~[jersey-server-1.19.1.jar:1.19.1]
    at com.sun.jersey.spi.inject.Errors.processWithErrors(Errors.java:193) ~[jersey-core-1.19.1.jar:1.19.1]
    at com.sun.jersey.server.impl.application.WebApplicationImpl.initiate(WebApplicationImpl.java:795) ~[jersey-server-1.19.1.jar:1.19.1]
    at com.sun.jersey.server.impl.application.WebApplicationImpl.initiate(WebApplicationImpl.java:790) ~[jersey-server-1.19.1.jar:1.19.1]
    at com.sun.jersey.spi.container.servlet.ServletContainer.initiate(ServletContainer.java:509) ~[jersey-servlet-1.19.1.jar:1.19.1]
    at com.sun.jersey.spi.container.servlet.ServletContainer$InternalWebComponent.initiate(ServletContainer.java:339) ~[jersey-servlet-1.19.1.jar:1.19.1]
    at com.sun.jersey.spi.container.servlet.WebComponent.load(WebComponent.java:605) ~[jersey-servlet-1.19.1.jar:1.19.1]
    at com.sun.jersey.spi.container.servlet.WebComponent.init(WebComponent.java:207) ~[jersey-servlet-1.19.1.jar:1.19.1]
    at com.sun.jersey.spi.container.servlet.ServletContainer.init(ServletContainer.java:394) ~[jersey-servlet-1.19.1.jar:1.19.1]
    at com.sun.jersey.spi.container.servlet.ServletContainer.init(ServletContainer.java:744) ~[jersey-servlet-1.19.1.jar:1.19.1]
    at org.apache.catalina.core.ApplicationFilterConfig.initFilter(ApplicationFilterConfig.java:285) ~[tomcat-embed-core-8.5.27.jar:8.5.27]
    at org.apache.catalina.core.ApplicationFilterConfig.<init>(ApplicationFilterConfig.java:112) ~[tomcat-embed-core-8.5.27.jar:8.5.27]
    at org.apache.catalina.core.StandardContext.filterStart(StandardContext.java:4591) [tomcat-embed-core-8.5.27.jar:8.5.27]
    at org.apache.catalina.core.StandardContext.startInternal(StandardContext.java:5233) [tomcat-embed-core-8.5.27.jar:8.5.27]
    at org.apache.catalina.util.LifecycleBase.start(LifecycleBase.java:150) [tomcat-embed-core-8.5.27.jar:8.5.27]
    at org.apache.catalina.core.ContainerBase$StartChild.call(ContainerBase.java:1419) [tomcat-embed-core-8.5.27.jar:8.5.27]
    at org.apache.catalina.core.ContainerBase$StartChild.call(ContainerBase.java:1409) [tomcat-embed-core-8.5.27.jar:8.5.27]
    at java.base/java.util.concurrent.FutureTask.run(FutureTask.java:264) [na:na]
    at java.base/java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1167) [na:na]
    at java.base/java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:641) [na:na]
    at java.base/java.lang.Thread.run(Thread.java:844) [na:na]
Caused by: java.lang.ClassNotFoundException: javax.xml.bind.JAXBContext
    at java.base/jdk.internal.loader.BuiltinClassLoader.loadClass(BuiltinClassLoader.java:582) ~[na:na]
    at java.base/jdk.internal.loader.ClassLoaders$AppClassLoader.loadClass(ClassLoaders.java:185) ~[na:na]
    at java.base/java.lang.ClassLoader.loadClass(ClassLoader.java:496) ~[na:na]
    at java.base/java.lang.Class.forName0(Native Method) ~[na:na]
    at java.base/java.lang.Class.forName(Class.java:375) ~[na:na]
    at java.base/sun.reflect.generics.factory.CoreReflectionFactory.makeNamedType(CoreReflectionFactory.java:114) ~[na:na]
```




## 问题原因：jdk1.9之后的模块化

因为JAXB-API是java ee的一部分，在jdk9中没有在默认的类路径中；

java ee api在jdk中还是存在的，默认没有加载而已，jdk9中引入了**模块**的概念，可以使用

模块命令`--add-modules java.xml.bind`引入jaxb-api;



## 解决办法：
除了使用模块命令之外，可以添加依赖。

```xml
<dependency>
    <groupId>javax.xml.bind</groupId>
    <artifactId>jaxb-api</artifactId>
    <version>2.3.0</version>
</dependency>
<dependency>
    <groupId>com.sun.xml.bind</groupId>
    <artifactId>jaxb-impl</artifactId>
    <version>2.3.0</version>
</dependency>
<dependency>
    <groupId>org.glassfish.jaxb</groupId>
    <artifactId>jaxb-runtime</artifactId>
    <version>2.3.0</version>
</dependency>
<dependency>
    <groupId>javax.activation</groupId>
    <artifactId>activation</artifactId>
    <version>1.1.1</version>
</dependency>
```

