### Resilient Distributed Datasets\(RDDs\)

Spark围绕着RDD的概念，RDD是一个元素的容错集合，可以进行并行操作。创建RDDs的两种方式：并行化（_parallelizing_）驱动程序中存在的集合，或者通过外部存储系统的数据集创建，比如共享文件系统，HDFS、HBase或者其它支持Hadoop InputFormat的数据源。

#### 并行化驱动程序集合创建RDD

对驱动程序中存在的集合（Scala Seq）使用`SparkContext`的`parallelized`函数。集合的元素会被复制以构成一个可以进行并操作的分布式数据集。

```scala
val data = Array(1, 2, 3, 4, 5)
val distData = sc.parallelize(data)
```

一个并行化集合的重要参数是`partitions`，指定把数据集切分成的分区的数量，比如`sc.parallelize(data, 10)`指定分区数量。Spark会为集群中的每个分区运行一个task。通常分区数量为集群CPU（应用可用CPU）数量的2~4倍。一般Spark会自动根据集群设置分区数量。

#### 从外部数据集创建RDD

`SparkContext`的`textFile`函数可以从Text文件创建RDD，这个方法使用文件的URI（机器的本地路径或者`hdfs://`，`s3n://`，等）作为参数并且把文件读取为行的集合。

```scala
val distFile = sc.textFile("data.txt")
```

读取文件的注意事项：

* 如果使用的路径时本地文件系统，必须确保其它工作节点的相同路径文件是可以访问的。或者把文件复制到所有的工作节点，或者使用基于网络挂载的共享文件系统
* Spark的所有的基于文件的输入方法，包括`textFile`都支持目录、压缩文件，也支持通配符。比如：`textFile("/my/directory")`, `textFile("/my/directory/*.txt")`, 和 `textFile("/my/directory/*.gz")`。
* `textFile`方法可以通过可选参数指定文件的分区数量。默认情况下，Spark为文件的每个block（HDFS默认block大小128M）创建一个分区，但是可以通过设置参数来获得更多的分区数量。但是，分区数不能比block数小。

除了文本文件，Spark的Scala API也支持几种其它的数据格式：

* `SparkContext.wholeTextFiles`：读取包含多个文本文件的目录，将每个文件以`(filename, content)`对的格式返回；可以通过可选第二参数来指定分区。与`textFile`方法不同的是，`textFile`方法返回文件中的每行内容作为一条记录。
* `SparkContext.sequenceFile[K,V]`：用于SequenceFiles，K和V分别对应文件的键值类型，它们的类型应该是Hadoop的`Writable`借口的子类型。另外，对某些常用的`Writable`类型，Spark允许使用原生类型，比如`sequenceFile[Int, String]`会被自动转换为`IntWritable`和`Text`。
* 对于其它Hadoop输入格式，可以使用`SparkContext.hadoopRDD`方法，可以使用任意的`JobConf`和输入格式类型、键值类型。`SparkContext.newAPIHadoopRDD`是基于新的MapReduce API（`org.apache.hadoop.mapreduce`）的对应方法。
* `RDD.saveAsObjectFile`和`SparkContext.objectFile`支持以序列化的Java对象组成的简单格式保存RDD。但是，这种方式效率不如像AVRO这样的专用格式。

#### RDD操作

两种：转换操作（transformations）和行为操作（actions）。

转换操作是懒惰的（lazy），只有被行为操作触发时才会进行计算。这种设计使Spark运行更加高效，比如，`map`操作返回的结果会被用于`reduce`，并且只会把`reduce`的结果返回给driver，而比较大`map`结果不会返回给driver。

默认情况下，每当行为操作用到转换操作的结果RDD时，转换操作结果RDD都会重新计算；但是，可以通过持久化使RDD的下次使用更快。

##### RDD基本操作

```scala
val lines = sc.textFile("data.txt")
val lineLengths = lines.map(s => s.length)
val totalLength = lineLengths.reduce((a, b) => a + b)
```

RDD持久化：

```scala
lineLengths.persist()
```

##### 传递函数

传递函数的**推荐方式**：

* 匿名函数语法（Anonymous function syntax），代码更简洁。

