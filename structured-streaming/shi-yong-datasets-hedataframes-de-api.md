### 使用Datasets和DataFrames的API

从Spark 2.0开始，DataFrames和Datasets可以表示静态的有边界的数据，也可以表示流、无边界的数据。与静态Datasets/DataFrames类似，可以使用通用的入口`SparkSession`来从流数据源创建流DataFrames/Datasets，并且可以对它们应用与静态DataFrames/Datasets相同的操作。

#### 1、创建流DataFrames和流Datasets

可以用`SparkSession.readStream()`返回的`DataStreamReader`接口来创建流DataFrames。在R语言中，使用`read.stream()`方法。与创建静态DataFrame的read接口类似，可以指定一些数据源的详情——数据格式，schema，options，等等。

##### 1.1、输入数据源

Spark 2.0中，有一些内置的数据源：

- **File source**：将写到一个目录中的文件读取为一个数据的流。支持的文件格式有：text、csv、json、parquet。可以通过`DataStreamReader`接口的文档来查看最新支持的格式，和每个格式支持的options。注意，文件必须原子性地放在指定的目录中，在大多数的文件系统中，可以通过文件移动操作实现原子性。

  文件数据源是容错的；支持的选项有`path`（输入**目录**路径）、`maxFilesPerTrigger`（每次触发考虑的新文件的最大数量，默认没有最大数量限制）、`latestFirst`（是否首先处理最新的文件，默认false，在处理大量文件时是有用的）、`fileNameOnly`（是否只基于文件名而不是全路径来检查新的文件，默认false，如果设置为true，则只考虑文件名，比如，此时` "file:///dataset.txt"`和` "s3://a/dataset.txt"`是相同的）。

  文件数据源支持通配符路径，但是不支持逗号分隔的多路径。

- **Kafka source**：从Kafka获取数据，参考集成Kafka章节。兼容0.10.0和更高版本的Kafka broker。Kafka数据源是容错的。

- **Socket source**（用于测试）：从socket连接读取UTF8文本数据。在driver中监听server socket。注意，这种数据源只能用于测试，它是不提供端到端的容错保证的。支持的选项有`host`（必须，主机名）和`port`（必须，端口）。

```scala
val spark: SparkSession = ...

// Read text from socket
val socketDF = spark
  .readStream
  .format("socket")
  .option("host", "localhost")
  .option("port", 9999)
  .load()

// Returns True for DataFrames that have streaming sources
socketDF.isStreaming

socketDF.printSchema

// Read all the csv files written atomically in a directory
val userSchema = new StructType().add("name", "string").add("age", "integer")
val csvDF = spark
  .readStream
  .option("sep", ";")
  .schema(userSchema)      // Specify schema of the csv files
  // Equivalent to format("csv").load("/path/to/directory")
  .csv("/path/to/directory")
```

这些例子中生成的流DataFrames是无类型的（untyped），即在编译时不会检查这些DataFrames的schema，只有运行时当查询提交后才会检查。对于某些需要在编译时知道类型的操作，比如`map`，`flatMap`等，可以使用与静态DataFrame相同的方法将无类型的流DataFrame转换为有类型的流Dataset，详细参看Spark SQL相关章节。

##### 1.2、Schema推导和流式DataFrames/Datasets的分区

默认情况下，使用基于文件的数据源的Structured Streaming需要指定schema，而不是依赖Spark去自动地推导Schema。这个限制确保流查询使用的schema是一致的，即使是发生了故障。对某些特定的（ad-hoc）使用场景，可以通过设置`spark.sql.streaming.schemaInference`为`true`来开启schema推导。

当命名为`/key=value/`的子目录出现时会发生分区发现（partition discovery），分区发现会在这些子目录中递归。如果这些字段出现在了用户提供的schema中，Spark会基于读取文件的路径对它们进行填充。构成分区schema的目录必须在查询启动时就存在，并且必须是静态的。比如，`/data/year=2015`存在时添加`/data/year=2016`是可以的，但是改变分区字段是非法的（即，创建目录`/data/date=2016-04-17`）。

#### 2、对流式DataFrames/Datasets的操作

可以对流DataFrames/Datasets应用各种操作——包括无类型的（untyped）、类SQL操作（比如`select`，`where`，`groupby`）、有类型的（typed）类RDD操作（比如`map`，`filter`，`flatMap`）。

