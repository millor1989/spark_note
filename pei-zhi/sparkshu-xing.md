# Spark属性（Spark Properties）

Spark属性控制大多数应用的设置，对于每个应用分别设置。这些属性可以直接在传递给SparkContext的SparkConf上设置。SparkConf允许设置一些通用的属性（例如，master URL、应用的名称），也可以使用set\(\)方法通过任意键-值对进行设置。例如，可以用如下方式初始化一个两个线程的应用：

注意，使用local\[2\]，意味着两个线程——代表了最小的并行度，可以帮助检测只有在分布式context中运行时存在的bugs。

```scala
val conf = new SparkConf()
             .setMaster("local[2]")
             .setAppName("CountingSheep")
val sc = new SparkContext(conf)
```

注意，在本地模式中，可以使用多于1个的线程，并且在Spark Streming之类的情况下，可能实际需要多余一个的线程以防止任何类型的饥饿问题（starvation issues）。

指定一些时间属性时候应该使用时间单位，格式如下：

```
25ms (milliseconds)
5s (seconds)
10m or 10min (minutes)
3h (hours)
5d (days)
1y (years)
```

指定一些字节大小的配置是使用的单位如下：

```
1b (bytes)
1k or 1kb (kibibytes = 1024 bytes)
1m or 1mb (mebibytes = 1024 kibibytes)
1g or 1gb (gibibytes = 1024 mebibytes)
1t or 1tb (tebibytes = 1024 gibibytes)
1p or 1pb (pebibytes = 1024 tebibytes)
```



