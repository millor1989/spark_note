# Quick Start[\_](http://spark.apache.org/docs/2.2.0/quick-start.html)

Spark 2.0之前，Spark主要的编程接口是RDD（Resillient Distributed Dataset）。Spark 2.0之后，RDDs被Dataset取代，Dataset如RDD一样是强类型的，但是比RDD有更好的优化。官方强烈建议使用Dataset替换RDD，Dataset比RDD性能更好。

### 1、Interactive Analysis with the Spark Shell

#### 1.1、Basics

运行Spark目录中的如下命令启动Spark shell：

```shell
./bin/spark-shell
```

Spark Shell有一些可用的选项，比如，“--jars &lt;comma seperated jar-pathes&gt;”添加jar文件，通过“--help”可以查看可用选项列表。

Spark的主要抽象概念是一个叫做Dataset的分布式的数据集合。Datasets可以通过Hadoop InputFormats（例如HDFS文件）或者通过转换其它Datasets创建。

```
scala> val textFile = spark.read.textFile("README.md")
textFile: org.apache.spark.sql.Dataset[String] = [value: string]
```

Dataset的一些行动操作（actions）：

```
scala> textFile.count() // Number of items in this Dataset
res0: Long = 126 // May be different from yours as README.md will change over time, similar to other outputs

scala> textFile.first() // First item in this Dataset
res1: String = # Apache Spark
```

Dataset的转换操作（transformations）：

```
scala> val linesWithSpark = textFile.filter(line => line.contains("Spark"))
linesWithSpark: org.apache.spark.sql.Dataset[String] = [value: string]
```

转换操作和行动操作一起进行：

```
scala> textFile.filter(line => line.contains("Spark")).count() // How many lines contain "Spark"?
res3: Long = 15
```

#### 1.2、More on Dataset Operations

Dataset行为操作和转换操作可以用于更复杂的运算。举个找出字数最多行字数的例子：

```
scala> textFile.map(line => line.split(" ").size).reduce((a, b) => if (a > b) a else b)
res4: Long = 15
```

可以引用其它地方定义的函数，是代码变得简洁：

```
scala> import java.lang.Math
import java.lang.Math

scala> textFile.map(line => line.split(" ").size).reduce((a, b) => Math.max(a, b))
res5: Int = 15
```

随着Hadoop的普及，有一种常见的数据流类型MapReduce。Spark可以简洁地实现MapReduce流。计算每个单词数量的例子：

```
scala> val wordCounts = textFile.flatMap(line => line.split(" ")).groupByKey(identity).count()
wordCounts: org.apache.spark.sql.Dataset[(String, Long)] = [value: string, count(1): bigint]
```

使用_flatMap_把行的Dataset转换为了单词组成的Dataset，使用_groupByKey_和_count_计算文件中每个单词的数量并输出\(String,Long\)对组成的Dataset。执行_collect_操作：

```
scala> wordCounts.collect()
res6: Array[(String, Int)] = Array((means,1), (under,2), (this,3), (Because,1), (Python,2), (agree,1), (cluster.,1), ...)
```

#### 1.3、Caching

Spark也支持把数据集合放进集群的分布式的内存缓存。当需要重复性访问数据时，这是非常有用的，比如查询一个小的“热点”dataset或者使用像PageRank这种迭代算法时。

```
scala> linesWithSpark.cache()
res7: linesWithSpark.type = [value: string]

scala> linesWithSpark.count()
res8: Long = 15

scala> linesWithSpark.count()
res9: Long = 15
```

### 2、Self-Contained Applications

```scala
/* SimpleApp.scala */
import org.apache.spark.sql.SparkSession

object SimpleApp {
  def main(args: Array[String]) {
    val logFile = "YOUR_SPARK_HOME/README.md" // Should be some file on your system
    val spark = SparkSession.builder.appName("Simple Application").getOrCreate()
    val logData = spark.read.textFile(logFile).cache()
    val numAs = logData.filter(line => line.contains("a")).count()
    val numBs = logData.filter(line => line.contains("b")).count()
    println(s"Lines with a: $numAs, Lines with b: $numBs")
    spark.stop()
  }
}
```