##### 2.1、基本操作-Selection，Projection，Aggregation

大多数DataFrame/Dataset操作都适用于流DataFrame/Dataset。本节稍后会讨论那些不支持的操作。

```scala
case class DeviceData(device: String, deviceType: String, signal: Double, time: DateTime)
// streaming DataFrame with IOT device data with schema { device: string, deviceType: string, signal: double, time: string }
val df: DataFrame = ... 
// streaming Dataset with IOT device data
val ds: Dataset[DeviceData] = df.as[DeviceData]    

// Select the devices which have signal more than 10
df.select("device").where("signal > 10")      // using untyped APIs   
ds.filter(_.signal > 10).map(_.device)         // using typed APIs

// Running count of the number of updates for each device type
df.groupBy("deviceType").count()                          // using untyped API

// Running average signal for each device type
import org.apache.spark.sql.expressions.scalalang.typed
ds.groupByKey(_.deviceType).agg(typed.avg(_.signal))    // using typed API
```

##### 2.2、基于事件时间的窗口操作

用Structured Streaming对一个滑动的事件时间窗口的聚合很简单，与分组聚合（grouped aggregations）很相似。在分组聚合中，聚合值（比如，counts）是针对用户指定的分组字段的每个唯一值的。而在基于窗口的聚合中，聚合值是针对每个事件时间窗口进行的。通过一个例子来阐释它……

数据流包含文本行和文本行产生的时间，计算窗口时长10分钟，每5分钟更新的窗口中的每个单词的数量。即，统计窗口12:00 - 12:10, 12:05 - 12:15, 12:10 - 12:20等等期间接收到的单词的单词数量。12:00 - 12:10意味着12:00之后12:10之前接收到的数据。比如，12:07接收到的单词既在12:00 - 12:10窗口中，也在12:05 - 12:15窗口中。

结果表的用图像表示如下：

![Window Operations](/assets/structured-streaming-window.png)

因为窗口操作和分组操作类似，代码上，可以用`groupBy()`和`window()`操作来表示窗口聚合（windowed aggregations）。

```scala
import spark.implicits._
// streaming DataFrame of schema { timestamp: Timestamp, word: String }
val words = ... 

// Group the data by window and word and compute the count of each group
val windowedCounts = words.groupBy(
  window($"timestamp", "10 minutes", "5 minutes"),
  $"word"
).count()
```

##### 2.3、处理迟到数据和水印

如果数据迟到，比如，应用在12:11接收到一个12:04产生的单词。应用应该用时间12:04而不是12:11来更新窗口12:00-12:10的计数。对于基于窗口的分组操作这是很自然地——Structured Streaming可以维持部分聚合的中间状态很长一段时间，以便迟到的数据可以正确地更新旧的窗口，如下图所示。

![Handling Late Data](/assets/structured-streaming-late-data.png)

但是，要运行查询很多天的话，很有必要限制在内存中的累加的中间状态的量。这意味着，系统需要知道什么时间（因为应用将不再为这个旧的聚合接收迟到的数据）可以从内存状态中删除旧的聚合。要实现它，Spark 2.1中采用了水印（watermarking），让引擎自动的跟踪数据中的当前事件时间并根据情况清除旧的状态。可以通过指定事件时间字段和事件时间的迟到阈值来定义一个查询的水印。对于在时间`T`启动的特定窗口，引擎会维持状态并允许迟到的数据更新这个状态直到达到水印时间（水印时间`wm`=最大事件时间-减去-迟到阈值，最大事件时间就是上次的触发时间）。即，阈值范围内的迟到数据会被聚合，但是更晚的数据会被删除。

使用`withWatermark()`可以定义水印：

```scala
import spark.implicits._
// streaming DataFrame of schema { timestamp: Timestamp, word: String }
val words = ... 

// Group the data by window and word and compute the count of each group
val windowedCounts = words
    .withWatermark("timestamp", "10 minutes")
    .groupBy(
        window($"timestamp", "10 minutes", "5 minutes"),
        $"word")
    .count()
```

