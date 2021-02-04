#### 1、依赖

Spark Streaming依赖：

```
<dependency>
    <groupId>org.apache.spark</groupId>
    <artifactId>spark-streaming_2.11</artifactId>
    <version>2.2.0</version>
</dependency>
```

不同的Spark Streaming输入数据源，需要不同的jar依赖：

| Source  | Artifact                                                   |
| ------- | ---------------------------------------------------------- |
| Kafka   | spark-streaming-kafka-0-8_2.11                             |
| Flume   | spark-streaming-flume_2.11                                 |
| Kinesis | spark-streaming-kinesis-asl_2.11 [Amazon Software License] |

#### 2、`StreamingContext`初始化

使用Spark Streaming必须初始化`StreamingContext`对象。可以使用`SparkConf`对象来初始化`StreamingContext`（内部地，会创建一个`SparkContext`，可以用`ssc.sparkcontext`访问），也可以用既存的`SparkContext`：

```scala
val conf = new SparkConf().setAppName(appName).setMaster(master)
val ssc = new StreamingContext(conf, Seconds(1))

val sc = ...                // existing SparkContext
val ssc = new StreamingContext(sc, Seconds(1))
```

批次间隔时间的设置要根据应用的需要和集群资源而定。

`StreamingContext`初始化完成后，可以：

- 通过创建输入DStreams定义输入数据源
- 通过应用转换操作和DStreams的输出操作定义流的运算
- 使用`streamingContext.start()`开始接收数据并处理
- 使用`streamingContext.awaitTermination()`等待处理结束（手动结束或者错误地终止）
- 使用`streamingContext.stop()`手动结束应用

需要注意的是：

- 启动`StreamingContext`后，不能设置或添加新的流运算
- 停止`StreamingContext`后，不能重启
- 一个JVM同时只能有一个活跃的`StreamingContext`
- `streamingContext.stop()`会同时停止`SparkContext`，但是可以使用重载的`stop()`方法只停止`StreamingContext`
- 可以重用`SparkContext`来创建新的`StreamingContext`，但是要保证新的创建之前，老的`StreamingContext`已经停止。

#### 3、离散化的流（DStreams）

DStream是Spark Streaming的基本抽象概念，它表示一组连续的流数据，而不管数据从何而来。在内部，DStream由一组连续的RDDs表示，每个RDD包含特定时间间隔内的数据：

![Spark Streaming](/assets/streaming-dstream.png)

对DStream的所有操作都会被转换为对底层RDD的操作，底层RDD的运算由Spark引擎执行。

#### 4、输入DStreams和Receivers

输入DStream是代表来自流数据源的输入数据流的Dstream。每个输入DStream（除了文件流）都是与一个`Receiver`对象相关联的，`Receiver`从数据源接收数据并保存到Spark的内存中。

Spark Streaming提供两类自带（built-in）流数据源：

- 基本数据源（basic sources）：StreamingContext API直接可用的数据源，比如：文件系统，socket连接。
- 高级数据源（advanced sources）：需要通过外部工具类进行连接的数据源，比如Kafka、Flume、Kinesis等等。

如果要在流应用中并行接收多个数据流，可以创建多个输入DStreams。多个DStreams，意味着多个Receivers，它们会同时接收多个数据流。但是，Spark的worker/executor是一个长时间运行的任务，因此会占用一个分配给Spark Streaming应用的核心。因此，要分配给Spark Streaming应用足够的核心。需要注意的是：

- 本地运行Spark Streaming程序时，不要使用”local“或者”local[1]“作为master URL。这两种master意味着本地运行任务时只会使用一个线程。如果使用的是基于receiver的输入DStream，仅有的线程会被用于运行receiver，就没有线程可以用来处理接收到的数据。因此，本地运行时设置的线程必须多于receivers的数量。
- 同上，在集群上运行时，分配给Spark Streaming应用的核心数必须大于receivers的数量。否则，应用只能接收数据，不能处理数据。

