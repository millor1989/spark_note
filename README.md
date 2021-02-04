# Spark总览（Spark Overview）[\_](http://spark.apache.org/docs/2.2.0/index.html)

Apache Spark是一个快速的通用目的的（general-purpose）集群运算系统。它提供了Java，Scala，R语言的高级（high-level）APIs，和一个支持通用执行图（general execution graphs）优化的引擎。也支持丰富的更高级的（higher-level）工具，包括用于结构化数据处理的[Spark SQL](http://spark.apache.org/docs/2.2.0/sql-programming-guide.html)，用于机器学习的[MLlib](http://spark.apache.org/docs/2.2.0/ml-guide.html)，用于图处理的[GraphX](http://spark.apache.org/docs/2.2.0/graphx-programming-guide.html)，和[Spark Streaming](http://spark.apache.org/docs/2.2.0/streaming-programming-guide.html)。

Spark可以在Windows和类UNIX（UNIX-like，Linux，Mac OS）系统运行。在一台机器上本地性的运行很简单——只需要在系统 _PATH_ 上安装java，或者让 _JAVA\_HOME_ 环境变量指向Java安装目录。

#### 1、Running the Examples and Shell

可以在Spark顶层目录，使用命令 _bin/run-example &lt;class&gt; \[params\]_ 运行Spark的Java或Scala样例，如：

```shell
./bin/run-example SparkPi 10
```

通过一个修改版本的Scala shell可以交互式地运行Spark。这是学习Spark框架的一种很棒的方式。

```shell
./bin/spark-shell --master local[2]
```

带上 _--help_ 选项运行Spark shell，可以查看Spark shell使用帮助。

Spark也提供了Python API。在Python编译器中使用 _bin/pyspark_ 交互地运行Spark：

```shell
./bin/pyspark --master local[2]
```

运行Python样例应用：

```shell
./bin/spark-submit examples/src/main/python/pi.py 10
```

Spark也有R API。在R编译器中交互地运行Spark如下：

```shell
./bin/sparkR --master local[2]
```

R样例应用：

```shell
./bin/spark-submit examples/src/main/r/dataframe.R
```

#### 2、Launching on a Cluster

Spark可以单独运行，也可以在某些集群管理器上运行。

* Standalone Deploy Mode：在一个私有集群上部署Spark
* Apache Mesos
* Hadoop YARN



