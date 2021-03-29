### Spark框架

Spark 是一个基于 JVM 的分布式数据处理框架，采用先进的DAG（Directed Acyclic Graph，有向无环图）数据处理引擎。Spark 是基于内存进行运算的，它会尽量避免产生磁盘IO。基于内存的数据处理结合基于DAG的数据处理引擎，使得Spark非常高效。

#### DAG

DAG 用数学术语来说，由一组**顶点**（Vertex）和连接顶点的**有向的边**（Directed Edge）组成。tasks 是按照每个 DAG 执行的。在MapReduce中，DAG只有两种顶点，map和reduce，边是从map指向reduce的。Spark中，DAG 可以非常复杂。Spark 有工具可以展示Spark Job的DAG。

#### Spark shell

Spark shell有Scala、Python、R版本的REPL（Read、Evaluate、Print、Loop）功能。

#### driver program和worker nodes

driver program分配tasks给合适的workers。

Spark 应用中，SparkContext是driver program，它与集群管理器通信以运行tasks。Spark支持的集群管理器有Spark Master（Spark Core的一部分），Mesos master，YARN Resource Manager。

在YARN部署的Spark集群中，Spark driver program运行在YARN的application master进程中，或者Spark driver program作为YARN的一个client运行。Spark从Hadoop配置中获取YARN 资源管理器的地址，所以提交Spark jobs的时候，不用明确指定master URL。

#### Scala

JVM编程语言、函数式（像数据函数一样输入确定，输出就确定）编程语言、面向对象。

Spark 从Scala获取的一个最重要特性是，能够将函数作为参数用于Spark 转换操作和行动操作。

#### RDD

RDD，类似于Scala中的集合，可以使用某些类似Scala集合的转换操作。

弹性分布式数据集，支持在集群的节点上进行并行的分布式的处理。对于大型的数据集，可以被切分为小块并分发到集群中的多个节点。

RDD的特性：

- 不可变

  一旦创建，不能更改。

  可以将大的切分为小的，分发到多个节点处理，最后汇集各部分结果产生最终的结果，而不用担心底层数据的改变。

  当处理一个RDD的某些部分的节点故障，driver program可以重建这些部分并将处理tasks分配给其它节点。

- 分布式

  分布式处理，故障恢复，高弹性

- 存活于内存中

  RDD 一般在内存中保存和处理，内存不足才溢出到磁盘。

- 强类型（strongly typed）

  可以使用任意支持的数据类型创建RDD。这些数据类型可以是Scala / Java 支持的内部数据类型或者自定义的数据类型，比如自定义的classes。

  这样可以避免运行时类型错误，编译时就可以发现数据类型方面的问题。

#### 惰性求值

Lazy Evaluation

#### 操作

map

flatMap

reduce

coalesce

count

collect

first

foreach

take

countByKey

groupByKey

reduceByKey（类似于Scala集合的`reduceLeft`操作）

repartition

sortByKey

#### Join

转换操作

join：按key进行连接，内连接？？

leftOuterJoin

rightOuterJoin

fullOuterJoin