# Sleuth

* sleuth 应该是一个spring cloud的原生概念
* 基本的术语是
  * Span，基本工作单位。记录一个作用域，记录作用域的时间。
  * Trace，一系列由spans组成的树。
  * Annotation，用来及时记录一个事件的存在？



* TraceAutoConfiguration

  ```java
  	@Bean
  	@ConditionalOnMissingBean(Tracer.class)
  	public Tracer sleuthTracer(Sampler sampler, Random random,
  			SpanNamer spanNamer, SpanLogger spanLogger,
  			SpanReporter spanReporter, TraceKeys traceKeys) {
  		return new DefaultTracer(sampler, random, spanNamer, spanLogger,
  				spanReporter, this.properties.isTraceId128(), traceKeys);
  	}
  ```

  看到这里面的Sampler、Random、SpanNamer、SpanReporter都给了啥都不做的默认实现。而SpanLogger和TraceKeys在这里面就没有，需要有特定的Bean。TraceKeys就是一组ConfigurationProperties，设置一些traceKey的名字。

  唯一有一点特殊的，是SpanLogger是怎么一回事。

  

* 看起来DefaultTracer和SpanContextHolder是两个比较重要的东西

  * SpanContextHolder用来管理SpanContext的ThreadLocal。
  * 这个SpanContext实质是和Span本身有关

* Span

  这个类里面除了一些字符串名称，主要有以下比较重要的字段

  ```java
  private final long begin;
  private long end = 0;
  private final String name;
  private final long traceIdHigh;
  private final long traceId;
  private List<Long> parents = new ArrayList<>();
  private final long spanId;
  private boolean remote = false;
  private boolean exportable = true;
  private final Map<String, String> tags;
  private final String processId;
  private final Collection<Log> logs;
  private final Span savedSpan;
  @JsonIgnore
  private final Map<String,String> baggage;
  
  // Null means we don't know the start tick, so fallback to time
  @JsonIgnore
  private final Long startNanos;
  private Long durationMicros; 
  private final boolean shared;
  ```

  Span就是记录了cr、cs、sr、ss四种状态值。用来确定时间线。

* 啊啊啊