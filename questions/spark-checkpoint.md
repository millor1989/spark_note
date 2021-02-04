### 持久化

对多次使用的RDD（特别是来源复杂、计算比较耗时、计算链条比较长的）要进行持久化。可以使用cache或者persist进行持久化，前者持久化到内存，后者可以指定持久化级别；当action操作触发RDD的计算后，会对RDD进行持久化；之后，再次用到这个RDD就会从持久化的结果获取，而不是重新计算。

持久化级别：

* **MEMORY\_ONLY**：持久化到内存，不需要序列化和反序列化（避免了相应的开销），不保存副本。需要内存足够，否则可能会导致JVM的内存溢出错误。
* **MEMORY\_ONLY\_SER**：将RDD数据序列化后持久化到内存，此时每个partition仅仅是一个字节数组大大减少了对象数量，减少内存使用。但是有序列化和反序列化开销。也有可能造成JVM的内存溢出错误。
* **MEMORY\_AND\_DISK\_SER**：如果内存紧张或者RDD数据量过多，纯内存的存储方式无法满足，那么就使用这种方式。该方式优先考虑将数据持久化到内存，内存不足则溢出到硬盘。**MEMORY\_AND\_DISK**不进行序列化，进行序列化能够大大节省存储空间。
* **DISK\_ONLY**：不建议使用，完全基于硬盘的持久化性能很差，可能还不如重新计算RDD。
* **\_2**：备份级别，需要将所有数据复制一份副本发送到其它节点上，数据复制和网络传输会导致较大的性能开销，除非是作业的高可用性要求很高，否则不推荐使用。

可以尝试使用“**OFF\_HEAP**”（非堆内存）的方式，该方式将数据持久化到Alluxio（Alluxio是一套基于内存的分布式文件存储系统，曾用名Tachyon）中，而不是executor内存中。**OFF\_HEAP**的优势：允许多个executors共享一个内存池、减少垃圾回收的开销、即使某个executor崩溃，持久化的数据也不会丢失。

此外，Spark持久化具有容错性，持久化的RDD某部分如果丢失，后续使用到时会重新计算并持久化该分区；Spark有一套机制（通过LRU算法）删除果实的持久化数据，所以，在使用持久化的RDD时该RDD的持久化不一定是有效的。可以通过`RDD.unpersist()`方法删除持久化的RDD。

### checkpoint

把RDD进行持久化，虽然快速，但并不是最可靠的。使用checkpoint可以提高容错性和高可用性。

checkpoint机制切断了RDD的lineage（对所有父RDD的引用关系都会被切断），并且把中间RDD保存到hdfs实现容错。

```scala
...
/*首先指定checkpoint目录，一般是hdfs地址*/
spark.sparkContext.setCheckpointDir("hdfs://...")
val interrdd = ...
/*进行checkpoint标记*/
interrdd.checkpoint()
...
```

checkpoint和persist一样是惰性的。`RDD.checkpoint()`只是进行标记（创建`ReliableRDDCheckpointData`，主要维护RDD的checkpoint状态——包括Initialized、CheckpointingInProgress、Checkpointed），这个函数必须放在RDD的action操作之前。对RDD执行第一个action操作时，action操作（所有action操作的主要切入点都是`runjob`）的`runjob`函数的最后一个操作就是`RDD.doCheckpoint()`，通过`RDD.writeRDDToCheckpointDirectory()`将RDD写入指定的checkpoint目录；写完checkpoint数据到hdfs后，会更新checkpoint状态为`Checkpointed`，并调用`RDD.markCheckpointed`斩断RDD的上游依赖。

对一个RDD进行checkpoint时，事实上计算了两次这个RDD：一次是action操作触发checkpoint时，一次是进行checkpoint时（即将RDD保存为文件时，遍历RDD分区并将它们保存）。所以，强烈建议在`RDD.checkpoint()`函数之前对RDD进行持久化。

使用RDD时，首先会查看RDD的依赖关系中是否有checkpointRDD的依赖关系。如果RDD被checkpoint过，则会在计算时触发读取checkpoint文件的过程。

Spark在StreamingContext中使用了checkpoint进行容错；在需要反复迭代计算RDD的机器学习中也使用了checkpoint，比如ALS、Decision Tree等算法需要反复使用RDD，如果RDD数据集特别大cache就没有意义了，需要使用checkpoint。

https://www.cnblogs.com/ourtest/p/10251369.html

[https://issues.apache.org/jira/browse/SPARK-8582](https://issues.apache.org/jira/browse/SPARK-8582)

