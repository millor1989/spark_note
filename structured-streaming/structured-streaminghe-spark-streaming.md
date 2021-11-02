流式计算有两种模式：continuous-based和micro-batch。Spark Streaming是基于micro-batch的。

Structured Streamin最初只支持micro-batch模式，从Spark 2.3开始增加了试验性的continuous processing（continuous-based）模式。基于micro-batch模式的流处理并不是真正意义上的实时流处理。micro-batch模式的延时是~100ms。

Spark Streaming的直接运算引擎是Spark Core，而Structured Streaming的直接计算引擎是Spark SQL。Spark SQL虽然是基于Spark Core的，但是在计算之前会有一系列的优化，性能可能会有提升。



**Spark Streaming的不足**：

- **Spark Streaming使用的是Processing Time，而不是Event time**。Spark Streaming基于DStream模型，DStream是Spark Streaming基于Processing Time切割的，导致使用Event Time特别困难。
- **复杂的低级别的API**。DStream API与RDD API类似，基于Spark Core。（Spark Streaming是***Imperative***命令式的，而Structured Streaming是***Declarative***声明式的）
- **推导端到端仅仅一次的应用**。DStream只能保证自己的端到端仅仅一次的一致性语义，从输入到Spark Streaming和Spark Streaming到外部接收器的语义需要用户自己保证，对用户来说是困难的。
- streaming/batch代码不统一。DStream虽然是对RDD的封装，但是将DStream完全转换为RDD还是需要工作量的。另外Spark批处理都转向DataFrame/Dataset了。



**Structured Streaming优点**：

- 增量查询模型：Structured Streaming在行增的流数据上执行增量查询，流处理代码和Spark批处理API（DataFrame/Dataset）统一。
- 保证端到端仅仅一次的应用
- 使用Spark SQL引擎

Structured Streaming的continuous processing模式可以达到~1ms级的延时，但是只提供了端到端精确一次的容错保证。