##### 4.1、基本数据源

- Socket流：`ssc.socketTextStream(...)` ，用接收自TCP socket连接的文本数据创建DStream

- 文件流（File Streams）：用于从与HDFS API兼容的文件系统中的文件读取数据，DStream创建：

  ```
  streamingContext.fileStream[KeyClass, ValueClass, InputFormatClass](dataDirectory)
  ```

  Spark Streaming会监控目录`dataDirectory`，并且处理目录中创建的任何文件（不支持改目录子目录中文件的读取）。需要注意的是：

  - 这些文件数据格式必须相同
  - 这些文件必须通过（原子操作地）移动（moving）或者重命名（renaming）到`dataDirectory`目录
  - 一旦移动过后，这些文件被禁止修改。如果往文件中追加数据，将不会读取新追加的数据。对于文本文件有一个更简单的方法`streamingContext.textFileStream(dataDirectory)`。文件流不需要运行receiver，所以，也就不需要分配核心。

- 基于自定义Receiver的流：可以通过自定义receiver接收的数据流创建DStream

- RDDs的Queue作为流：对于Spark Streaming应用的测试数据，可以基于RDDs的队列来创建DStream，用`streamingContext.queueStream(queueOfRDDs)`。每个被压入队列的RDD都会被作为DStream中的一个数据批次，并且被作为流进行处理。

##### 4.2、高级数据源（Advanced Sources）

