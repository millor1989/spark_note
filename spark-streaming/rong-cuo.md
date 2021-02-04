### 容错

Spark Streaming的容错

#### 1、背景

Spark RDD的容错：

1. RDD是一个不可变的、重新运算具有确定性的、分布式数据集。每个RDD都保留了基于容错的数据集用来创建这个RDD的确定性的操作的血缘关系（lineage）。
2. 如果由于某个工作节点的故障导致了一个RDD的某个分区的丢失，可以基于最初的容错的数据集使用操作的血缘关系重新计算丢失的分区。
3. 假设所有的RDD转换操作都是确定性的，即使Spark集群故障，最终转换后的RDD也总是相同的。

Spark操作容错的文件系统（HDFS or S3）中的数据。因此，从容错的数据中产生的RDDs也是容错的。但是，对于Spark Streaming大多数情况下数据都来自于网络，而不是文件系统。为了使所有产生的RDDs达到相同容错性，接收到的数据会在集群中的工作节点的executors中间进行备份（默认的备份因子是2）。这就导致发生故障时系统中有两种数据需要恢复：

1. 接收到并且备份了的数据：这些数据会在某个特定工作节点故障时幸存，因为备份存在于其它节点中的某个上。
2. 接收到并且为了备份已经缓存的数据：因为这些数据没有备份，恢复这些数据的唯一方法是重新从数据源获取。

另外，需要考虑两种类型的故障：

1. 工作节点的故障：任何运行executor的工作节点都会故障，并且这些节点上的所有的内存中数据都会丢失。如果recevier运行在故障的节点上，那么缓存的数据就会丢失。
2. Driver节点的故障：如果运行Spark Streaming应用的driver节点故障，那么SparkContext会丢失，所有的executors和它们的内存中数据都会丢失。

#### 2、定义

通常根据系统处理每条记录的次数来获得流系统的语义。系统在所有可能的操作条件下提供三中类型的保证：

1. 最多一次：每条记录被处理一次或者不处理。
2. 最少一次：每条记录会被处理一次或多次。这种类型比“最多一次”健壮，它确保了不会遗漏数据，但是可能会有重复。
3. 仅仅一次：每条记录处理一次。不遗漏数据也不多次处理数据。这个是最健壮的保证类型。

#### 3、基本语义

笼统地说，任何流处理系统中，处理数据分三步：

1. 接收数据：使用Recevier或其它从数据源接收数据
2. 转换数据：使用DStream和RDD的转换操作对接收到的数据进行转换
3. 输出数据：把最终转换的数据输出到外部系统（文件系统、数据块、报表，等）

如果流应用要做到端到端仅一次处理的保证，那么每一步都要提供仅处理一次的保证，即每条记录接收一次、转换一次、输出一次。就Spark Streaming而言，这些步骤如下：

1. 接收数据：不同输入数据源提供不同的保证，后面讨论。
2. 转换数据：由于RDD提供的保证，所有接收到的数据仅仅处理一次。即使有故障，只要接收到的数据可以访问，最终转换后的RDDs总是内容相同的。
3. 输出数据：默认输出操作保证的是“最少一次”的语义，因为它依赖于输出操作的类型（是否幂等）和下游系统的语义（是否支持事务）。但是用户可以实现自己的事务机制以达到“仅仅一次”的语义。后面更加详细的讨论。

#### 4、接收到的数据的语义

不同输入数据源提供不同的保证

##### 4.1、使用文件

如果所有的输入数据都在像HDFS的这种容错的文件系统中，Spark Streaming总能够从任何的故障中恢复并且处理所有的数据。这种数据源提供了“仅仅一次”的语义，意味着不管发生什么故障所有的数据都只会处理一次。

##### 4.2、使用基于Receiver的数据源

对于基于recevier的数据源，容错的语义则视故障场景和receiver类型而定。有两种类型的receivers：

1. 可靠的Receiver：这类receivers在确保接收到的数据备份完成后通知可靠的数据源。如果这种receiver故障，数据源将收不到已经缓存（未备份）数据的通知。因而，如果重启receiver，数据源会重新发送这些数据，不会有数据因为故障而丢失。
2. 不可靠的Receiver：这些receivers不会发送通知，因而在工作节点或者driver故障时会丢失数据。

基于使用的receivers类型可以得出如下的结论：如果工作节点故障，可靠receiver不会丢失数据，而不可靠receiver已经接收但是没有备份的数据会丢失。如果driver节点故障，所有过去接收到的和备份在内存的数据都会丢失。这会影响有状态的转换操作。

为了避免过去接收到的数据的丢失，Spark 1.2采用了*write ahead logs*，将接收到的数据保存在了容错的存储中。使用可靠数据源并开启*write ahead logs*，将不会丢失数据。在语义方面，这提供了“至少一次”的保证。

总结如下：

| eployment Scenario                                           | Worker Failure                                               | Driver Failure                                               |
| ------------------------------------------------------------ | ------------------------------------------------------------ | ------------------------------------------------------------ |
| *Spark 1.1 or earlier,* OR        *Spark 1.2 or later without write ahead logs* | Buffered data lost with unreliable receivers        Zero data loss with reliable receivers        At-least once semantics | Buffered data lost with unreliable receivers        Past data lost with all receivers        Undefined semantics |
| *Spark 1.2 or later with write ahead logs*                   | Zero data loss with reliable receivers          At-least once semantics | Zero data loss with reliable receivers and files          At-least once semantics |

##### 4.3、使用Kafka Direct API

在Spark 1.3，采用了新的Kafka Direct API，它可以确保Spark Streaming只接收一次Kafka的数据。使用它的同时，如果实现了“仅仅一次”的输出操作，即可达到端到端仅仅一次的保证。详情参考[Kafka集成向导](http://spark.apache.org/docs/2.2.0/streaming-kafka-integration.html)

#### 5、输出操作语义

输出操作（像`foreachRDD`）具有“至少一次”的语义，即，工作节点故障时转换后的数据可能会被多次写到外部系统。尽管对于使用`saveAs***Files`这种保存到文件系统的操作是可以接受的（因为同样数据的的文件会被覆盖），但是为了达到“仅仅一次”的语义还需要一些额外的努力。有两种方法：

- 幂等的更新（Idempotent updates）：多次尝试都写相同的数据。例如，`saveAs***Files`总是把相同的数据写到生成的文件中。

- 事务性的更新（Transactional updates）：所有的更新都是事务性的，以便更新都是原子性地只执行一次。一种实现的方式如下：

  - 使用批次时间（`foreachRDD`中可以使用）和RDD的分区索引来创建一个identifier。这个identifier在流应用中唯一地分配一个blob数据。

  - 使用这个identifier使用这个blob事物性地（即，原子性地、仅仅一次地）更新外部系统。即，如果这个identifier没有被committed，那么原子性地commit这个分区和这个identifier。否则，如果这个identifier已经committed，跳过更新。

    ```scala
    dstream.foreachRDD { (rdd, time) =>
      rdd.foreachPartition { partitionIterator =>
        val partitionId = TaskContext.get.partitionId()
        val uniqueId = generateUniqueId(time.milliseconds, partitionId)
        // use this uniqueId to transactionally commit the data in partitionIterator
      }
    }
    ```