此处，用字段“timestamp”定义了水印，并且指定“10 minutes”作为迟到阈值。如果查询的输出模式是“Update”，引擎会持续更新某个窗口在结果表中的计数，直到窗口时间超过了水印的阈值。每次触发后，更新后的计数会被作为本次触发的输出写到外部接收器。比如，12:15-12:20（12:20触发）收到了一个12:08的数据，窗口12:00-12:10对应的结果数据会被更新并写到外部接收器。

某些外部接收器（比如，文件）不支持“Update”输出模式需要的细粒度更新。为了能使用它们，也支持“**Append**”模式，此时，只有最终的计数结果（当超过水印时间范围的窗口的结果）才会写到外部接收器，如下图所示。

注意，对一个非流式的Dataset使用`withWatermark`是没有任何效果的。因为，水印不以任何方式影响批查询，会被直接忽略。

![Watermarking in Append Mode](/assets/structured-streaming-watermark-append-mode.png)

与“Update”模式类似，引擎会维护每个窗口的中间计数结果。但是，直到水印时间大于窗口的时间才会把结果表中的对应行写出到外部接收器。比如，12:20-12:25（12:25触发）收到一个12:04的数据和12:08的数据，窗口12:00-12:10对应的结果不再更新并且会被写到外部接收器，12:04对应的数据会被删除；12:08的数据不会被删除，12:05-12:15对应的结果会被更新但是不会写到外部接收器。

**水印清除聚合状态的条件**：

- **输出模式必须是“Append”或“Update”**：“Complete”模式是需要保留所有聚合数据的，因此不能使用水印删除中间状态。
- 聚合必须要有事件时间字段或者基于事件时间字段的`window`
- `withWatermark`使用的字段和聚合使用时间字段必须相同；`df.withWatermark("time", "1 min").groupBy("time2").count()` 是错误的。
- `withWatermark`必须在聚合之前调用，以便水印详情可以被应用；`df.groupBy("time").count().withWatermark("time", "1 min")`是错误的。

##### 2.4、连接操作

流DataFrames可以和静态的DataFrames进行连接以创建新的流DataFrames：

```scala
val staticDf = spark.read. ...
val streamingDf = spark.readStream. ...
// inner equi-join with a static DF
streamingDf.join(staticDf, "type")          
// right outer join with a static DF
streamingDf.join(staticDf, "type", "right_join")  
```

##### 2.5、流去重

可以使用事件中的唯一identifier来去重数据流中的记录。这和静态DataFrame使用唯一的identifier字段去重是相同的。查询会从之前的记录保存必要数量的数据以便过滤重复的记录。与聚合类似，用不用水印都可以去重。

- 用水印：如果有重复数据最后到达的时间上限，可以基于事件时间定义一个水印，并且用guid和事件时间字段来去重。查询会使用水印把过去的记录中不重复记录的的状态数据删除。这限定了查询需要维护的状态的数量。
- 不用水印：如果没有重复数据最后到达的边界，查询会存储所有过去的记录作为状态。

```scala
val streamingDf = spark.readStream. ...  // columns: guid, eventTime, ...

// Without watermark using guid column
streamingDf.dropDuplicates("guid")

// With watermark using guid and eventTime columns
streamingDf
  .withWatermark("eventTime", "10 seconds")
  .dropDuplicates("guid", "eventTime")
```

##### 2.6、任意有状态操作

许多场景需要比聚合更加高级的有状态操作。比如，许多场景中，必须要从事件的数据流追踪会话。要进行会话化（sessionization），必须保存任意类型的数据作为状态，并且在每次触发时使用数据流事件对状态执行任意的操作。从Spark 2.2开始可以使用操作`mapGroupWithState`和更加强大的操作`flatMapGroupsWithState`进行会话化。两种操作都可以将用户定义的代码应用到分组的Datasets上以更新用户定义的状态。

##### 2.7、不支持的操作

- 流Datasets的多重流聚合（即，对一个流DF进行一串的聚合）暂不支持
- 流Datasets的`limit`和take前N行不支持
- 流Datasets的`distinct`操作不支持
- 流Datasets在聚合后和在Compelte输出模式时，才支持排序操作
- 流和静态Datasets的外连接是有条件支持的：
  - 和流Dataset的全外连接不支持
  - 流Dataset在右边的左外连接不支持
  - 流Dataset在左边的右外连接不支持
- **不支持两个流Datasets的任意连接**