需要使用外部的非Spark库的数据源，Spark shell中默认不能使用，需要把对应的数据源依赖加入到Spark shell的classpath才能使用。比如，[Kafka](http://spark.apache.org/docs/2.2.0/streaming-kafka-integration.html)、[Flume](http://spark.apache.org/docs/2.2.0/streaming-flume-integration.html)、[Kinesis](http://spark.apache.org/docs/2.2.0/streaming-kinesis-integration.html)。

##### 4.3、自定义数据源

可以从自定义数据源创建输入DStreams。只用实现一个用户自定义的Receiver，并且这个Receiver能够从自定义数据源接收数据并把数据推送给Spark。详细参见[文档](http://spark.apache.org/docs/2.2.0/streaming-custom-receivers.html)。

##### 4.4、Receiver可靠性

基于可靠性可以把数据源分为两类：

1. 可靠Receiver：在Receiver从数据源收到数据并把数据用备份保存在Spark中时，Receiver会发送确认信息（acknowledgement）给数据源。
2. 不可靠Receiver：不会往数据源发送确认信息的Receiver。可以用于不支持确认信息的数据源，也可以用于不需要复杂的确认信息的可靠数据源（比如Kafka、Flume）。

#### 5、DStreams的转换操作

常见的DStream转换操作：`map`，`flatMap`，`filter`，`repartition`，`union`，`count`，`count`，`reduce`，`countByValue`，`reduceByKey`，`join`，`cogroup`，`transform`，`updateStateByKey`。

##### 5.1、`UpdateStateByKey`操作

`updateStateByKey`操作可以在维持任意状态的同时，持续地用新的信息来更新它。使用这个操作，要执行如下两个步骤：

- 定义状态：状态可以是任意的数据类型；
- 定义状态更新函数：指定一个函数，该函数定义了如何使用之前状态和输入流中的新数据来更新状态。

在每个批次中，Spark会为所有的键应用状态更新函数，不管这一批次中它们有没有新的数据。如果更新函数返回`None`对应的键值对会被清除。

###### 例子：

```scala
def updateFunction(newValues: Seq[Int], runningCount: Option[Int]): Option[Int] = {
    val newCount = ...  // add the new values with the previous running count to get the new count
    Some(newCount)
}
```

此处，`runningCount`是先前的状态值，`newValues`是新的输入数据，函数返回新的状态。

```scala
val runningCounts = pairs.updateStateByKey[Int](updateFunction _)
```

使用`updateStateByKey`需要设置检查点（checkpoint）目录。

##### 5.2、`transform`操作

`transform`操作（以及它的变种`transformWith`）可以将任意的RDD-to-RDD函数应用到DStream上。可以用它来实现DStream API中没有的RDD操作。例如，DStream API中没有把数据流中每个批次与另一个数据集进行连接的功能；但是，可以通过`transform`来实现。

```scala
val spamInfoRDD = ssc.sparkContext.newAPIHadoopRDD(...) // RDD containing spam information

val cleanedDStream = wordCounts.transform { rdd =>
  rdd.join(spamInfoRDD).filter(...) // join data stream with spam information to do data cleaning
  ...
}
```

每个批次间隔都会调用提供的函数。所以，可以进行随时间变化（time-varying）的RDD操作，例如，对于不同的批次，RDD操作、分区数、广播变量可以不同。

##### 5.3、窗口操作

Spark Streaming也支持开窗运算（windowed computations），可以对于一个窗口的数据进行转换操作。

![Spark Streaming](/assets/streaming-dstream-window.png)

如图，可见窗口操作需要指定的两个参数，窗口长度（window length）是3，滑动间隔（sliding interval）是2。这两个参数必须是源DStream批次间隔（图中是1）的整数倍。

例如：

```scala
// Reduce last 30 seconds of data, every 10 seconds
val windowedWordCounts = pairs.reduceByKeyAndWindow((a:Int,b:Int) => (a + b), Seconds(30), Seconds(10))
```

常见的窗口操作：`window`，`countByWindow`，`reduceByWindow`，`reduceByKeyAndWindow`，`countByValueAndWindow`。

##### 5.4、连接操作

###### 5.4.1、Stream-stream连接

```scala
val stream1: DStream[String, String] = ...
val stream2: DStream[String, String] = ...
val joinedStream = stream1.join(stream2)
```

每个批次间隔中，`stream1`产生的RDD和`stream2`产生的RDD进行连接。可以执行`leftOuterJoin`，`rightOuterJoin`，`fullOuterJoin`。另外对流窗口进行连接也很有用：

```scala
val windowedStream1 = stream1.window(Seconds(20))
val windowedStream2 = stream2.window(Minutes(1))
val joinedStream = windowedStream1.join(windowedStream2)
```

###### 5.4.2、Stream-dataset连接

需要用到流数据窗口的`transform`操作：

```scala
val dataset: RDD[String, String] = ...
val windowedStream = stream.window(Seconds(20))...
val joinedStream = windowedStream.transform { rdd => rdd.join(dataset) }
```

事实上，可以动态地改变进行连接的`dataset`。提供给`transform`的函数对于每个批次的执行都重新加载，所以会用当前`dataset`所引用的数据集。

#### 6、DStream的输出操作

与RDDs的行动操作类似，DStreams的输出操作触发DStreams的转换操作的执行。

- print()：在运行Streaming应用的驱动器节点上输出DStream中每个批次的前10个元素。
- saveAsTextFiles：将DStream的内容保存为文本文件。
- saveAsObjectFiles：将DStream内容保存为序列化的Java对象的SequenceFiles文件。
- saveAsHadoopFiles：将DStream内容保存为Hadoop文件。
- foreachRDD：为流中产生的每个RDD执行一个函数。所执行的函数应该把每个RDD的数据推送到外部系统，比如保存RDD为文件或者把RDD通过网络存入数据库。这个函数是在运行流应用的驱动进程中执行的，通常会有RDD行动操作来触发流RDDs的运算。

##### 6.1、使用foreacheRDD的设计模式（Design Patterns）

`dstream.foreachRDD`是一个功能强大的原函数。但是使用时需要注意，RDD是一个分布式的数据集，如果要创建连接对象，创建连接对象的位置需要注意了：

```scala
dstream.foreachRDD { rdd =>
  val connection = createNewConnection()  // executed at the driver
  rdd.foreach { record =>
    connection.send(record) // executed at the worker
  }
}
```

上面的例子是错误的，执行时会抛出连接对象不能序列化之类的错误。

```scala
dstream.foreachRDD { rdd =>
  rdd.foreach { record =>
    val connection = createNewConnection()
    connection.send(record)
    connection.close()
  }
}
```

上面的例子也是错误的，为RDD每条记录创建连接对象会有很大的时间和资源开销。较好的解决方案是使用`rdd.foreachPartition`——为每个RDD分区创建一个连接对象：

```scala
dstream.foreachRDD { rdd =>
  rdd.foreachPartition { partitionOfRecords =>
    val connection = createNewConnection()
    partitionOfRecords.foreach(record => connection.send(record))
    connection.close()
  }
}
```

使用连接池，可以进一步对上面的代码进行优化：

```scala
dstream.foreachRDD { rdd =>
  rdd.foreachPartition { partitionOfRecords =>
    // ConnectionPool is a static, lazily initialized pool of connections
    val connection = ConnectionPool.getConnection()
    partitionOfRecords.foreach(record => connection.send(record))
    ConnectionPool.returnConnection(connection)  // return to the pool for future reuse
  }
}
```

其它需要注意的：

- DStreams是通过输出操作懒惰地执行的，与RDDs通过RDD行为操作懒惰执行类似。特别是，DStream中的RDD行为操作推动了对接收到的数据的处理。因此，如果应用没有任何的输出操作，或者使用了`dstream.foreachRDD`而其中的函数没有任何的RDD行为操作，那么就什么都不会执行。系统只会简单地接收数据然后丢弃它。
- 默认情况下，一次只执行一个输出操作，并且按照在应用中定义的顺序执行。

#### 7、DataFrame和SQL操作

可以简便地将DataFrame和SQL操作用于数据流。必须要用StreamingContext使用的SparkContext创建一个SparkSession。

```scala
/** DataFrame operations inside your streaming program */

val words: DStream[String] = ...

words.foreachRDD { rdd =>

  // Get the singleton instance of SparkSession
  val spark = SparkSession.builder.config(rdd.sparkContext.getConf).getOrCreate()
  import spark.implicits._

  // Convert RDD[String] to DataFrame
  val wordsDataFrame = rdd.toDF("word")

  // Create a temporary view
  wordsDataFrame.createOrReplaceTempView("words")

  // Do word count on DataFrame using SQL and print it
  val wordCountsDataFrame = 
    spark.sql("select word, count(*) as total from words group by word")
  wordCountsDataFrame.show()
}
```

可以从一个不同的线程（即，与运行的StreamingContext异步）来运行基于流数据定义的表的SQL查询。须要确保StreamingContext记住了足够量的流数据，以便查询可以执行。否则，StreamingContext不知道异步SQL查询的存在，会在查询完成前删掉旧的流数据。比如，如果要查询上一个批次，但是查询需要执行5分钟，那么就调用`streamingContext.remember(Minutes(5))`。

#### 8、MLlib操作

可以简便的运用MLlib提供的机器学习算法。首先，有流式的机器学习算法（比如`Streaming liner Regression`，`Streaming KMeans`等）可以在从流数据学习的同时把模型应用到流数据。此外，对于大多数的机器学习算法，可以离线进行模型训练（比如使用历史数据）然后把模型运用于在线的流数据。

#### 9、缓存/持久化

DStreams也可以把流数据持久化到内存。使用DStream的`persist()`方法可以自动地将DStream的每个RDD持久化到内存。如果要多次使用DStream中的数据，这是很有用的。对于基于窗口或基于状态的操作，比如`reduceByWindow`，`reduceByKeyAndWindow`，`updateStateByKey`，持久化是隐式开启的。因此，不用调用`persist()`，基于窗口的操作产生的DStream会自动地持久化到内存。

对于通过网络（比如Kafka，Flume，sockets等）接收数据的输入流，默认的持久化级别是把数据备份到两个节点以容错。

与RDDs不同，DStream的默认持久化级别是把数据序列化到内存。

#### 10、检查点

流式应用必须24/7地运行，因此必须对应用逻辑无关的故障（比如，系统故障、JVM崩溃）具有弹性。因此，Spark Streaming需要*checkpoint*足够的信息到容错的存储系统，以便能够从故障中恢复。会对两类数据进行*checkpoint*：

- *Metadata checkpointing*：把定义流运算的信息保存到容错的存储系统（比如HDFS）。用来从运行Streaming应用的driver的节点故障中恢复。Metadata包括：
  - *Configuration*：创建流应用的配置。
  - *DStream operations*：定义流应用的DStream操作。
  - *Incomplete batches*：那些作业已经加入队列但是没有完成的批次。
- *Data checkpointing*：把产生的RDDs保存到可靠存储。在某些需要结合多批次数据的有状态的转换操作中是必要的。在这些转换操作中，生成的RDDs依赖于之前批次的RDDs，这就导致依赖链的长度随着时间保持增长。为了避免恢复时的无限增长，有状态的转换操作的中间RDDs会每隔一段时间就*checkpoint*到可靠存储以斩断依赖链。

总之，metadata checkpointing主要是为了从driver故障恢复，而如果使用了有状态的转换操作，data checkpointing对于基本运行是必须的。

##### 10.1、何时开启检查点

- 使用有状态的转换操作时：比如使用`updateStateByKey`或`reduceByKeyAndWindow`时，同时需要设置检查点目录。
- 从运行应用的driver故障中恢复时：Metadata checkpoints用于使用进度信息来恢复。

不使用有状态的转换操作的简单流应用不开启检查点也可以运行，但是会存在应用从driver故障中恢复的问题（会丢失已经接收但是未处理的数据）。

##### 10.2、配置检查点

通过设置一个检查点目录来开启检查点。`streamingContext.checkpoint(checkpointDirectory)`。如果要从driver故障中恢复：

- 首次启动程序时，会创建一个新的StreamingContext，配置所有的流然后调用`start()`
- 从故障恢复时，从检查点目录中的检查点数据重建StreamingContext

使用`StreamingContext.getOrCreate`来实现：

```scala
// Function to create and setup a new StreamingContext
def functionToCreateContext(): StreamingContext = {
  val ssc = new StreamingContext(...)   // new context
  val lines = ssc.socketTextStream(...) // create DStreams
  ...
  ssc.checkpoint(checkpointDirectory)   // set checkpoint directory
  ssc
}

// Get StreamingContext from checkpoint data or create a new one
val context = StreamingContext.getOrCreate(checkpointDirectory, functionToCreateContext _)

// Do additional setup on context that needs to be done,
// irrespective of whether it is being started or restarted
context. ...

// Start the context
context.start()
context.awaitTermination()
```

如果`checkpointDirectory`存在，则从检查点数据创建context，如果`checkpointDirectory`目录不存在则调用``functionToCreateContext` `函数创建一个新的context并设置DStream

除了使用`getOrCreate`方法，还需要确保driver进程在遇到故障时自动重启。这就需要运行应用的部署框架来实现了。

注意，RDDs的checkpoint会引起保存数据的开销，可能会导致处理时间的增加。因此，checkpointing的时间间隔需要认真设置。对于小批次间隔（比如1秒）checkpointing每个批次可能会显著地减少操作通量（throughput），相反地，checkpointing频率太低则会导致lineage和task size增加，可能会有不利的影响。对于需要RDD checkpointing的有状态的转换操作，默认的时间间隔是批次间隔的整数倍，至少10s，可以通过`dstream.checkpoint(checkpointInterval)`设置。一般，checkpoint时间间隔是DStream批次间隔的5-10倍。

#### 11、Accumulators、Broadcast Variables，和 Checkpoints

Spark Streaming中，累加器和广播变量不能从checkpoint恢复。如果开启了checkpointing，并且使用了累加器或者广播变量，那么必须为累加器或者广播变量创建懒实例化单个实例（lazily instantiated singleton instances）以便在driver从故障中恢复时可重新将它们实例化。

```scala
object WordBlacklist {

  @volatile private var instance: Broadcast[Seq[String]] = null

  def getInstance(sc: SparkContext): Broadcast[Seq[String]] = {
    if (instance == null) {
      synchronized {
        if (instance == null) {
          val wordBlacklist = Seq("a", "b", "c")
          instance = sc.broadcast(wordBlacklist)
        }
      }
    }
    instance
  }
}

object DroppedWordsCounter {

  @volatile private var instance: LongAccumulator = null

  def getInstance(sc: SparkContext): LongAccumulator = {
    if (instance == null) {
      synchronized {
        if (instance == null) {
          instance = sc.longAccumulator("WordsInBlacklistCounter")
        }
      }
    }
    instance
  }
}

wordCounts.foreachRDD { (rdd: RDD[(String, Int)], time: Time) =>
  // Get or register the blacklist Broadcast
  val blacklist = WordBlacklist.getInstance(rdd.sparkContext)
  // Get or register the droppedWordsCounter Accumulator
  val droppedWordsCounter = DroppedWordsCounter.getInstance(rdd.sparkContext)
  // Use blacklist to drop words and use droppedWordsCounter to count them
  val counts = rdd.filter { case (word, count) =>
    if (blacklist.value.contains(word)) {
      droppedWordsCounter.add(count)
      false
    } else {
      true
    }
  }.collect().mkString("[", ", ", "]")
  val output = "Counts at time " + time + " " + counts
})
```

#### 12、部署应用

##### 12.1、部署前提（requirements）

Streaming应用的部署前提：

- 有集群管理器的集群
- 打包应用JAR
- 为executors配置足够的内存
- 配置checkpointing
- 配置应用driver的自动重启
  - Spark Standalone：Standalone汲取管理器可以监督driver，对driver进行重启
  - YARN：Yarn也可以自动重启应用
  - Mesos：使用Marathon重启应用
- Configuring wirte ahead logs：用来保证强容错，如果开启，从receiver接收的数据都会写到checkpoint目录中中的一个write ahead log中。
- 设置最大接收率：如果集群资源不足以处理接收到的数据，可以限制接收率（记录数/秒）。

##### 12.2、更新应用代码

如果要用新的应用代码更新正在运行的Spark Streaming应用，有两种可用的机制：

- 启动新的Spark Streaming应用与旧的并行运行。新的运行正常后，旧的就可以撤下。这适用于支持把数据发送到两个目的地的数据源。
- 优雅地（gracefully）关闭旧的应用，可以确保关闭前接收到的数据处理完毕。然后启动新的应用，接着旧的应用处理完的数据的点继续进行处理。这适用于可以在数据源侧缓存数据的数据源（比如Kafka、Flume），并且不能让新的应用从旧应用的检查点恢复。可以删除旧的检查点目录，或者新的应用使用不同的检查点目录。

#### 13、应用监控

除了Spark的监控工具，Spark Streaming有专用的监控工具。当使用了StreamingContext，Spark Web UI会展示额外的`Streaming`标签，这个页面展示了运行的receivers（receivers是否活跃，接收的记录数，receiver错误，等等）和完成的批次的统计数据，可以用来监控streaming应用的进度。

Web UI中最重要的两个指标是：

- Processing Time：处理每个批次数据的时间；
- Scheduling Delay：队列中一个批次等待上一个批次处理完成的时间。

如果批次处理时间总是比批次间隔长，并且（或）队列（调度）延时一直增长，这表示系统不能及时的处理数据批次；这时就要考虑降低批次的处理时间。

还可以使用`StreamingListener`接口来监控Spark Streaming的进度。