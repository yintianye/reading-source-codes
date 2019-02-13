# prometheus的基本概念
时间：20180713

1. 其本质是一个TSDB时间序列数据库。
2. 其数据是以一种叫做metric的结构传进来的。
	* 这个metric通过名字来标记自己。
	* 里面的数据结构大概是一个List<Node(time, value)>
	* 记录的时间点应该是prometheus周期请求exporter时的时间戳
3. 通过metricName和[ ]labels可以确定一个metric中的子序列，可能是从里面抽取的。
4. 这么说来，对照我们的ratel
	* ratel中的prometheus中存有全量的metric
	* 根据配置文件，我们构造了几个按[]labels抽取的metric
   		* ratel中当前有4个metricVec，本质上是个map，key：[labels],
   		* 根据标签，我们在里面创建了和key对应的metric
   		* 其实所谓的metricName和 labels都是人为起的名字，为了方便Prometheus去获取特定的metric的。
	* Prometheus发请求从ratel中获取数据的时候，拿的是我们抽取好的metric

5. 我们看到的metric的结构大概是List<Node(time, value)>，这个value是实际的某个时刻的数据。当前共有四种类型
	* Counter 计数器
		* 顾名思义，某种东西某个时刻的值。
		* 这个值只能随时间单调递增。
		* 这个值的类型在go中是float64
		* 有Inc()和Add(float64)两种，+1或+x
	* Gauge 度量器
		* 也是某种东西某个时刻的值。
		* 但这个值随时间可增可减
		* 这个值的类型在go中也是float64
		* 有Set(float64)、Inc()、Dec()、Add(float64)、Sub(float64)几个函数
		* 还有个SetToCurrentTime()，不知道啥意思，用当前值打个时间点？
	* HistoGram 直方图
		* 是某时刻某种东西的一个向量。可以理解为某个时刻某个直方展示图中的纵坐标数据。
		* 横坐标被称为buckets，每个bucket的名字都不一样，你懂的。
		* 每个直方图都有个basename，这个name带着很多特定的exporter，比如：
			* basename_bucket{le="bucketName1"}，
				* 这个就可以构造一个counter，把时间序列的每个该bucket值累加。
				* 举例：bucketName1= 宝马，1月卖宝马10量，2月9辆，这个exporter就成了一个计数总宝马销售量的exporter。bucketName2 = 奔驰，一样的意思。
			* basename_sum
				* 记录一下当前直方图里所有值的总数，这个应该是一个gauge
				* 举例：这个月各种车的销售量之和。
			* basename_count
				* basename_count (identical to basename_bucket{le="+Inf"} above)
				* 这玩意儿猜一下的话，应该是获取一下某个bucket的gauge。
	* Summary 统计量
		* 和HistoGram一样也是一个横纵坐标数据
		* 它和HistoGram不一样的地方在于横坐标。Summary的横坐标有大小关系。
		* 所以Summary有个直接计算分位数的exporter
			* basename{quantile="0.2"}这种，应该也是个gauge