此外，某些Dataset方法在流Datasets上不起作用。它们是会直接运行查询并返回结果的行动操作，对流Dataset是没有用的。但是，通过明确地启动一个流查询这些功能是可以实现的。

- `count()`：不能返回流Dataset的记录数量。使用`ds.groupBy().count()`来返回一个包含运行中的计数的流Dataset。
- `foreach()`：使用`ds.writeStream.foreach(...)`作为替代（下节介绍）
- `show()`：使用console外部接收器作为替代（下节介绍）。

如果对流Dataset直接使用这些操作，将会发生像"operation XYZ is not supported with streaming DataFrames/Datasets"这种`AnalysisException`错误。但是，以后版本的Spark可能会支持它们中的某些操作，其它的操作（比如排序）在流数据上高效地实现是困难的。

#### 3、启动流查询

使用`Dataset.writeStream()`返回的`DataStreamWriter`来启动流运算。需要在接口中指定如下的一个或者多个：

- 输出外部接收器的详情：数据格式、位置等
- 输出模式：输出到输出外部接收器的内容
- 查询名称：可选，为查询指定一个方便识别的唯一名称
- 触发时间间隔（trigger interval）：可选，指定触发时间间隔。如果不指定，系统会在之前的数据处理完成后立即检查新的数据是否可用。如果因为之前的处理未完成而错过了触发时间，系统会尝试在下一次触发时间触发，而不是处理完成后马上触发。
- 检查点位置：对于需要端到端容错保证的输出外部接收器，指定检查点位置以保存检查点信息。应该是一个兼容HDFS的容错的文件系统目录。检查点语义会在下节介绍。

##### 3.1、输出模式

输出模式的类型：

- **Append mode**（默认）：默认的模式，只有最近一次触发引起的追加到结果表的新行会被输出到外部接收器。只有已经追加到结果表的行不会改变的查询才支持这种模式。因此，这种模式能够保证每行只输出一次（假设是容错的外部接收器）。比如，只有`select`，`where`，`map`，`flatMap`，`filter`，`join`等的查询支持使用追加模式。
- **Compelte mode**：每次触发后，整个结果表会输出到外部接收器。聚合查询支持使用这种模式。
- **Update mode**：（从Spark 2.1.1可用）只有最近一次触发后结果表中被更新的行才会被输出到外部接收器。

不同类型的流查询支持不同输出模式。

**基于事件时间并且有水印的聚合**支持使用三种输出模式。追加模式时，使用水印可以删除旧的聚合状态，但是因为水印的使用和追加模式的语义，窗口聚合结果的输出会延时，延时时间即在`withWatermark()`中指定的延迟阈值，结果只有在最终确定之后（finalized）才会被加到结果表。更新模式时，使用水印删除旧的聚合状态。完整模式不会删除旧的聚合状态，因为按照定义这种模式将所有数据保留在结果表中。

**不使用水印的聚合**不支持追加模式，因为聚合结果的更新与这种模式的语义相冲突。因为没有使用水印，旧的聚合状态不会被删除。

**使用`mapGroupsWithState`的查询**支持更新模式。

**使用`flatMapGroupsWithState`的查询**，在使用追加模式时，`flatMapGroupsWithState`后面可以使用聚合；而在使用更新模式时，`flatMapGroupsWithState`后面不能使用聚合。

其它查询，不能使用完整模式，因为在结果表中保存所有的非聚合数据是不可行的。

##### 3.2、输出外部接收器

内置的输出外部接收器类型有：

- **File sink**：保存输出到目录。是容错的。支持输出到分区表。必须用`path`选项指定输出目录。支持的输出格式参考`DataFrameWriter`中的方法。只支持追加输出模式。

  ```scala
  writeStream
      .format("parquet")        // can be "orc", "json", "csv", etc.
      .option("path", "path/to/destination/dir")
      .start()
  ```

- **Foreach sink**：对输出的记录运行任意的运算。是否容错要根据`ForeachWriter`的实现而定。支持三种输出模式。

  ```scala
  writeStream
      .foreach(...)
      .start()
  ```

