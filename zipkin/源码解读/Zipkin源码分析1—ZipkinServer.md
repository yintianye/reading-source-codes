# Zipkin源码分析1—ZipkinServer

首先在spring boot的MainClass上再添加一下@EnableZipkinServer，配好spring boot的server.port，就能够起一个ZipkinServer了。一切从@EnableZipkinServer开始。

这个注解上import了那么四个configuration。这里先看ZipKinServerConfigration。

```java
@Configuration
public class ZipkinServerConfiguration {
    @Bean
    ZipkinHealthIndicator zipkinHealthIndicator(HealthAggregator healthAggregator) {
        return new ZipkinHealthIndicator(healthAggregator);
    }

    @Bean
    @ConditionalOnMissingBean({CollectorSampler.class})
    CollectorSampler traceIdSampler(@Value("${zipkin.collector.sample-rate:1.0}") float rate) {
        return CollectorSampler.create(rate);
    }

    @Bean
    @ConditionalOnMissingBean({CollectorMetrics.class})
    CollectorMetrics metrics(CounterService counterService, GaugeService gaugeService) {
        return new ActuateCollectorMetrics(counterService, gaugeService);
    }
    
    @Configuration
    @ConditionalOnProperty(
        name = {"zipkin.storage.type"},
        havingValue = "mem",
        matchIfMissing = true
    )
    @ConditionalOnMissingBean({StorageComponent.class})
    static class InMemoryConfiguration {
        InMemoryConfiguration() {
        }

        @Bean
        StorageComponent storage(@Value("${zipkin.storage.strict-trace-id:true}") boolean strictTraceId) {
            return InMemoryStorage.builder().strictTraceId(strictTraceId).build();
        }
    }
}
```

这4个bean的实例化顺序依次是：

* InMemoryStorage(StorageComponent)        storage
* ZipkinHealthIndicator                                       zipkinHealthIndicator
* CollectorSampler                                               traceIdSampler
* ActuateCollectorMetrics                                   metrics

等等，还有一个藏在和ZipkinServerConfiguration同一个包里的ZipkinHttpCollector：

* @RestController ZipkinHttpCollector              zipkinCollector

【一共这个5个重要的结构体】挨个看。

--------------------------------------



## storage

看挂在InMemoryConfiguration类上的@ConditionalXXX注解，以及zipkin-server-shared.yml文件，发现默认有个InMemoryStorage型的storage。

```java
public final class InMemoryStorage implements StorageComponent {
    final InMemorySpanStore spanStore;
    final AsyncSpanStore asyncSpanStore;
    final AsyncSpanConsumer asyncConsumer;
}
```

这个存在内存里的storage有三个东西，分别了解一下他们都是做什么的。

* spanStore。
  * 这个就是直接把Span信息存进内存的。
  * 里面有一些InMemoryStorage.MultiMap<key, Span>用来存放Span
* asyncSpanStore
  * 这个提供了一些存放时调用的接口，用来传回调方法，让你在【存储Span时】（注意，这个地方有歧义，是值从内存里传出，存到其他地方去的意思）附加一一些处理逻辑。
  * 默认实现是InternalBlockingToAsyncSpanStoreAdapter这么个类。里面有有一个实际的spanStore和一个executor，用来异步处理。看源码的意思，不是对Span本身处理而只是前后做一些不阻塞存储过程的其他活动。
* asyncConsumer【这个才是存储空间，即Store从Collector拿到数据的时候，span存入这里。】
  * 和asyncSpanStore的意义相同，这个无非是在消费SpanStore的前后，可以做一些异步处理。

由于storage我们需要配成其他能够持久化的，所以实际应用中，尽量不会用这个内存Storage。

【用到哪个就专门看看那块儿对应的源码好了， storage这里就过了。】



## zipkinHealthIndicator

这玩意儿是提供健康自检的，实现了springboot-actuate的CompositeHealthIndicator ，属于Spring Boot里面的概念。

