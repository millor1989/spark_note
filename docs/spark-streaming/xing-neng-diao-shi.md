Spark Streaming性能调试需要考虑的两方面：

- 高效地使用集群资源减少每个批次的数据的处理时间
- 设置恰当的批次大小，以便数据批次可以在被接收的同时被处理掉（即，数据处理的速度与数据接收的速度一致）

#### 1、减少批次处理时间

##### 1.1、数据接收并行度

通过网络（比如，Kafka、Flume、socket等）接收数据，需要把数据反序列化并保存在Spark中。如果数据接收是系统的瓶颈，可以考虑并行接收数据。注意，每个输入DStream只会创建一个receiver（在一个工作节点上运行）并且只接收一个数据流。接收多个数据流可以通过创建多个输入DStreams并让它们从数据源的不同分区接收数据流实现。例如，一个Kafka输入DStream接收两个topics的数据可以改进为两个输入流，每个接收一个topic。这会运行两个receivers，可以并行接收数据，从而增加全局通量。多个DStreams可以合并为一个DStream，然后进行进一步的处理。

```scala
val numStreams = 5
val kafkaStreams = (1 to numStreams).map { i => KafkaUtils.createStream(...) }
val unifiedStream = streamingContext.union(kafkaStreams)
unifiedStream.print()
```

另一个需要考虑的参数是receiver的块间隔（block interval），块间隔有配置参数`spark.streaming.blockInterval`决定。对于大多数的receivers，接收到的数据在保存到Spark的内存中之前，会被整合到数据块中。对于map类的转换操作，每个批次中块的个数决定了处理接收到的数据的tasks的数量。每个receiver每个批次tasks的数量都是相近的（批次间隔/块间隔）。比如，200ms块间隔没2s会产生10个tasks。如果tasks数量太少，处理数据的效率就低。对于一定的批次间隔，减少块间隔可以增加tasks的数量。但是，推荐的最小的块间隔是50ms，太小的话，task的启动开销可能会是一个问题。

使用多个输入流的替代方案是对输入数据流进行重分区（使用`inputStream.repartition(<number of partitions>)`）。这会在对接收到的数据批次进行处理前，把数据在指定的数量的机器上进行分发。

##### 1.2、数据处理的并行度

对于分布式的reduce操作，比如`reduceByKey`和`reduceByKeyAndWindow`，默认的并行tasks的数量由`spark.default.parallelism`配置属性控制。

##### 1.3、数据序列化

可以通过调试序列化格式减少数据序列化的开销。对于流，有两类数据被序列化：

- **输入数据**：默认情况下，通过Receivers接收到的输入数据使用`StroageLevel.MEMORY_AND_DISK_SER_2`存储级别保存在executor的内存中。
- **流操作产生的持久化RDDs**：流操作产生的RDDs可能持久化在内存中。例如，窗口操作把数据持久化到内存，因为这些数据需要被多次处理。但是与Spark Core默认的存储级别`StorageLevel.MEMORY_ONLY`不同，此处默认使用的是`StorageLevel.MEMORY_ONLY_SER`

这两种情况下，使用Kryo序列化可以减少CPU和内存开销。

在特定情况下，如果留给Streaming应用的数据量不是很大，把数据保存为非序列化的对象而不到之过大的GC开销也是可能的。例如，如果使用的是几秒钟的批次间隔，并且没有窗口操作，可以根据情况通过明确地设置非序列化的存储级别来持久化数据。这会降低序列化引起的CPU开销，会在不引起太大GC开销的情况下提升性能。

##### 1.4、Task启动开销

如果每秒启动的tasks的数量很高（比如，每秒50个或更多），那么把任务发送给slaves的开销可能很大，并且使获得亚秒级的延迟变得困难。通过如下改变可以减少这个开销：

- **Execution mode**：用Standalone模式或者粗粒度（coarse-grained）Mesos模式运行Spark可以比细粒度（fine-grained）Mesos获得更佳的启动时间。