- **Console sink（用于debugging）**：每次触发时把输出打印到控制台或者标准输出。支持追加模式和完整模式。因为每次触发后的所有输出都被收集并存储在driver的内存中，这种sink只能用于数据量不大的debugging。可以使用`numRows`（默认20）选项指定每次触发输出的行数；使用`truncate`（默认true）在输出太长时截掉输出。支持三种输出模式。

  ```scala
  writeStream
      .format("console")
      .start()
  ```

- **Memory sink（for debugging）**：输出以内存表的形式保存在内存中。因为所有输出都被收集并存储在driver的内存中，这种sink只能用于数据量不大的debugging。内存表的表名就是查询名。支持追加和完整两种输出模式。不容错，但是使用完整输出模式时，重启查询可以重建整个表。

  ```scala
  writeStream
      .format("memory")
      .queryName("tableName")
      .start()
  ```

注意，需要调用`start()`方法来事实上启动查询的执行。`start()`方法返回一个`StreamingQuery`对象，它是持续运行的查询的一个操作句柄，可以用它来管理查询。

例子：

```scala
// ========== DF with no aggregations ==========
val noAggDF = deviceDataDf.select("device").where("signal > 10")   

// Print new data to console
noAggDF
  .writeStream
  .format("console")
  .start()

// Write new data to Parquet files
noAggDF
  .writeStream
  .format("parquet")
  .option("checkpointLocation", "path/to/checkpoint/dir")
  .option("path", "path/to/destination/dir")
  .start()

// ========== DF with aggregation ==========
val aggDF = df.groupBy("device").count()

// Print updated aggregations to console
aggDF
  .writeStream
  .outputMode("complete")
  .format("console")
  .start()

// Have all the aggregates in an in-memory table
aggDF
  .writeStream
  .queryName("aggregates")    // this query name will be the table name
  .outputMode("complete")
  .format("memory")
  .start()

spark.sql("select * from aggregates").show()   // interactively query in-memory table
```

##### 3.3、使用Foreach

`foreach`操作从Spark 2.1开始可用。使用它可以对输出数据执行任意的操作。要使用它，需要实现`ForeachWriter`接口，流触发后产生了输出结果时，会调用`ForeachWriter`中的方法。需要注意的是：

- 这个writer必须是可序列化的，因为它会被序列化并发送到executors去执行
- executors会执行writer的所有方法（`open`，`process`，`close`）
- 只有当调用writer的`open`方法时才执行所有的初始化（比如，打开连接、开启事务等）。要知道，如果创建对象时就初始化，会在driver上发生初始化（因为driver会创建它的实例），这可能是不愿见到的。
- `version`和`partitionId`是`open`方法的两个参数，唯一性的代表一组需要输出的记录。`version`是一个随着每次触发单调递增的id。`partitionId`是一个表示输出分区的id，因为输出是分布式的并且会在多个executors上进行。
- `open`会使用`version`和`partitionId`来判断是否要输出这些记录，并返回true或false。如果返回false，`process`方法就不会处理这些记录。比如，一个故障发生后，失败的触发对应的输出分区的一部分可能已经保存到数据库。基于数据库的元数据，writer可以分辨那些分区已经输出，并根据情况返回false以跳过再次将它们输出。
- 调用完`open`后，也会调用`close`（除非JVM因为错误而退出）。即使`open`返回false，也会调用`close`。如果在处理和写数据时发生错误，`close`会随着错误而被调用。在`close`中清除`open`中创建的状态（比如，连接、事务等）以免资源泄露。

#### 4、管理流查询

`StreamQuery`对象可以用来监控和管理查询。

```scala
val query = df.writeStream.format("console").start()   // get the query object
// get the unique identifier of the running query that persists across restarts from checkpoint data
query.id          
// get the unique id of this run of the query, which will be generated at every start/restart
query.runId       
// get the name of the auto-generated or user-specified name
query.name        
// print detailed explanations of the query
query.explain()   
// stop the query
query.stop()      
// block until query is terminated, with stop() or with error
query.awaitTermination()   
// the exception if the query has been terminated with error
query.exception       
// an array of the most recent progress updates for this query
query.recentProgress  
// the most recent progress update of this streaming query
query.lastProgress    
```

一个SparkSession中可以启动任意数量的查询。它们会共享集群资源，同步运行。可以使用`sparkSession.streams()`来获取`StreamingQueryManager`，使用`StreamingQueryManager`可以管理当前活跃的查询。

