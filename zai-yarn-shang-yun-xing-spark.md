# 在YARN上运行Spark（Running Spark on YARN）

Spark自0.6.0版本开始支持在YARN上运行。

## 1、在YARN上启动Spark（Launching Spark on YARN）

确保`HADOOP_CONF_DIR`或`YARN_CONF_DIR`指向Hadoop集群的包含配置文件（client侧）目录。这些配置用于连接到YARN资源管理器并写HDFS。这个目录包含的配置会被分发到YARN集群，以便应用使用的所有容器都使用相同的配置。如果配置指向不是YARN管理的Java系统属性或者环境变量，他们应该在Spark应用的配置（driver，executors，client模式运行的AM）中设置。

在YARN集群上有两种部署模式用来启动Spark应用。**在cluster模式**，Spark的driver在一个由YARN集群管理的**application master**进程中运行，在应用初始化之后就与客户端client无关了。**在client模式**，Spark的driver运行在客户端client的进程中，applicaiton master仅仅被用于从YARN请求资源。

在Spark standalone和Mesos模式下，通过_spark-submit_的参数_--master_来指定master的地址。在YARN模式下资源管理器的地址通过Hadoop的配置获取。所以，只用将参数_--master_设置为yarn即可。

cluster模式启动Spark应用：

```shell
$ ./bin/spark-submit --class path.to.your.Class --master yarn --deploy-mode cluster [options] <app jar> [app options]
```

例如：

```shell
$ ./bin/spark-submit --class org.apache.spark.examples.SparkPi \
    --master yarn \
    --deploy-mode cluster \
    --driver-memory 4g \
    --executor-memory 2g \
    --executor-cores 1 \
    --queue thequeue \
    lib/spark-examples*.jar \
    10
```

上面启动了一个YARN客户端client程序，启动了默认的Applicaiton Master。然后SparkPi会作为Applicaiton Master的子线程来运行。客户端会定期从Applicaiton Master轮询状态更新并在控制台展示它们。一旦应用停止运行，客户端就会退出。

client模式启动Spark应用：

```shell
$ ./bin/spark-shell --master yarn --deploy-mode client
```

### 1.1、添加其它JAR文件（Adding Other JARs）

在cluster模式，driver在client以外的机器上运行，所以SparkContext.addJar对于client本地的文件不再是开箱即用（work out of box）的。为了使文件对于SparkContext.addJar可用，在启动命令中用--jars选项来添加其它jar文件。

```shell
$ ./bin/spark-submit --class my.main.Class \
    --master yarn \
    --deploy-mode cluster \
    --jars my-other-jar.jar,my-other-other-jar.jar \
    my-main-jar.jar \
    app_arg1 app_arg2
```

## 2、准备Preparations

