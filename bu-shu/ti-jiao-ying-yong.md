### Submitting Applications

[文档原文](http://spark.apache.org/docs/2.2.0/submitting-applications.html)

Spark的_bin_目录中的_spark-submit_脚本用于在集群上启动应用。可以用于Spark支持的所有集群管理器。

#### 绑定应用的依赖（Bundling Your Application's Dependencies）

如果应用依赖了其它的项目，就要将依赖的项目一起打包，以将代码分发到Spark集群。可以使用sbt或maven将代码和依赖打包为一个装配jar（assembly jar）。依赖的Spark和Hadoop依赖应该被标记为_provided_；它们不用绑定，因为这些依赖在运行时会由集群管理器来提供。

#### 使用_spark-submit_来启动应用（Launching Applications with spark-submit）

bin/spark-submit脚本可以使用Spark和他的依赖来设置classpath，可以支持不同的集群管理器和部署模式

```shell
./bin/spark-submit \
  --class <main-class> \
  --master <master-url> \
  --deploy-mode <deploy-mode> \
  --conf <key>=<value> \
  ... # other options
  <application-jar> \
  [application-arguments]
```

###### 通用选项：

* --class _CLASS\_NAME_：应用的入口类（Java或Scala的类名）
* --master _MASTER\_URL_：集群的master URL（spark://host:port, mesos://host:port, yarn, 或者local）
* --deploy-mode _DEPLOY\_MODE_：将驱动程序部署在工作节点（_cluster_）还是本地性的作为一个外部客户端（_client_），默认_client_
* applicaiton-jar：包括应用和它的所有依赖的jar文件的路径。路径必须是在集群内全局可见的，例如hdfs://路径或者文件路径
* application-arguments：要传到main class的main方法的参数，如果有的话。
* --conf _PROP=VALUE_：Spark配置属性

* --properties-file _FILE_：属性文件，默认conf/spark-defaults.conf

* --driver-meomory _MEM_：驱动器内存（1000M、2G）默认1024M

* --executor-memory _MEM_：executor内存，默认1G

* --files _FILES_：

* --name _NAME_：应用名称

* --jars _JARS_：应用依赖的jar文件，多个jar文件以“,”分隔，不支持目录（文件夹）

* --help：显示帮助信息

* --version：显示当前Spark的版本

###### 仅仅Spark standalone的cluster支持的选项：

* --driver-cores _NUM_：driver的核心数

###### 仅Spark standalone和Mesos的cluster模式支持的选项：

* --supervise：如果有，在失败的场合（非零的exit code）重启driver

###### 仅Spark standalone和Mesos支持的选项：

* --total-executor-cores _NUM_：所有executors的核心总数

###### 仅Spark standalone和YARN支持的选项：

* --executor-cores _NUM_：executor核心数（默认，1in YARN mode or all available cores on the worker in standalone mode）

###### 仅YARN支持的选项：

* --driver-cores _NUM_：driver使用的核心数，仅在cluster模式支持，默认1；
* --queue _QUEUE\_NAME_：所要提交到的YARN队列（默认：“default”）
* --num-executors _NUM_：要启动的executors的数量（默认：2）

###### 脚本例子：

```shell
# Run application locally on 8 cores
./bin/spark-submit \
  --class org.apache.spark.examples.SparkPi \
  --master local[8] \
  /path/to/examples.jar \
  100

# Run on a Spark standalone cluster in client deploy mode
./bin/spark-submit \
  --class org.apache.spark.examples.SparkPi \
  --master spark://207.184.161.138:7077 \
  --executor-memory 20G \
  --total-executor-cores 100 \
  /path/to/examples.jar \
  1000

# Run on a Spark standalone cluster in cluster deploy mode with supervise
./bin/spark-submit \
  --class org.apache.spark.examples.SparkPi \
  --master spark://207.184.161.138:7077 \
  --deploy-mode cluster \
  --supervise \
  --executor-memory 20G \
  --total-executor-cores 100 \
  /path/to/examples.jar \
  1000

# Run on a YARN cluster
export HADOOP_CONF_DIR=XXX
./bin/spark-submit \
  --class org.apache.spark.examples.SparkPi \
  --master yarn \
  --deploy-mode cluster \  # can be client for client mode
  --executor-memory 20G \
  --num-executors 50 \
  /path/to/examples.jar \
  1000

# Run a Python application on a Spark standalone cluster
./bin/spark-submit \
  --master spark://207.184.161.138:7077 \
  examples/src/main/python/pi.py \
  1000

# Run on a Mesos cluster in cluster deploy mode with supervise
./bin/spark-submit \
  --class org.apache.spark.examples.SparkPi \
  --master mesos://207.184.161.138:7077 \
  --deploy-mode cluster \
  --supervise \
  --executor-memory 20G \
  --total-executor-cores 100 \
  http://path/to/examples.jar \
  1000
```

#### Master的不同地址形式（Master URLS）

| **Master URL** | **含义** |
| :---: | :--- |
| local | 在本地运行一个工作者线程（没有并行性） |
| local\[K\] | 在本地运行K个工作者线程（理想情况下，设置为本机的核心数） |
| local\[K,F\] | 本地运行K个工作者线程，F次最大失败次数 |
| local\[\*\] | 根据本机的核心数在本地运行尽可能多的线程 |
| local\[\*,F\] | 根据本机的核心数在本地运行尽可能多的线程，F次最大失败次数 |
| spark://HOST:PORT | 通过指定的host和端口，连接到指定的Spark standalone cluster主机 |
| spark://HOST1:PORT1,HOST2:PORT2 | 通过Zookeeper连接到指定的Spark standalone cluster with standby master |
| mesos:// HOST:PORT | 连接到MESOS集群 |
| yarn | 连接到YARN 集群，可以指定是cluster或client模式。通过环境变量_**HADOOP\_CONF\_DIR**_或者_**YARN\_CONF\_DIR**_来获取集群的位置。 |

#### 通过问价加载配置\(Loading Configuration  from a File\)

通过_--properties-file FILE_选项配置额外属性文件的路径，默认文件路径_conf/spark-defaults.conf_；

通常，驱动程序的_**SparkConf**_的选项配置优先级最高，_spark-submit_脚本的选项配置次之，属性文件中的选项配置优先级最低。

#### **高级依赖管理（Advanced Dependency Management）**

使用spark-submit时，应用的jar文件和_--jars JARS_选项配置的jar文件会自动的传送给集群。

Spark支持不同形式的jar文件URL：

* **file:--**绝对路径和file:/URIs由driver的HTTP文件服务器serve，每个executor通过driver的HTTP服务器拉取文件。
* **hdfs:,http:,https:ftp:--**通过指定的URI拉取文件（需要用户密码认证的形式[https://user:password@host/](https://user:password@host/)_..._）
* **local-**一个以本地“/”开始的URI，期待的是每个工作者节点都存在一个本地文件。这意味着没有网络IO的开销，并且对于推送到各个工作者的大文件/大JAR工作良好。

JAR文件和文件被复制到每个执行器executor节点上的_**SparkContext**_的工作目录。这会用掉很可观的量的空间，所以需要被清理。对于YARN，会自动执行清理，对于Spark standalone可以通过属性_spark.worker.cleanup.appDataTtl_配置自动清理。

### `SparkLauncher()` 提交 Spark 应用

除了使用 `spark-submit` 通过命令行提交 Spark Application 外，还可以使用 Spark 提供的 `SparkLauncher()` 在代码中提交 Spark 应用。

```scala
new SparkLauncher()
      // SPARK_HOME 目录
      .setSparkHome("/opt/cloudera/parcels/SPARK2-2.4.0.cloudera1-1.cdh5.13.3.p0.1007356/lib/spark2")
      // .setAppName(appName)
      .setAppResource(appJarPath)
      .addJar(dependenceJars)
      .setMainClass(mainClass)
      //.addAppArgs("", "")
      .setMaster("yarn")
      .setDeployMode("cluster")
      .addSparkArg("--driver-memory", "4G")
      .addSparkArg("--num-executors", "5")
      .addSparkArg("--executor-cores", "2")
      .addSparkArg("--executor-memory", "4G")
      // .launch()
      .startApplication(
      new SparkAppHandle.Listener {
        override def stateChanged(handle: SparkAppHandle): Unit = {
          println(s"state changed---${handle.getAppId}-----------------${handle.getState}")
        }

        override def infoChanged(handle: SparkAppHandle): Unit = {
          println(s"info changed---${handle.getAppId}-----------------${handle.getState}")
        }
      }
    )
```