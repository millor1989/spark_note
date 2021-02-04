# Viewing Spark Properties

通过地址[http://&lt;driver&gt;:4040访问](http://<driver>:4040访问) Web UI的“Environment”标签页。这是一个确保属性正确设置的有用方式。注意，只有明确的通过spark-defaults.conf、SparkConf或者命令行指定的属性才会在这里展示。对于其它的配置属性，你可以认为它们是使用的默认值。

## 1、可用的属性（Available Properties）

[文档链接](http://spark.apache.org/docs/2.2.0/configuration.html#available-properties)

许多控制内部设置的属性都有合理的默认值。一些最常用的选项是：

### 1.1、应用属性

* spark.app.name    默认值：\(none\)    应用的名字，会在UI和日志数据中显示.
* spark.driver.cores    默认值：1    用于驱动器进程\(driver process\)的核心数，只在cluster模式有用。
* spark.driver.maxResultSize    默认值1g    为每个Spark action限制所有分区的序列化结果的总大小。最小1M，如果是0就不限制。如果总大小超过这个限制，job会被放弃。这个限制如果过大可能会导致Driver中out-of-memory的错误（视spark.driver.memory和JVM中对象的内存开销而定）。设置一个何时的限制值可以保护driver不发生out-of-memory的错误。
* spark.driver.memory    默认值：1g    驱动进程（即SparkContext初始化）使用的内存大小（例如，1g，2g）。注意，在client模式，这个设置不能在应用通过SparkConf直接设置，应为在那个时间点驱动器的JVM已经启动。作为替换，要通过--driver-memory命令行选项设置或者在默认属性文件中设置。
* spark.executor.memory    默认值：1g   每个executor进程使用的内存大小。
* spark.extraListeners    默认值\(none\)    一都好分割的实现了SparkListener的类的列表；在SparkContext初始化的时候，这些类的实例会通过Spark的监听总线被创建和注册。如果一个类有一个只有一个参数为SparkConf的构造器，会调用这个构造器；否则会调用无参构造器。如果没有可用的构造器，SparkContext的创建会有一个异常并失败。
* spark.local.dir    默认值：/tmp    Spark中用于“临时”空间的目录，包括map输出的文件和存储在磁盘上的RDDs。这应该在系统中一个快速的、本地的磁盘上。也可以是不同磁盘上以逗号分割的多个目录的列表。**Note.**在Spark1.0和之后的版本中，这个设置会被集群管理器设置的环境变量_SPARK\_LOCAL\_DIRS_或_LOCAL\_DIRS_覆盖。
* spark.logConf    默认值：false    Logs the effective SparkConf as INFO when a SparkContext is started.
* spark.master   默认值：\(none\)    要连接到的集群管理器。
* spark.submit.deployMode   默认： \(none\)    Spark驱动程序的部署模式，client或者cluster，意味着本地性地启动驱动程序（client）或者在集群中一个节点上远程地启动驱动程序（cluster）。
* spark.log.callerContext    默认：\(none\)    在Yarn上或者HDFS上运行时应用信息会被写到Yarn RM 日志或者HDFS audit日志。它的长度依赖于Hadoop配置hadoop.caller.context.max.size。应该简洁，最多50个字符。
* spark.driver.supervise    默认值：false    自动重启driver如果它以非零的exit status失败。只有在Spark standalone模式或者Mesos集群部署模式时有效。

除了这些，下面的属性也是可用的，在某些情况下可能有用：

### 1.2、运行时环境（Runtime Environment\)

* spark.driver.extraClassPath
* spark.driver.extraJavaOptions
* spark.driver.extraLibraryPath
* spark.executor.extraJavaOptions
* spark.executor.extraLibraryPath
* spark.executor.logs.rolling.maxRetainedFiles
* spark.executorEnv.\[EnvironmentVariableName\]
* spark.redaction.regex 屏蔽日志、UI中的配置属性的敏感词
* spark.files
* spark.jars

### 1.3、混洗行为（Shuffle Behavior）

* spark.reducer.maxSizeInFlight 同时从每个reduce task获取map输出的最大size。
* spark.reducer.maxReqsInFlight
* spark.shuffle.compress
* spark.shuffle.file.buffer
* spark.shuffle.service.enabled 开启外部混洗服务
* spark.shuffle.service.port
* spark.shuffle.service.index.cache.entries
* spark.io.encryption.enable 开启IO加密
* spark.io.encryption.keySizeBits

### 1.4、Spark UI

* spark.eventLog.compress 是否压缩事件日志,使用的压缩方式是spark.io.compression.codec
* spark.eventLog.dir
* spark.eventLog.enabled
* spark.ui.enabled
* spark.ui.retainedJobs

### 1.5、压缩和序列化（Compression and Serialization）

* spark.broadcast.compress
* spark.io.compression.codec
* spark.io.compression.lz4.blockSize
* spark.kryo.classesToRegister
* spark.rdd.compress
* spark.serializer

### 1.6、内存管理（Memory Management）

#### 1.6.1、执行行为（Execution Behavior）

* spark.memory.fraciton   默认值：0.6    执行和存储所使用的（堆空间，300MB）的分数。越小，溢出和缓存的数据驱逐就越频繁发生。推荐使用默认值。
* spark.memory.storageFraction    默认值：0.5    这个值越大执行可使用的内存越少，并且tasks可能会更加频繁的溢出到磁盘。推荐使用默认值。
* spark.memory.offHeap.enabled    默认值：false    如果为true，对于某些操作Spark会尝试使用offHeap内存
* spark.memory.offHeap.size
* spark.storage.replication.proactive    默认值：false    为RDD块开启主动块副本。
* spark.broadcast.blockSize
* spark.executor.cores
* spark.default.parallelism
* spark.executor.heartbeatInterval
* spark.files.fetchTimeout
* spark.storage.memoryMapThreshold
* spark.hadoop.mapreduce.fileoutputcommitter.algorithm.version

#### 1.6.2、网络（Networking）

* spark.rpc.message.maxSize        默认值：128          单位MB；当运行的job有数千的map和reduce的tasks并且能看到RPC message大小的信息时，增大这个值。
* spark.network.timeout
* spark.rpc.numRetries

#### 1.6.3、调度（Scheduling）

* spark.locality.wait         默认值：3s            在放弃在数据本地的机架启动task而在非本地节点启动task之前的等待时间。
* spark.locality.wait.node    默认值：spark.locality.wait     节点本地性级别的等待超时时间，设置为0时跳过该级别，直接去更低的本地性级别
* spark.locality.wait.process    默认值：spark.locality.wait     进程本地性级别的等待超时时间，（进程本地性，指的是从特定的executor进程中获取缓存数据）
* spark.locality.wait.rack    默认值：spark.locality.wait     机架本地性级别的等待超时时间
* spark.scheduler.mode
* spark.blacklist.enabled           默认值：false       设置为true，防止Spark调度执行器上因为失败次数太多而被加入黑名单的tasks。
* spark.blacklist.timeout
* spark.speculation              默认值：false             如果一个stage的一个或者多个tasks运行很慢，tasks会被重新运行。
* spark.task.maxFailures

#### 1.6.4、动态分配（Dynamic Allocation）

* spark.dynamicAllocation.enabled            默认值：false
* spark.dynamicAllocation.minExecutors
* spark.dynamicAllocation.maxExecutors          默认值：无限           如果开启了动态分配，executors的数量上限。
* spark.dynamicAllocation.initialExecutors

#### 1.6.5、安全

spark.acls.enable             默认值：false           是否开启spark acls。如果开启将会检查用户是否有权限查看和修改job。

spark.modify.acls

#### 1.6.6、TLS\SSL

spark.ssl.enable      默认值：false             是否在所有支持的协议上开启SSL连接。

spark.ssl.\[namespace\].port           默认值：none          SSL服务监听的端口。

spark.ssl.protocol         默认值：None           协议名称，必须是JVM支持的协议。

#### 1.6.7、Spark SQL相关属性

```scala
// spark is an existing SparkSession
spark.sql("SET -v").show(numRows = 200, truncate = false)
```

在应用中运行以上代码，则能够显示Spark SQL相关的属性和说明。

#### 1.6.8、Spark Streaming相关属性

* spark.streaming.backpressure.enabled
* spark.streaming.receiver.maxRate
* spark.streaming.unpersist

#### 1.6.9、SparkR相关属性

#### 1.6.10、GraphX相关属性

spark.graphx.pregel.checkpointInterval        默认值：-1      Pregel中graph和消息的检查点时间间隔。

#### 1.6.11、部署相关属性（Deploy）

spark.deploy.recoveryMode           默认值：none            集群模式提交的spark job失败重新启动时使用的恢复模式。仅在Standalon或者Mesos的集群模式运行时适用。

#### 1.6.12、集群管理器属性

每个集群管理器都有额外的配置选项。在它们各自对应的页面可以找到对应的配置：[YARN配置](http://spark.apache.org/docs/2.2.0/running-on-yarn.html#configuration)、[Mesos配置](http://spark.apache.org/docs/2.2.0/running-on-mesos.html#configuration)、[Standalone模式配置](http://spark.apache.org/docs/2.2.0/spark-standalone.html#cluster-launch-scripts)。

##### 1.6.12.1、环境变量（Environment Variables）

一些Spark设置可以通过环境变量配置。环境变量读取自‘conf/spark-env.sh’脚本，这个脚本在Spark的安装目录。在Standalone和Mesos模式，这个文件可以设置机器指定的信息例如主机名。在运行本地Spark应用或者提交的脚本时，也从它获取资源。Note.在Spark安装后，这个文件默认是不存在的。可以复制‘conf/spark-env.sh.template’来创建它。要确保它是可执行的。可以在'spark-env.sh'中设置如下变量：

* JAVA\_HOME             Java的安装路径（如果不在默认的PATH中）
* PYSPARK\_PYTHON
* PYSPARK\_DRIVER\_PYTHON
* SPARKR\_DRIVER\_R
* SPARK\_LOCAL\_IP             机器绑定的本地IP地址
* SPARK\_PUBLIC\_DNS           Spark程序向其它机器广告的主机名

除了上面的环境变量，也有选项用于[设置Spark\[standalone集群脚本\]](http://spark.apache.org/docs/2.2.0/spark-standalone.html#cluster-launch-scripts)，例如每个机器上使用的核心数和最大内存。由于‘spark-env.sh’是一个shell脚本，有些环境变量可以通过程序设置，例如，可以通过查找指定的网络接口来计算‘SPARK\_LOCAL\_IP’。

NOTE.如果在YARN上使用cluster模式运行Spark，要使用\`spark.yarn.appMasterEnv.\[EnvironmentVariableName\]\`属性来在\`conf/spark-defaults.conf\`文件中设置环境变量。在cluster模式，spark-env.sh中配置的环境变量不会反映在YARN的Application Master进程中。YARN相关的Spark属性，参考[链接](http://spark.apache.org/docs/2.2.0/running-on-yarn.html#spark-properties)。

##### 1.6.12.2、配置日志记录属性（Configuring Logging）

在conf目录中增加log4j.properties文件。可以使用log4j.properties.template文件作为模板来配置。

##### 1.6.12.3、更改默认的配置目录（Overriding configuration directory）

默认的配置目录为“SPARK\_HOME/conf”，可以通过设置SPARK\_CONF\_DIR来改变默认的配置文件目录，设置后Spark会从设置的目录中读取配置文件（spark-defaults.conf, spark-env.sh, log4j.properties, etc）

##### 1.6.12.4、继承Hadoop集群配置（Inheriting Hadoop Cluster Configuration）

如果要使用Spark读写HDFS，在Spark的classpath中需要包括两个Hadoop的配置文件\* \`hdfs-site.xml\`（为HDFS客户端提供默认的行为）、\* \`core-site.xml\`（设置默认文件系统名称）。这两个配置文件的位置因Hadoop的版本的不同而不同，但是通常的位置是在\`/etc/hadoop/conf\`目录中。有些工具即时（on-the-fly）创建配置，但是提供了下载它们副本的机制。为了让这些文件对Spark可见，设置\`$SPARK\_HOME/spark-env.sh\`中的\`HADOOP\_CONF\_DIR\`为一个包含这些配置文件的位置。