这个改变可以使批处理时间减少100毫秒，从而使亚秒批次处理成为可能（viable）。

#### 2、设置合适的批次间隔

为了使集群上运行的Spark Streaming应用稳定，系统应该能够在接收到数据后尽快地进行处理。换句话说，数据批次应该在生成后尽快地进行处理。即，在Streaming web UI中，批次处理时间应该比批次间隔小。

#### 3、内存调试

Spark Streaming应用需要的内存大小强烈依赖于使用的转换操作的类型。比如，如果要使用一个基于最近10分钟数据的窗口操作，那么集群应该有足够的内存来10分钟的数据。或者，如果要对有大量key的数据使用`updateStateByKey`，那么需要的内存会很多。相反地，如果只是进行简单的map-filter-store操作，需要的内存就会很少。

通常，因为通过Receiver接收到的数据使用`StorageLevel.MEMORY_AND_DISK_SER_2`保存，超过内存限制的数据会溢出到磁盘。这可能会降低流应用的性能，因此需要确保为流应用提供了充足的内存。

内存调试的另一个方面是垃圾回收。因为流应用需要低延时，所以不希望JVM的垃圾回收引起很大的停顿。

可以用来调试内存使用和GC开销地参数：

- **DStreams的持久化级别**：输入数据和RDD默认以序列化的字节进行持久化。与非序列化的持久化相比，这能够减少内存使用和GC开销。开启Kryo序列化能进一步减小序列化尺寸和内存使用。使用压缩可以再进一步的减少内存使用，但是增加CPU时间成本。
- **清除旧的数据**：默认情况下，所有的输入数据和DStreams转换操作产生的持久化RDDs会被自动清除。Spark Streaming会基于使用的转换操作决定合适清除数据。例如，如果使用一个10分钟的窗口操作，Spark Streaming会保留最近10分钟的数据，并清除较早的数据。通过设置`streamingContext.remember`可以将数据保存更长的时间。
- **CMS垃圾回收器**：强烈推荐使用CMS（concurrent mark-and-sweep） GC以将GC引发的暂停维持在较低水平。即使CMS GC会减少系统全局处理通量，为了维持一致的批处理时间还是推荐使用它。要确保驱动器（使用`spark-submit`的`--driver-java-options`选项设置）和executors（使用Spark配置`spark.executor.extraJavaOptions`设置）都设置为CMS GC。
- 其它建议：
  - 使用`OFF_HEAP`存储级别持久化RDDs
  - 使用更多具有更小堆空间的executors。这会减少每个JVM堆中的GC压力。

**需要牢记的重要事项**：

- 一个DStream只和一个receiver关联。如果要实现并行读入数据，要创建多个receivers也即创建多个DStreams。一个Receiver运行在一个executor中，需要占用一个核心。要确保receiver占用核心后有足够的核心来进行数据处理，也即`spark.cores.max`要把receiver占用的核心考虑在内。receivers被以轮换的方式分配到executors。
- 当从流数据源接收数据时，receiver创建数据块。每隔一段块间隔产生一个新的数据块。这些数据块被当前executor的BlockManager分发给其它executors的BlockManagers。然后，driver上运行的Network input Tracker会被通知block的位置，以便后续进行处理。
- driver会为一个批次间隔中产生的数据块创建一个RDD。批次间隔中产生的数据块是RDD的分区。每个分区对应Spark中的一个task。块间隔等于批次间隔则意味着只创建一个分区并且可能会在本地进行处理。
- 数据块的map tasks会在拥有数据块的executors（一个是接收数据块的executor，一个是对数据块进行备份的executor）上进行处理，除非进行了非本地调度。块间隔越大则块越大。较大的`spark.locality.wait`意味着在块的本地节点进行块处理的几率越大。需要在这两者之间进行权衡，以确保较大的块在本地进行处理。