* 全局单例对象（global singleton object）中的静态方法。比如：

  ```scala
  object MyFunctions {
    def func1(s: String): String = { ... }
  }

  myRdd.map(MyFunctions.func1)
  ```

  也可以把类实例（class instance，不同于单例对象singleton object）中方法的引用进行传递，但是需要对象包含方法所在的类。比如：

  ```scala
  class MyClass {
    def func1(s: String): String = { ... }
    def doStuff(rdd: RDD[String]): RDD[String] = { rdd.map(func1) }
  }
  ```

  如果创建了一个新的`MyClass`实例，并调用它的`doStuff`，它里面的`map`就会引用这个`MyClass`实例的`func1`方法，所以就需要把整个对象传递到集群。这与写法`rdd.map(x => this.func1(x))`类似。

  类似地，访问外部对象的属性也会引用整个外部对象：

  ```scala
  class MyClass {
    val field = "Hello"
    def doStuff(rdd: RDD[String]): RDD[String] = { rdd.map(x => field + x) }
  }
  ```

  这段代码和`rdd.map(x => this.field + x)`等价，这里引用整个`this`。为了避免引用整个`this`对象，最简单的方式是把`field`复制到一个局部变量（local variable）：

  ```scala
  def doStuff(rdd: RDD[String]): RDD[String] = {
    val field_ = this.field
    rdd.map(x => field_ + x)
  }
  ```

#### 理解闭包（understanding closures）

在集群执行代码时变量的生命周期和作用域（life cycle and scope）。RDD操作对作用域外的变量进行修改是一个常见困惑。

如下，初衷为累加RDD元素的代码，会因为是否运行于同一个JVM中而有不同的结果。

```scala
var counter = 0
var rdd = sc.parallelize(data)

// Wrong: Don't do this!!
rdd.foreach(x => counter += x)

println("Counter value: " + counter)
```

为了执行jobs，Spark会把对RDD的操作拆分为tasks，每个executor执行一个task。执行前，Spark会计算task的**closure**（closure是executor执行RDD运算所需要的变量和方法），closure会被序列化并且发送到每个executor。因为closure中的变量会被拷贝，所以`foreach`函数中引用的`counter`不再是driver节点上的`counter`。driver节点上的内存中仍然有一个`counter`，但是它对executors是不可见的，executors只能见到序列化的closure中`counter`的拷贝。由于对`counter`的所有操作均针对的是序列化的closure，`counter`最终的结果仍然是0。

在本地模式中，某些情况下`foreach`操作会事实上会和driver在同一个JVM中执行并且会使用相同的原始的`counter`，因而可能会更新`counter`的值。

要确保这些场景下的行为执行，需要使用累加器`Accumulator`。Spark中累加器专门用于在集群中，当操作的执行被分散到各个工作节点时，提供一种安全的更新一个变量的机制。

通常，closures——像循环结构或者本地定义的方法，不应该去改动某些全局的状态（global state）。Spark既没有定义也不保证对closures以外引用的对象的改动。

##### RDD元素的输出

集群模式中，每个executor的输出都输出到该executor的`stdout`，而不是driver的输出。

#### 键值对操作

尽管大多数Spark操作可以作用于包含任何类型对象的RDDs，但是某些操作只作用于键值对组成的RDDs。最常见的是分布式的混洗操作，比如按照某个key对元素进行grouping或者aggregating。

在Scala中，键值对操作对于二元组`Tuple2`自动可用。键值对操作包含在`PairRDDFunctions`类中。

```scala
val lines = sc.textFile("data.txt")
val pairs = lines.map(s => (s, 1))
val counts = pairs.reduceByKey((a, b) => a + b)
```

在键值对操作中，如果把自定义对象作为键，必须确保重写了`equals()`和`hashCode()`方法。

#### RDD转换操作

map、filter、flatmap、mapPartitions、mapPartitionsWithIndex、sample、union、intersection、distinct、groupByKey、reduceByKey、aggregateByKey、sortByKey、join、cogroup、cartesian、pipe、coalesce、repartition、repartitionAndSortWithinPartitions

#### RDD行动操作

reduce、collect、count、first、take、takeSample、takeOrdered、saveAsTextFile、saveAsSequenceFile、saveAsObjectFile、countByKey、foreach

Spark RDD API也有些异步版本的行动操作，比如对应`foreach`的`foreachAsync`，返回给调用者的是一个`FutureAction`而不是阻塞直到行为操作完成。

#### 混洗操作

Spark中的某些操作会触发混洗（shuffle）事件。混洗是Spark重新分发（re-distributing）数据的机制，它使数据跨分区进行不同的分组。

##### 混洗背景

对于`reduceByKey`操作，它产生一个新的RDD，在这个RDD中所有具有相同key的值被组合到一个tuple——key和对这个key相对应的所有值执行reduce函数的结果。问题是，某个key对应的所有数据不一定保存在同一个分区（或者机器）中，但是必须将它们联合起来才能计算出结果。

