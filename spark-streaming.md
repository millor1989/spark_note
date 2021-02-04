Spark Streaming是Spark API的扩展，对实时数据流（live data streams）进行可扩展的（scalable），高吞吐（high-throughput），容错的（fault-tolerant）流式处理（stream processing）。可以对Kafka，Flume，Kinesis，或者TCP sockets的数据进行处理，并且可以通过像`map`，`reduce`，`window`这样的高级函数用复杂的算法进行处理。处理后的数据可以保存到文件系统、数据库、和动态看板（live dashboards）。事实上，可以将Spark的机器学习和图运算算法运用到数据流上。

Spark Streaming工作原理如下，接收输入数据流，把数据分为批次，使用Spark引擎对输入批次进行处理，结果也是按批次产生。

![Spark Streaming](/assets/streaming-flow.png)

Spark Streaming提供了高级别的抽象叫作离散流（discretized stream）或者DStream，它代表一个连续的数据流。可以使用Kafka、Flume、或Kinesis数据源的数据流来创建DStreams，也可以用其它DStreams的高级操作生成DStreams。在内部，DStream代表一个RDDs的序列。

##### 例子

计算TCP socket的text数据流中的单词个数：

```scala
import org.apache.spark._
import org.apache.spark.streaming._
import org.apache.spark.streaming.StreamingContext._ // not necessary since Spark 1.3

// Create a local StreamingContext with two working thread and batch interval of 1 second.
// The master requires 2 cores to prevent from a starvation scenario.

val conf = new SparkConf().setMaster("local[2]").setAppName("NetworkWordCount")
val ssc = new StreamingContext(conf, Seconds(1))
```

`StreamingContext`是Spark Streaming所有功能的入口

```scala
// Create a DStream that will connect to hostname:port, like localhost:9999
val lines = ssc.socketTextStream("localhost", 9999)
// Split each line into words
val words = lines.flatMap(_.split(" "))

import org.apache.spark.streaming.StreamingContext._ // not necessary since Spark 1.3
// Count each word in each batch
val pairs = words.map(word => (word, 1))
val wordCounts = pairs.reduceByKey(_ + _)

// Print the first ten elements of each RDD generated in this DStream to the console
wordCounts.print()
```

执行以上代码时，Spark Streaming只是配置了它启动后需要执行的运算，真正的运算并没有开始。需要执行如下代码，启动运算：

```scala
ssc.start()             // Start the computation
ssc.awaitTermination()  // Wait for the computation to terminate
```

这个程序启动后，每秒都会接收端口的输入，并输出一次结果。