```scala
val spark: SparkSession = ...

spark.streams.active    // get the list of currently active streaming queries

spark.streams.get(id)   // get a query object by its unique id

spark.streams.awaitAnyTermination()   // block until any one of them terminates
```

#### 5、监控流查询

有两种监控和debugging活跃查询的APIs——交互地和异步的。

##### 5.1、交互APIs

使用`streamingQuery.lastProgress()` 和`streamingQuery.status()`获取活跃查询的当前状态和metrics。`lastProgress()` 返回一个`StreamingQueryProgress`对象，它包含流最近一次触发的进度的所有信息：处理的什么数据、处理率（processing rates）、延迟等等。`streamingQuery.recentProgress` 返回最近的几条进度信息。

此外，`streamingQuery.status()` 返回一个`StreamingQueryStatus` 对象，它包含的是关于查询当前执行（immediately doing）的信息——触发是否活跃，是否正在处理数据，等等。

```scala
val query: StreamingQuery = ...

println(query.lastProgress)

/* Will print something like the following.

{
  "id" : "ce011fdc-8762-4dcb-84eb-a77333e28109",
  "runId" : "88e2ff94-ede0-45a8-b687-6316fbef529a",
  "name" : "MyQuery",
  "timestamp" : "2016-12-14T18:45:24.873Z",
  "numInputRows" : 10,
  "inputRowsPerSecond" : 120.0,
  "processedRowsPerSecond" : 200.0,
  "durationMs" : {
    "triggerExecution" : 3,
    "getOffset" : 2
  },
  "eventTime" : {
    "watermark" : "2016-12-14T18:45:24.873Z"
  },
  "stateOperators" : [ ],
  "sources" : [ {
    "description" : "KafkaSource[Subscribe[topic-0]]",
    "startOffset" : {
      "topic-0" : {
        "2" : 0,
        "4" : 1,
        "1" : 1,
        "3" : 1,
        "0" : 1
      }
    },
    "endOffset" : {
      "topic-0" : {
        "2" : 0,
        "4" : 115,
        "1" : 134,
        "3" : 21,
        "0" : 534
      }
    },
    "numInputRows" : 10,
    "inputRowsPerSecond" : 120.0,
    "processedRowsPerSecond" : 200.0
  } ],
  "sink" : {
    "description" : "MemorySink"
  }
}
*/


println(query.status)

/*  Will print something like the following.
{
  "message" : "Waiting for data to arrive",
  "isDataAvailable" : false,
  "isTriggerActive" : false
}
*/
```

##### 5.2、异步API

通过添加`StreamingQueryListener`可以异步地监控与`SparkSession`关联的所有查询。使用`sparkSession.streams.attachListener()`添加自定义`StreamingQueryListener` 后，当查询启动、停止或产生新的进度时，将会得到反馈。

```scala
val spark: SparkSession = ...

spark.streams.addListener(new StreamingQueryListener() {
    override def onQueryStarted(queryStarted: QueryStartedEvent): Unit = {
        println("Query started: " + queryStarted.id)
    }
    override def onQueryTerminated(queryTerminated: QueryTerminatedEvent): Unit = {
        println("Query terminated: " + queryTerminated.id)
    }
    override def onQueryProgress(queryProgress: QueryProgressEvent): Unit = {
        println("Query made progress: " + queryProgress.progress)
    }
})
```

### 使用Checkpointing从故障中恢复

使用checkpointing和write ahead logs，在故障或故意的关闭时，可以恢复之前查询的进度和状态，并且继续执行。可以为查询配置一个checkpoint位置，查询会把所有的进度信息（即，每次触发处理的偏移量范围）和运行中的聚合（比如，之前例子中的单词计数）保存到checkpoint位置。checkpoint 位置必须是一个兼容HDFS的文件系统，当查询启动时要在`DataStreamWriter`中的选项中设置 checkpoint 位置。

```scala
aggDF
  .writeStream
  .outputMode("complete")
  .option("checkpointLocation", "path/to/HDFS/dir")
  .format("memory")
  .start()
```

`checkpointLocation` 是一个目录，如果 `option("checkpointLocation", "hdfs://nameservice/user/hdfs/checkpoints/rcevf")`指定目录不存在，Spark 会自动创建目录。