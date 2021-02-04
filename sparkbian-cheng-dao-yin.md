### Spark Programming Guide[$](http://spark.apache.org/docs/2.2.0/rdd-programming-guide.html)

#### Overview

每个Spark应用都由一个驱动程序（*driver program*)构成，驱动程序运行`main`函数并且在集群上执行各种并行的操作。Spark的一个主要抽象概念是RDD（*resilient distributed dataset*），RDD是一个在集群中的节点上分区保存的元素的集合，可以对RDD进行并行操作。RDD可以从Hadoop文件创建，也可以由驱动程序中的Scala集合创建，可以对RDD进行转换。用户可以将RDD持久化（*persist*）到内存，以便在并行操作中高效的重用RDD。在节点故障是RDD能够自动地恢复。

Spark的另一个抽象概念是共享变量（*shared varibales*），共享变量也可以用在并行操作中。默认情况下，当Spark以不同节点上的*tasks*的形式并行运行函数时，Spark会将函数中使用的每个变量的拷贝（*copy*）发送给每个*task*。有时，需要在tasks之间，或者在tasks和驱动程序之间共享一个变量。Spark支持两种类型的共享变量：广播变量（*broadcast variables*）和累加器（*accumulators*）。广播变量在所有节点的内存中缓存（cache）一个值；累加器是只会被“增加”的变量，比如计数器（*counters*）和求和（*sums*）。

#### 导入Spark（Linking with Spark）

Spark 2.2.0默认使用Scala 2.11。

使用Spark就要导入`spark-core_2.11`，如果要访问HDFS集群，需要依赖对应HDFS版本的`hadoop-client`。

导入Spark入口类

```scala
import org.apache.spark.SparkContext
import org.apache.spark.SparkConf
```

#### 初始化Spark

首先要创建`SparkContext`，要用`SparkConf`来配置`SparkContext`。每个JVM只会有一个活跃的`SparkContext`，如果要创建新的`SparkContext`必须对当前活跃的`SparkContext`调用`stop()`函数来关闭它。

```scala
val conf = new SparkConf().setAppName(appName).setMaster(master)
new SparkContext(conf)
```

`appname`是应用名，`master`是集群URL，或者是本地模式对应的“local”字符串。实践中，不会硬编码`master`，而是在使用`spark-submit`提交应用时指定`master`。

#### 使用Spark Shell

Spark shell中，会自动创建一个`SparkContext`，变量名为`sc`（Spark 2的shell对应创建一个名为`spark`的`SparkSession`）。使用`--master`参数设置master，使用`--jars`导入JARs到classpath。使用`--package`来添加依赖的包，使用`--repositories`参数指定仓库。

```shell
$ ./bin/spark-shell --master local[4]
```

添加`code.jar`到classpath：

```shell
$ ./bin/spark-shell --master local[4] --jars code.jar
```

添加maven依赖：

```shell
$ ./bin/spark-shell --master local[4] --packages "org.example:example:0.1"
```

运行`spark-shell --help`可以查看所有可用选项。

