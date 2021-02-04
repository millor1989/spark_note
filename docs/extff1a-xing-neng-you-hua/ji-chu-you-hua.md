### 开发调优

开发优化包括：RDD lineage设计、算子的合理使用、特殊操作的优化

##### 1、避免创建重复的RDD

RDD lineage：RDD的血缘关系；由某个RDD通过各种算子操作得到一系列RDD的这一过程，这些RDD直间的关系。

对于同一份数据，只应该创建一个RDD；对于同一份数据创建多个RDD会增加作业的开销。

##### 2、尽可能复用同一个RDD

在对来自同一个RDD的不同部分数据时，要尽可能的复用一个RDD。

##### 3、对多次使用的RDD进行持久化

Spark处理RDD的默认流程是：每次对RDD执行算子操作时，都会先从RDD的源头计算，然后再执行算子。而将RDD持久化后，对这个RDD执行算子时会从持久化的RDD来执行算子，而不用从头计算RDD。

默认的持久化级别是`MEMORY_ONLY`。可选的持久化级别包括`MEMORY_AND_DISK`、`MEMORY_ONLY_SER`、`MEMORY_AND_DISK_SER`，通常不建议使用`DISK_ONLY`和后缀为`_2`的持久化级别，基于硬盘的持久化读写很慢，性能可能还不如重新计算RDD；而后缀为`_2`的级别必须将所有数据都复制一份副本发送到其它节点上，数据复制和网络传输会加大开销，除非作业要求高可用性，否则不建议使用。

##### 4、尽量避免使用shuffle类型的算子

shuffle（混洗），可能会引起大量数据的网络传输和硬盘读写，是Spark作业中最消耗性能的过程。要避免使用reduceByKey、join、distinct、repartition等shuffle算子，尽量使用map类的非shuffle算子。没有shuffle算子或者少量的shuffle算子，可以减少不必要的性能开销。

使用broadcast将一个数据量较小的RDD作为广播变量，执行map侧操作。

##### 5、使用map-side预聚合的shuffle操作

如果一定要使用shuffle操作，无法使用map类算子替代，尽量使用可以map-side预聚合的算子。

map-side预聚合就是在每个节点本地对相同的key进行一次聚合操作，map-side预聚合之后每个节点本地只会有一条相同的key，那么将减少网络传输和硬盘读写的数据量。

建议使用reduceByKey或者aggregateByKey替换groupByKey算子，groupByKey不会进行预聚合，前两者会对每个节点本地的相同key进行预聚合。

##### 6、使用高性能的算子

- 使用reduceByKey/aggregateByKey替代groupByKey

- 使用mapPartitions替代普通map

  mapPartitions类算子一次调用会处理一个partition的数据，而不是一次函数调用处理一条，性能较高。但是如果内存不够一次处理一个partition，可能会造成OMM的异常。所以此处要权衡。

- 使用foreachPartitions替代foreach

  原理类似于“使用mapPartitions替代普通map”

- 使用filter之后进行coalesce操作

  如果对一个RDD的filter算子过滤掉RDD中的较多数据，建议使用coalesce算子，将RDD数据压缩到更少的partitions中，减少RDD的partitions数量。过多的partition可能会导致过多的task，进而造成资源浪费，也可能使作业变慢。

- 使用repartitionAndSortWithinPartitions替代repartition与sort类操作

  repartitionAndSortWithinPartitions是Spark官方推荐的算子，如果在repartition重分区后还要进行排序，建议使用repartitionAndSortWithinPartitions算子，它可以一边进行repartition这一shuffle操作，一边排序，比先repartition再sort性能要高。

##### 7、广播大变量

如果要在算子函数中使用外部变量（尤其是大变量，比如100M以上的大集合），此时应该使用广播broadcast来提升性能。

当算子函数中使用外部变量时，默认情况下Spark会将该变量复制多个副本，通过网络传输到task中，此时每个task都有一个变量副本。如果变量较大则会造成很大的网络和内存开销。而广播后的变量会保证每个Executor内存中只驻留一个变量副本，Executor中的task执行时共享该副本，从而大大减少网络和内存开销。

##### 8、使用Kryo序列化优化序列化性能

Spark涉及序列化的三个地方：

- 算子函数使用外部变量时，变量会被序列化后进行网络传输
- 将自定义的类型作为RDD的泛型类型时，所有自定义类型对象都会进行序列化。这种情况下要求自定义的类型必须实现Serializable接口
- 使用序列化的持久化级别时

Spark会将RDD中的每个partition都序列化为一个大的字节数组。

这三种情况都可以使用Kryo序列化。Spark默认使用Java的序列化机制，Kryo要求注册所有需要序列化的自定义类型。

##### 9、优化数据结构

Java中，三种比较耗内存的数据类型：

- 对象，Java对象有对象头、引用等额外信息
- 字符串，每个字符串都有一个字符数组以及长度等额外信息
- 集合类型，比如HashMap、LinkedList等，集合类型内部通常会有一些内部类来封装集合元素，比如Map.Entry。

