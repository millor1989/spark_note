#### 1、[概览](http://spark.apache.org/docs/2.2.0/structured-streaming-programming-guide.html#overview)

Structured Streaming是一个基于Spark SQL引擎构建的，可扩展的并且容错的流处理引擎。可以用与基于静态数据的批次运算相同的方式表示流运算。随着流数据的到达，Spark SQL引擎负责渐进地、持续地更行最终结果。可以使用Dataset、DataFrame API来表示流的聚合、事件时间窗口（event-time windows）、流到批次连接（stream-to batch joins），等。运算是基于相同的优化的Spark SQL引擎的。系统通过使用checkpointing和*Write Ahead Logs*来确保端到端的仅仅一次的容错保证。

简而言之，Structured Streaming提供了快速的、可扩展的、容错的、端到端的仅仅一次的流处理，而不必推导流。

#### 2、例子

```scala
import org.apache.spark.sql.functions._
import org.apache.spark.sql.SparkSession

val spark = SparkSession
  .builder
  .appName("StructuredNetworkWordCount")
  .getOrCreate()
  
import spark.implicits._

// Create DataFrame representing the stream of input lines from connection to localhost:9999
val lines = spark.readStream
  .format("socket")
  .option("host", "localhost")
  .option("port", 9999)
  .load()

// Split the lines into words
val words = lines.as[String].flatMap(_.split(" "))

// Generate running word count
val wordCounts = words.groupBy("value").count()

// Start running the query that prints the running counts to the console
val query = wordCounts.writeStream
  .outputMode("complete")
  .format("console")
  .start()

query.awaitTermination()
```

其中DataFrame`lines`代表一个包含流式文本数据的无边界表（unbounded table）。使用`.as[String]`将DataFrame转换为String的Dataset。使用`outputMode("complete")`，在每次更新流数据时，把完整的结果输出到控制台。使用`start()`启动流应用。代码执行后，流运算会在后台启动。`query`对象是活跃的流查询的句柄，用它调用`awaitTermination()`来防止在查询活跃时进程退出。

#### 3、编程模型

Structured Streaming的主要理念是把实时数据流当作一个持续地追加数据的表。这是一个与批次处理模型非常相似的流处理模型。可以将流运算表达为，像基于一个静态表的标准的类批次查询（standard batch-like query），Spark会运行作一个基于无边界输入表的增加的（incremental）查询。

##### 3.1、基本概念

把输入数据流看作“输入表”。流中的每条数据就像是追加到输入表的一个新的行。

![Stream as a Table](/assets/structured-streaming-stream-as-a-table.png)

对输入的查询会生成“结果表”。每个触发间隔（比如，每1秒），新的行追加到输入表，最终会更新结果表。每当结果表更新后，则把改变的行写到一个外部接收器（sink）。

![Model](/assets/structured-streaming-model.png)

输出“Output”即写到外部存储系统的数据。输出模式：

- Complete Mode：将整个更新后的结果表写到外部系统。根据存储连接器（storage connector）来决定如何处理整个表的写出。
- Append Mode：只有最近一次触发的追加到结果表的新行才会写到外部系统。仅适用于结果表中已经存在的行不改变（not expected to change）的查询。
- Update Mode：只有最近一次触发的结果表中被更新的行才会写到外部系统（从Spark 2.1.1开始可用）。这种模式只输出最近一次触发的改变的行。如果查询不包含聚合，那么就和Append Mode等价。

每个模式只适用于某些特定类型的查询。

用之前的例子阐释Structured Streaming模型，`lines`DataFrame是输入表，`wordCounts`DataFrame是结果表。基于流DataFrame`lines`产生`wordCounts`的查询与基于静态DataFrame的查询是完全相同的。但是，查询启动后，Spark会持续地从socket连接检查新的数据。如果有新的数据，Spark会运行一个“增量”的（incremental）查询，将之前运行的计数结果和新的数据结合来计算更新后的计数结果。

![Model](/assets/structured-streaming-example-model.png)

这个模型与许多其它流处理引擎有显著地不同。许多流系统需要用户自己维持运行的流数据聚合（maintain running aggregations），因而需要推导容错、数据一致性（至少一次、最多一次、仅仅一次）。而这个模型中，Spark负责有新的数据时更新结果表，用户不用再进行复杂的推导。

#### 4、处理Event-time和Late Data（迟到数据）

事件时间（event-time）是指时间本身包含在数据中。有许多的应用，可能都要基于事件时间进行操作。比如，如果要统计每分钟IoT设备产生的时间数量，那么就需要使用数据产生的时间（即，数据中的时间），而不是Spark收到数据的时间。事件时间在这个模型中表现得非常自然——设备的每个事件都是表中的一行，事件时间则是行的一个字段值。这就使基于窗口的聚合（比如，每分钟的事件数）变成仅仅是对事件时间字段的grouping和aggregation的一种特例——每个事件窗口是一个组并且每行可以属于多个窗口/组。

另外，这个模型还可以基于事件时间很自然地处理迟到的数据。因为Spark是在更新结果表，当有迟到的数据时，它对更新旧的聚合具有完全的控制，也能清除旧的聚合来限制中间状态数据的大小。从Spark  2.1开始支持水印（watermarking），用户可以设置新的数据的阈值，让引擎可以根据情况清除旧的状态。

#### 5、容错语义

达到端到端仅仅一次的语义是Structured Streaming设计的一个关键目标。为了达到这个目的，设计了Structured Streaming的数据源、外部接收器（sink）和执行引擎，以可靠的追踪处理过程的详细进度，以便可以处理重启或/和重新处理中的任何故障。每个流数据源都被假设为是有偏移量（类似于Kafka偏移量，或者Kinesis sequence numbers）可以用来追踪流的读取位置。引擎使用checkpointing和write ahead logs来记录每次触发中被处理的数据的偏移量。流的外部接收器被设计为幂等性的以处理重新处理的情况（handling reprocessing）。使用可以重播的（repalyable）数据源和幂等的外部接收器，Structured Streaming可以确保在任何故障中都能够是端到端仅仅一次的语义。

