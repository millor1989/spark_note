### RDD

**RDD**（resilient distributed dataset，弹性分布式数据集），是一个可以进行并行操作的、容错的数据集合。

分布在一个或者多个分区中的数据的集合，可以对它进行容错的并且是弹性的运算。RDD与Scala的集合相比，RDD可以在多个JVMs中进行运算，而Scala集合只能在一个JVM中进行运算。

RDD的特征：

* **Resilient**：弹性，使用RDD lineage graph实现容错，可以重新计算由于节点故障导致的丢失或者损坏的分区
* **Distributed**：分布式，数据保存在集群中的多个节点上
* **Dataset**：分区数据（partitioned data）（可以是基本类型值、也可以是对象值）的集合

![](/assets/spark-rdds.png)

Spark的Scala文档中描述`RDD`是Spark的基本抽象概念，代表一个不可变的、可以进行并行操作的分区的元素集合。

Spark计算框架中的`RDD`还有以下特点：

* 在内存（In-Memory）：RDD中的数据尽量多并且尽量长时间地保存在内存中
* 不可变（Immutable or Read-Only）：一旦创建就不能改变，只能被转换为新的RDDs
* 懒加载（Lazy evaluated）：对RDD的操作是延迟地，直到触发行为操作
* 可缓存（Cacheable）：可以持久化到内存或者硬盘
* 并行（Parallel）：可以并行处理
* 强类型（Typed）：RDD有自己的类型，例如`RDD[Long]`、`RDD[Int]`、`RDD[(Int,String)]`
* 分区的（Partitioned）：RDD中的数据是分区的并且分布在集群中的节点上
* 地址粘性（Location-Stickiness）：本地性偏好

##### RDD Lineage

RDD血缘关系，RDD的来源图谱，包括RDD所依赖的所有RDDs和其间进行的操作。

### Dataset

Dataset是一个分布式的数据集合。Dataset是从Spark 1.6开始引入的新接口，既具有RDD优势，又具有Spark SQL优化的执行引擎带来的优势。Dataset可以从JVM对象来构建，并且可以进行函数式的转换（比如`map`、`flatMap`、`filter`等等）

### DataFrame

DataFrame是一个具有命名的列的分布式数据集。在概念上与关系型数据库的表或者R/Python中的data frame等价，但是Spark对它进行了丰富的优化。可以从多个来源构建DataFrame，比如结构化的数据文件、Hive中的表、外部数据库或者RDDs。

### 总结

Spark 2.0之前，Spark主要的编程接口是RDD（Resillient Distributed Dataset）。Spark 2.0之后，RDDs被Dataset取代，Dataset如RDD一样是强类型的，但是比RDD有更好的优化。[$](http://spark.apache.org/docs/2.4.5/quick-start.html)官方强烈建议使用Dataset替换RDD，Dataset比RDD性能搞好。