Spark建议，在Spark编码中，特别是算子函数中，尽量不要使用以上三种数据结构，尽量使用字符串代替对象，使用原始类型代替字符串，使用数组代替集合；这样能够减少内存占用，从而降低GC频率提高性能。

这点完全做到很不容易，要综合考虑代码的可读性、可维护性，在合适的时候使用占内存较少的数据结构。



### 资源优化

###### Spark作业的基本运行原理

![](/assets/1f1ddad5.png)

提交Spark作业后，会启动一个Driver，根据部署模式（deploy-mode）Driver可能在本地启动，也可能在集群中某个工作节点启动。根据配置的参数，Driver会占用一定数量的内存和CPU核心，Driver会向集群管理器申请Spark作业需要使用的资源（executor进程），集群管理器根据Spark作业的资源参数设置，在工作节点启动一定数量的executor进程，每个executor进程占用一定数量的内存和CPU核心。

资源分配完成后，Driver进程开始调度和执行作业代码，根据代码将作业划分为多个stage，每个stage包含一批task，执行一部分的代码；这些task被分配到各个executor进程中执行。task是最小的计算单元，负责执行相同的任务，只是每个task处理的数据不同。一个stage的所有task执行完毕后，会在各个节点的本地硬盘写入计算的中间结果，Driver则调度运行下一个stage。下一个stage的输入就是上一个stage的输出。如此循环，直到作业执行完毕。

Spark根据shuffle算子划分stage，在shuffle算子处划出一条stage边界。因此一个stage刚开始执行时，它的每个task可能都会从上一个stage的task所在的节点通过网络获取各自需要处理的key，将拉取到的相同的key使用算子函数进行处理，这一过程就是shuffle。

如果有cache/persist操作，会根据持久化级别，每个task计算出来的数据会被保存到executor进程的内存或者所在节点的硬盘。

executor内存主要分为三部分：1、task运行代码使用的，默认占executor内存的20%；2、task通过shuffle拉取上一个stage的task的输出后，进行聚合操作时使用，默认占executor内存的20%；3、RDD持久化使用的内存，默认占executor内存的60%。

task的执行速度和每个executor进程的CPU核心数有直接关系。一个CPU核心同一时间只能执行一个线程，而每个executor进程分配到的多个task都是以每个task一个线程的方式，多线程并行运行的。如果CPU核心数量足够，task数量合理，那么通常可以快速高效地完成这些task。

#### 资源参数调优

##### 1、num-executors

executor进程数，集群根据这个参数启动响应的executor进程。如果不设置，只会给作业分配少量的executor进程。设置太少不能充分利用集群资源，太多则可能大部分队列无法给予足够的资源。

##### 2、executor-memory

每个executor进程的内存。要避免占用过多总内存，导致其它任务无法运行。`num-executors`*`executor-memory`最好不要超过队列总内存的1/3 ~ 1/2。

##### 3、executor-cores

每个executor的CPU核心数量，决定了每个executor进程并行执行task的能力。一个CPU核心同时只能执行一个task，executor进程的CPU核心数越多，越能够快速地执行完分配给它的所有task。

`num-executors`*`executor-cores`最好不要超过队列总CPU核心数的1/3 ~ 1/2。

##### 4、driver-memory

Driver的内存。通常Driver内存不用设置，或者1G就足够。但是，如果要使用collect算子将RDD的数据全部拉取到Driver上进行处理，那么必须确保Driver的内存足够大，否则会有OOM的问题。

##### 5、spark.default.parallelism

每个stage默认的task数量。**极为重要**。

如果不设置这个参数，Spark会根据底层HDFS的block数量来设置task的数量，默认一个HDFS block对应一个task。一般情况下，Spark默认设置的task数量是偏少的，task数量偏少，那么前面的executor参数设置可能会达不到预期的效果。

Spark官方建议的设置是，该参数为`num-executors`*`executor-cores`的2 ~ 3倍。这样能够充分利用Spark集群的资源。

##### 6、spark.storage.memoryFraction

RDD持久化数据在executor内存中能占的比例，默认0.6；即持久化到executor内存的数据量至多只能占到executor总内存60%，如果超过，那么会根据设定的持久化级别溢出到硬盘，或者不会持久化。

如果Spark作业中有较多的RDD持久化操作，该参数可以适当增加；但是，如果Spark作业中shuffle操作比较多，持久化比较少，该参数可以适当降低；另外，如果作业执行过程中频繁GC（通过spark web ui可以观察到作业GC耗时），则意味着task执行代码内存不够，同样可以适当降低这个参数值。

##### 7、spark.shuffle.memoryFraction

shuffle过程中一个task拉取到上一个stage的task的输出后，进行聚合操作时能够使用的executor内存比例，默认0.2。如果shuffle进行聚合时使用的内存超过20%，那么多出的数据会溢出到硬盘，此时会极大降低性能。

如果RDD持久化较少，shuffle操作较多，建议将该参数适当增加；如果作业执行过程中频繁GC，意味着task执行作业代码内存不够，可以适当调低这个参数值。