在YARN上运行Spark需要支持YARN的Spark二进制发行包。可以[下载](http://spark.apache.org/downloads.html)也可以自己[构建](http://spark.apache.org/docs/2.2.0/building-spark.html)。

为了使Spark runtime jar文件对YARN可访问，需要指定`spark.yarn.archive`或者`spark.yarn.jars`。详细的配置参考[Spark属性](http://spark.apache.org/docs/2.2.0/running-on-yarn.html#spark-properties)。如果`spark.yarn.archive`或者`spark.yarn.jars`都没有指定，Spark会创建一个zip文件打包_$SPARK\_HOME/jars_中的所有jar文件并把它上传到分布式缓存。

### 2.1、Spark的（YARN相关）属性（Spark Properties）

* spark.yarn.am.memory
  ：client模式的YARN application master的内存大小，和JVM内存大小的格式一样的字符串。在cluster模式，用spark.dirver.memory代替。使用小写的后缀，例如，k，m，g，t，p分别代表kibi，mebi，gibi，tebi和pebi字节。默认值：512m。
* spark.yarn.am.cores
  ：client模式的YARN applicaiton master的核心数量；cluster模式使用spark.driver.cores替代。默认：1。
* spark.yarn.am.waitTime
  ：client模式，YARN application master等待driver连接的时间；cluster模式YARN application master等待SparkContext初始化的时间。
* spark.yarn.submit.file.replication
  ：为应用而提交到HDFS的文件的HDFS副本级别。包括Spark jar、app jar和任意的分布式缓存文件/档案。
* spark.yarn.stagingDir
  ：提交应用时的暂存目录（staging directory）。默认：当前用户在文件系统中的home目录。
* spark.yarn.preserve.staging.files
  ：设置为true来在作业结束时保存暂存文件（spark jar，app jar，分布式缓存文件）而不是删除暂存文件。默认，false。
* spark.yarn.scheduler.heartbeat.interval-ms
  ：Spark application master向YARN发送心跳的时间间隔（ms）。最大值为YARN的到期时间，即_yarn.am.liveness-monitor.expiry-interval-ms_。默认，3000。
* spark.yarn.scheduler.initial-allocation.interval：当有等待中的容器分配请求时，Spark application master向YARN发送心跳的初始时间间隔。不应该比_spark.yarn.scheduler.heartbeat.interval-ms_大，如果请求等待一直存在，这个分配间隔会连续性的翻倍，直到达到_spark.yarn.scheduler.heartbeat.interval-ms_。默认，200ms。
* spark.yarn.max.executor.failures
* spark.yarn.historyServer.address
* spark.yarn.dist.archives
* spark.yarn.dist.files
* spark.yarn.dist.jars
* spark.executor.instances
* spark.yarn.executor.memoryOverhead
* spark.yarn.driver.memoryOverhead
* spark.yarn.am.memoryOverhead
* spark.yarn.am.port
* spark.yarn.queue
* spark.yarn.jars
* spark.yarn.archive
* spark.yarn.access.hadoopFileSystems
* spark.yarn.appMasterEnv.\[EnvironmentVariableName\]
* spark.yarn.containerLauncherMaxThreads
* spark.yarn.am.extraJavaOptions
* spark.yarn.am.extraLibraryPath
* spark.yarn.maxAppAttempts
* spark.yarn.am.attemptFailuresValidityInterval
* spark.yarn.executor.failuresValidityInterval
* spark.yarn.submit.waitAppCompletion
* spark.yarn.am.nodeLabelExpression
* spark.yarn.executor.nodeLabelExpression
* spark.yarn.tags
* spark.yarn.keytab
* spark.yarn.principal
* spark.yarn.config.gatewayPath

## 3、配置Configuration

YARN上运行Spark的大部分配置和其它部署模式一样。参考[Spark配置](http://spark.apache.org/docs/2.2.0/configuration.html)。

## 4、应用Debug（Debugging your Application）

在YARN术语中，executors和application masters在容器的内部运行。YARN在应用结束后有两种方式处理容器的日志。如果日志聚合（log aggregation）被开启（通过yarn.log-aggregation-enable 配置），容器日志会被复制到HDFS并从本地机器删除。这些日志可以通过_yarn logs_命令在集群的任意位置查看：

```shell
yarn logs -applicationId <app ID>
```

这个命令会输出指定应用的所有的容器的所有日志文件。也可以通过HDFS shell或API直接查看HDFS中的容器日志文件。日志文件的目录可用通过YARN配置（`yarn.nodemanager.remote-app-log-dir`和`yarn.nodemanager.remote-app-log-dir-suffix`）查看。这些日志在Spark Web UI的Executors标签页也可以查看。需要同时运行Spark history server和MapReduce history server并适当配置`yarn-site.xml`的`yarn.log.server.url`配置。Spark history UI的日志的URL会重定向到MapReduce history Server来展示聚合日志。

如果日志聚合没有开启，日志会本地性的保存在_YARN\_APP\_LOGS\_DIR_目录，这个目录根据Hadoop版本和安装的不同通常被配置为`/tmp/logs`或`$HADOOP_HOME/logs/userlogs`。容器的日志需要到包含容器的主机和这个目录去查看。子目录用applicaiton ID和容器ID组织日志文件。这些日志在Spark Web UI的Executors标签页也可以查看，并且不用运行MapReduce history server。

如果要回顾（review）每个container的启动环境，要增加_yarn.nodemanager.delete.debug-delay-sec_到一个非常大的值（例如，3600），然后再容器启动的节点上的yarn.nodemanager.local-dirs目录访问应用的缓存（application cache）。这个目录包括启动脚本、JAR文件和容器启动使用的环境变量。这个方法特别是对于debug classpath问题很有用。（Note.要启用这个配置需要集群配置的管理员权限_（admin privileges）_并且需要重启所有的节点管理器。所以，这不适用于托管集群_（hosted cluster）_。）

对application master或者executors使用**自定义log4j**的配置的几种方法：

* 使用spark-submit脚本的--files选项上传log4j.properties配置文件。
* 为driver的_spark.driver.extraJavaOptions_配置或者为executors的_spark.executor.extraJavaOptions_配置增加属性：-_Dlog4j.configuration=&lt;location of configuration file&gt;_。注意，如果使用文件，应该明确提供file: 协议，并且这个文件对于所有的节点都应该本地性地存在。
* 更新`$SPARK_CONF_DIR/log4j.properties`文件，它会和其它的配置一起被自动的上传。注意，如果使用了多种方法来配置log4j，其它两种方法有更高的优先级。

对一第一种方式，executors和appliaciton master都会使用相同的log4j配置，这可能导致一些问题（例如，尝试写同一个日志文件）。

如果要指定YARN日志文件的存放位置以便YARN可以展示和聚合它们，使用log4j.properties中的spark.yarn.app.container.log.dir。例如，log4j.appender.file\_appender.File=${spark.yarn.app.container.log.dir}/spark.log。对于流应用，配置`RollingFileAppender`并且设置YARN的日志目录能够避免大日志文件引起的磁盘溢出，并且可以通过YARN的日志工具来访问日志。

如果要对application master和executors使用自定义的metrics.properties，更新$SPARK\_CONF\_DIR/metrics.properties文件。它会和其它配置一起上传，所以不用使用--file来手动指定。

## 5、重要提示（Important notes）

* 对核心的请求是否在调度决策中取决于使用的调度器和它的配置方式。
* 在cluster模式中，Spark executors和driver使用的本地目录是为YARN配置的本地目录（Hadoop YARN的配置_yarn.nodemanager.local-dirs_）；如果用户指定了spark.local.dir，会被忽略。在client模式中，Spark executors会使用为YARN配置的本地目录，而Spark driver会使用spark.local.dir配置的本地目录。这是因为Spark dirver不在YARN集群上运行，只有Spark executors在YARN集群上运行。
* --files选项和--archives选项支持类似于Hadoop那样用“\#”指定文件名。例如，可以指定：--files localtest.txt\#appSees.txt，这样会上传本地名为localtest.txt的文件到HDFS，但是会被链接到名字appSees.txt，并且在YARN上运行的应用应该使用appSees.txt这个名字来指向它。
* 如果--jars选项使用的是本地文件并且在cluster模式运行，那么可以使用SparkContext.addJar方法。如果使用HDFS、HTTP、HTTPS、或者FTP文件，就不用使用这个选项。

## 6、在安全的集群运行（Running in a Secure Cluster）

如[Spark安全章节](http://spark.apache.org/docs/2.2.0/security.html)所述，Kerberos被用于安全的Hadoop集群来验证与服务和客户端相关的主体。这允许客户端请求这些认证服务；这些服务授权给认证主体。

Hadoop服务发布hadoop tokens来授权对服务和数据的访问。客户端必须获得他们将要访问服务的tokens并且把它们和他们的应用放在一起随着应用一起在YARN集群中启动。

对于一个要和任意Hadoop文件系统、HBase、Hive等进行交互的Spark应用，它必须使用启动应用程序的用户的Kerberos凭证获取相关tokens----即，其身份将成为已启动的Spark应用程序的身份的主体。

这通常在启动时完成：在一个安全集群Spark将自动获取集群的默认Hadoop文件系统、HBase、Hive的token。

如果HBase在classpath中，HBase配置声明应用是安全的（例如，`hbase-site.xml`设置`hbase.security.authentication`为`kerberos`）并且spark.yarn.security.credentials.hbase.enabled没有被设置为false，就会获取HBase的token。

类似的，如果Hive在classpath中，它的配置包括一个hive.metastore.uris配置中的metadata store的URI并且spark.yarn.security.credentials.hive.enabled没有被设置为false，就会获取Hive的token。

如果应用需要和其它的安全Hadoop文件系统交互，必须在应用启动的时候明确请求访问这些集群的token。通过把它们列在spark.yarn.access.hadoopFileSystems属性中来完成。例如：

```
spark.yarn.access.hadoopFileSystems hdfs://ireland.example.org:8020/,webhdfs://frankfurt.example.org:50070/
```

Spark支持通过Java Services机制（java.util.ServiceLoader）来和其它安全意识集成。要实现这个，org.apache.spark.deploy.yarn.security.ServiceCredentialProvider的实现对Spark应该是可用的，并且要把他们的名字列在Jar文件的META-INF/services目录中对应的文件里。这些插件可已通过设置`spark.yarn.security.credentials.{service}.enabled`为`false`来禁用，{service}是凭证提供程序的名字。

## 7、配置外部混洗服务

要在YARN集群的每个上`NodeManager`启动Spark外部混洗服务，要：

* 使用YARN profile构建Spark，如果有预先打包好的发行包（pre-packaged distribution）跳过。
* 找到spark-&lt;version&gt;-yarn-shuffle.jar文件，如果是自己构建的Spark，它应该在`$SPARK_HOME/common/network-yarn/target/scala-<version>`目录中，如果是发行包（distribution）应该在yarn目录中。
* 把这个jar文件加到集群中所有的NodeManager的classpath中。
* 在每个节点的yarn-site.xml文件中，把spark\_shuffle加到yarn.nodemanager.aux-services，然后设置`yarn.nodemanager.aux-services.spark_shuffle.class`为`org.apache.spark.network.yarn.YarnShuffleService`。
* .通过设置`etc/hadoop/yarn-env.sh`中的YARN\_HEAPSIZE（默认1000）来增加节点的heap堆大小，来避免混洗中的垃圾回收问题。
* 重启集群的所有NodeManager。

当外部混洗服务运行在YARN上的时候，如下配置可用：

spark.yarn.shuffle.stopOnFailure：如果Spark混洗服务初始化失败的时候是否停止NodeManager。这能防止在没有运行Spark混洗服务的NodeManager上运行container而导致应用的失败。默认值为false。

## 8、使用Apache Oozie来启动应用

Apache Oozie可以把Spark应用作为工作流（workflow）的一部分启动。在安全集群中，启动的应用需要访问集群服务相关的tokens。如果Spark是用密钥表（keytab）启动的，这是自动的。如果Spark不是用keytab启动，设置安全性的责任必须转交给Oozie。

为安全集群配置Oozie的详情和为job获取凭证可以在对应版本的Oozie文档中的Authentication章节查看。

对于Spark应用，Oozie的工作流必须设置好以让Oozie获取需要应用的所有tokens。包括：

* YARN 资源管理器
* 本地Hadoop文件系统
* 被用作I/O源和目的的任意远程Hadoop文件系统
* Hive（如果用到）
* HBase（如果用到）
* YARN timeline服务器，如果应用需要和这交互

为了避免Spark尝试然后失败的获取Hive、HBase和远程HDFS的tokens，Spark配置必须配置为将服务的token收集失效。

Spark的的配置必须包括：

```
spark.yarn.security.credentials.hive.enabled false spark.yarn.security.credentials.hbase.enabled false
```

配置选项`spark.yarn.access.hadoopFileSystems`必须不设置。

## 9、Kerberos故障排除（Troubleshooting Kerberos）

debug Hadoop/Kerberos的问题是困难的。一个有用的技术是通过设置_HADOOP\_JAAS\_DEBUG_环境变量来开启Hadoop中Kerberos操作的额外的日志记录。

```
bash export HADOOP_JAAS_DEBUG=true
```

JDK类可以通过设置系统属性`sun.security.krb5.debug`和`sun.security.spnego.debug=true`、-Dsun.security.krb5.debug=true -Dsun.security.spnego.debug=true来开启它们的Kerberos和SENGO/REST认证的额外日志记录。

所有这些选项都可以在Application Master中开启：

```
spark.yarn.appMasterEnv.HADOOP_JAAS_DEBUG true spark.yarn.am.extraJavaOptions -Dsun.security.krb5.debug=true -Dsun.security.spnego.debug=true
```

最后，如果日志级别`org.apache.spark.deploy.yarn.Client`被设置为`DEBUG`，日志将会包括所有tokens获取和失效的详细信息。

## 10、使用Spark History Server来替换Spark Web UI（Using the Spark History Server to replace the Spark Web UI）

当应用UI失效时，使用Spark History Server应用页面作为运行的应用的跟踪URL是可能的。对于安全集群和出于降低Spark Driver内存使用的目的，这是推荐的。设置通过Spark History Server进行追踪的方法如下：

* 在应用方面，在Saprk的应用中将`spark.yarn.historyServer.allowTracking=true`。这会告诉Spark在应用的UI失效时使用history Server的URL作为追踪URL。
* 在Spark History Server方面，添加org.apache.spark.deploy.yarn.YarnProxyRedirectFilter到`spark.ui.filters`配置的过滤器列表中。

要注意的是，history server信息可能不会及时更新应用的状态。