在Spark中，数据不会因为某个操作而跨分区分布到一个地方。在运算中，一个task只会操作一个分区，所以，为了某个`reduceByKey`的reduce task的执行需要获取所有的数据，Spark需要进行全部到全部（all-to-all）的操作。Spark必须从全部的分区中找出对应全部key的全部值，并且然后跨分区把数据组合到一起来计算出某个key对应的最终结果——这就是**混洗**。

尽管混洗后的数据的每个分区的元素set是确定的，分区也是确定的，但是这些元素的顺序不是确定的。如果要进行排序，可以进行如下操作：

* `mapPartitions`来排序每个分区，比如使用`.sorted`
* `repartitionAndSortWithinPartitions` 来在重新分区的同时高效地分区内排序
* `sortBy` 来生成一个全局排序的RDD

会引起混洗的操作：**repartition**操作，比如`repartition`和`coalesce`；**ByKey**操作（除了counting），比如`groupByKey`和`reduceByKey`；**join**操作比如`cogroup`和`join`。

##### 性能影响

混洗是一个高开销（expensive）的操作，它包括硬盘I/O，数据序列化，和网络I/O。为了组织数据进行混洗，Spark生成一系列的tasks——map tasks来组织数据，reduce tasks来聚合数据（此处的map、reduce是MapReduce的命名法，不是指Spark的map、reduce操作）。

在内部，map tasks的结果保存在内存中直到内存不足，然后这些结果基于目标分区进行排序并写入到一个文件。在reduce侧，tasks读取相对应的排序过的blocks。

某些混洗操作会消耗大量的堆内存，因为在传输数据之前，它们使用在内存的数据结构来组织数据。比如`reduceByKey`和`aggregateByKey`在map侧创建这些数据结构，`'ByKey`操作在reduce侧创建这些数据结构。当内存不足时，Spark会将这些数据结构（these tables）溢出到硬盘，从而导致额外的硬盘I/O并且增加GC。

混洗也会在硬盘上生成大量的中间文件。对于Spark 1.3，这些文件在对应的RDDs不再被使用并且被垃圾回收之前会一直被保存。这样当lineage重新计算的时候就不必重新创建这些混洗文件。如果应用保留了对这些RDDs的引用或者GC起作用不频繁，可能在很长一段时间之后才会进行垃圾回收。这也意味着长时间运行（long-running）的Spark jobs可能会消耗大量的硬盘空间。这些临时存储目录在配置Spark Context时通过`spark.local.dir`配置参数进行配置。

