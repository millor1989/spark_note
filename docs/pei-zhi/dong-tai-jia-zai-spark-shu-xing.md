# 动态加载Spark属性（Dynamically Loading Spark Properties）

在一些情况下可能需要避免对SparkConf的一些配置进行硬编码。例如，可能想要用不同的master或者不同数量的内存来运行同一个应用。Spark允许创建一个空的SparkConf：

```
val sc = new SparkContext(new SparkConf())
```

然后，可以在运行时提交配置如下：

```
./bin/spark-submit
 --name "My app" \
 --master local[4] \
 --conf spark.eventLog.enabled=false \
 --conf "spark.executor.extraJavaOptions=-XX:+PrintGCDetails -XX:+PrintGCTimeStamps" \
  myApp.jar
```

Spark shell和spark-submit工具支持两种动态加载配置的方式。第一种是命令行选项，例如--master。spark-submit可以使用--conf接收任意的Spark属性，但是对于作为Spark应用启动一部分的属性要使用特殊的标识。运行./bin/spark-submit --help将会显示这些选项的完整列表。

bin/spark-submit也从conf/spark-defaults.conf读取配置选项，在这个文件中每一行由一个键和一个以空格分开的值组成。例如：

```
spark.master            spark://5.6.7.8:7077
spark.executor.memory   4g
spark.eventLog.enabled  true
spark.serializer        org.apache.spark.serializer.KryoSerializer
```

任何通过命令标识或者属性文件指定的属性都会传递给应用并且和通过SparkConf指定的属性组合在一起。直接在SparkConf上指定的属性有最高的优先级，然后是传递给spark-submit和spark-shell的属性标识，最后是spark-defaults.conf中的选项。一些配置的键的名字可能与老版本的spark对应属性的名字不同，这种情况下，老版本的键也是可用的，但是优先级比任意配置方式的新版本的键低。