```java
@Bean
ZipkinHealthIndicator zipkinHealthIndicator(HealthAggregator healthAggregator) {
    return new ZipkinHealthIndicator(healthAggregator);
}
```

这个zipkinHealthIndicator会把自己添加到传入的SpringBoot的healthAggregator中。

## traceIdSampler

这个就是用来控制采样的bean。

```java
@Bean
@ConditionalOnMissingBean({CollectorSampler.class})
CollectorSampler traceIdSampler(@Value("${zipkin.collector.sample-rate:1.0}") float rate) {
    return CollectorSampler.create(rate);
}

```

这个bean通过读取zipkin-server-shared.yml文件中的采样率，用来计算traceId的有效边界值。提供一个isSampled方法，用来判断对一个traceId的Span们是否需要被Collector交给Storage。

## metrics

这个ActuateCollectorMetrics metrics是collector的统计信息，把自己的信息交给Spring Boot的counter和gauge，以供Spring Boot进行诸如Prometheus等监控。

```java
ActuateCollectorMetrics(CounterService counterService, GaugeService gaugeService) {
    this(counterService, gaugeService, (String)null);
}
```

## zipkinCollector

最后是接收trace信息的collector，由这个zipkinCollector持有。

```java
@RestController
@CrossOrigin({"${zipkin.query.allowed-origins:*}"})
public class ZipkinHttpCollector {
	。。。
    static final String APPLICATION_THRIFT = "application/x-thrift";
    final CollectorMetrics metrics;
    final Collector collector;//持有的那个collector

    @Autowired
    ZipkinHttpCollector(StorageComponent storage, CollectorSampler sampler, CollectorMetrics metrics) {
        this.metrics = metrics.forTransport("http");
        this.collector = Collector.builder(this.getClass()).storage(storage).sampler(sampler).metrics(this.metrics).build();//把之前的storage、traceIdSampler、metrics三个bean的引用都交给这个zipkinCollector
    }

    
    ////此zipkinCollector共支持json和thrift格式的数据。
    //拿到数据就去调用validateAndStoreSpans方法。
    @RequestMapping(value = {"/api/v1/spans"},method = {RequestMethod.POST})
    .. uploadSpansJson(...) {
        return this.validateAndStoreSpans(encoding, Codec.JSON, body);
    }
    
    @RequestMapping(value = {"/api/v1/spans"},method = {RequestMethod.POST},
        consumes = {"application/x-thrift"}
    )
    .. uploadSpansThrift(...) {
        return this.validateAndStoreSpans(encoding, Codec.THRIFT, body);
    }

   
}

```

看一下这个validateAndStoreSpans方法

```java
 ListenableFuture<ResponseEntity<?>> validateAndStoreSpans(String encoding, Codec codec, byte[] body) {
        final SettableListenableFuture<ResponseEntity<?>> result = new SettableListenableFuture();
        this.metrics.incrementMessages();//1、把metrics统计数据更新一下
		。。。

        //2、让collector去接请求的东西，然后回调
        this.collector.acceptSpans(body, codec, new Callback<Void>() {
            public void onSuccess(@Nullable Void value) {
                result.set(ZipkinHttpCollector.SUCCESS);
            }

            public void onError(Throwable t) { }
        });
        。。。
        //3、返回结果给HttpResponse
        return result;
    }
```

可见这个collector.acceptSpans是最重要的流程。进Collector看看：

```java
class Collector {
    acceptSpancs() {
        //1.反序列化List<byte[]>得到List<Span>
        //2.
        this.accept();
    }
    
    accept(List<Span> spans, Callback<Void> callback) {
        ...
        //看有哪些是真的需要被采样的
        List<Span> sampled = this.sampled(spans);
        //！！！！！！最为重要，把采样的List<Span>放进asyncSpanConsumer里。
        //其中内存式的storage是放在分开的好几个MultiMap里
        this.storage.asyncSpanConsumer().accept(sampled,this.acceptSpansCallback(spans));
    }
}
```