混洗行为调试参考[Spark Configuration Guide的Shuffle Behavior章节](http://spark.apache.org/docs/2.2.0/configuration.html#shuffle-behavior)

#### RDD持久化

Spark的一个重要能力是跨操作持久化（缓存，persisting or cacheing）数据集到内存。持久化RDD时，每个节点将在该节点上计算的该RDD的所有分区保存到内存，并且在其它对该数据集（或者从这个数据集派生的数据集）的操作中重用。这使得未来的操作更加快速（通常快10倍以上）。缓存是迭代算法和快速交互使用的一个关键工具。

通过`persist()` 或`cache()`方法可以将RDD标记为持久化的。在对RDD进行第一次行为操作时，RDD会被保存到节点的内存中。Spark的持久化时容错的（fault-tolerant）——如果RDD的任意分区丢失，都会根据创建它的操作自动地重新进行计算。

默认的存储级别是`MEMORY_ONLY`。存储级别的选择是内存使用与CPU效率直接的权衡，推荐的选择依据是：

* 如果内存足以存储RDD，就用`MEMORY_ONLY`。这是CPU效率最高的选项。
* 内存不足，则使用`MEMORY_ONLY_SER` 并选择快速的序列化库以保证对象存储节省空间，这种方式也足够快。
* 除非计算RDD的开销非常高或者过滤掉了巨多的数据量，否则不要使用硬盘持久化级别；因为重新计算RDD都可能比从硬盘读取RDD快速。
* 如果需要从错误中恢复（比如，使用Spark来提供web应用的请求）才需要使用备份的存储级别。所有的存储级别都通过重新计算丢失数据提供了容错，但是备份的存储级别能够在不等待重新计算丢失数据的情况下使得基于这个RDD的tasks继续运行。

在混洗操作中，即使不调用`persist`，Spark也会自动地持久化某些中间数据。这样能够避免在混洗过程中由于某个节点的故障而重新计算整个输入。推荐在需要重用RDD时进行`persist`。

##### 移除数据

Spark会自动监控每个节点的缓存使用，并且会根据LRU（least-recently-use） fashion删除旧的数据分区。可以使用`RDD.unpersist()` 方法手动移除RDD缓存。

#### 共享变量（Shared Variables）

一般，当给在远程集群上运行的Spark操作（比如`map`或者`reduce`）传递函数时，函数是工作在函数使用所有的变量的拷贝上的。这些变量被复制到每个机器，远程机器上这些变量拷贝的更新不会回传到驱动程序（driver program）。为通用的、读-写共享变量提供支持是不够高效的。但是，Spark针对两种使用场景提供了两种限制类型共享变量：广播变量（broadcast variables）和累加器（accumulators）。

##### 广播变量

广播变量使得使用者可以把一个只读的变量缓存在每个机器上而不是通过tasks传递变量的拷贝。它以一个高效的方式，为每个节点提供一个大的输入数据集拷贝。Spark也尝试用高效的广播算法来分发广播变量从而减少通信开销。

Spark行为操作通过被分布式的“混洗”（distributed “shuffle”）操作分割的一系列stages来执行。在每个stage内部，Spark自动地广播tasks需要的共用数据。这种方式广播的数据以序列化的格式进行缓存，并且在运行每个task之前进行反序列化。这意味着，只有跨越多个stages的tasks需要共用数据，或者以非序列化格式缓存的数据很重要时，才需要明确地创建广播变量。

广播变量的创建和使用：

```scala
scala> val broadcastVar = sc.broadcast(Array(1, 2, 3))
broadcastVar: org.apache.spark.broadcast.Broadcast[Array[Int]] = Broadcast(0)

scala> broadcastVar.value
res0: Array[Int] = Array(1, 2, 3)
```

创建完广播变量后，应该用广播变量代替原始变量来使用。另外，不能去修改原始变量，以确保全部节点获得的广播变量的值相同（比如，变量被广播到某个节点比较晚）。

##### 累加器

累加器时是只能通过关联的和交换的（associative and commutative）操作进行“增加”，并且高效地支持并行操作的变量。它可以用了实现计数器（counters）或者累加。Spark原生地支持数值类型的累加器，使用者可以增加对其它类型的支持。

用户可以创建命名的或者未命名的累加器。命名的累加器会显示在修改这个累加器的stage的Web UI中，在“Tasks”表中还会显示task修改的每个累加器。

![Accumulators in the Spark UI](/assets/spark-webui-accumulators.png)

通过调用`SparkContext.longAccumulator()` 或 `SparkContext.doubleAccumulator()`可分别创建用于累加Long或者Double类型值的数值累加器。集群中运行的tasks只能使用`add`方法增加累加器的值，而不能读取累加器的值。只有在driver程序中才能使用`value`方法读取累加器的值。

```scala
scala> val accum = sc.longAccumulator("My Accumulator")
accum: org.apache.spark.util.LongAccumulator = LongAccumulator(id: 0, name: Some(My Accumulator), value: 0)

scala> sc.parallelize(Array(1, 2, 3, 4)).foreach(x => accum.add(x))
...
10/09/29 18:41:08 INFO SparkContext: Tasks finished in 0.317106 s

scala> accum.value
res2: Long = 10
```

除了使用内置方法创建累加器，还可以通过继承`AccumulatorV2`来自定义累加器。必须要重写的方法有：`reset`——重置累加器为0；`add`——将累加器加上一个值；`merge`——合并另一个相同类型的累加器到当前累加器；还有`copy`、`isZero`、`value`也都需要进行重写，具体参考[官方API](http://spark.apache.org/docs/2.2.0/api/scala/index.html#org.apache.spark.util.AccumulatorV2)。代码示例：

```scala
class VectorAccumulatorV2 extends AccumulatorV2[MyVector, MyVector] {

  private val myVector: MyVector = MyVector.createZeroVector

  def reset(): Unit = {
    myVector.reset()
  }

  def add(v: MyVector): Unit = {
    myVector.add(v)
  }
  ...
}

// Then, create an Accumulator of this type:
val myVectorAcc = new VectorAccumulatorV2
// Then, register it into spark context:
sc.register(myVectorAcc, "MyVectorAcc1")
```

注意，自定义AccumulatorV2时，结果类型可以和被加的元素的类型不同。

对于在行动操作中的累加器更新，Spark可以确保每个task对累加器的更新只会应用一次，例如，重新启动tasks不会再次更新累加器。但是，在转换操作中，如果tasks或者job stages重新执行，那么task对累加器的更新可能会应用多余一次。

累加器不改变Spark懒求值（lazy evaluation）模型。如果被RDD的一个操作更新，累加器只会在RDD被执行运算时才会更新。所以，懒的转换操作中的累加器不保证一定会被更新。比如：

```scala
val accum = sc.longAccumulator
data.map { x => accum.add(x); x }
// Here, accum is still 0 because no actions have caused the map operation to be computed.
```



