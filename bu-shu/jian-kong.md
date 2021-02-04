### Monitoring and Instrumentation

[文档原文](http://spark.apache.org/docs/2.2.0/monitoring.html)

监控Spark应用的方法：Web UIs、metrics和外部的工具（external instrumentation）。

##### 网络接口（Web Interfaces）

每个驱动程序（driver program）都有Web UI，每个_SparkContext_都会启动Web UI，默认端口是4040，通过Web UI展示了运行的tasks、executors和存储的使用。

内容包括：

* 调度stages和tasks列表
* RDD的大小和内存使用汇总
* 环境信息
* 运行的executors的信息

用浏览器地址[http://&lt;driver-node&gt;:4040来访问这个UI](http://<driver-node>:4040来访问这个UI。)。如果在同一主机上运行了多个SparkContext，它们对应的WebUI端口是由4040开始的连续端口（4041、4041 etc）。

**Note.**默认情况下，只有在应用执行期间才可以访问Web UI。

如果要在应用结束后访问，需要在应用启动之前把属性_spark.eventLog.enabled_ 设置为true。

##### 应用结束后查看（Viewing After the Fact）

通过Spark的history Server，也可以访问应用的事件日志UI。

启动方法：

```
./sbin/start-history-server.sh
```

默认创建一个web接口 [http://&lt;server-url&gt;:18080，展示未完成、已完成的应用和尝试。](http://<server-url>:18080，展示未完成、已完成的应用和尝试。)

History Server的配置详见[链接](http://spark.apache.org/docs/2.2.0/monitoring.html#spark-configuration-options)

当使用文件系统提供类（属性_spark.history.provider_）时，必须通过设置属性_spark.history.fs.logDirectory_提供基础的日志目录。

Spark作业（jobs）必须配置为可记录事件，并把事件记录到相同分片的、可写的目录。

例如要记录日知道目录hdfs://namenode/shared/spark-logs，客户端侧的选项应该为：

```
spark.eventLog.enabled true
spark.eventLog.dir hdfs://namenode/shared/spark-logs
```

**Note.**

1、history Server展示完成的和未完成的Spark jobs。如果一个应用失败后尝试了多次，也会显示失败的尝试，包括正在执行的未完成的尝试和最终成功的尝试

2、未完成应用日志是间歇性更新的。更新间隔通过属性_spark.history.fs.update.interval_设置。在大的集群上，更新间隔应该尽量大。查看正在执行应用的信息的正确方式是查看这个引用的Web UI而不是history Server。

3、没有标示自己结束的应用在history server中被标记为未完成，即使应用不在运行。如果应用崩溃可能会有这种情况。

4、标识Spark Job结束的一种方式是明确的结束SparkContext（如调用sc.stop\(\)）。

##### **REST API**

除了UI，Spark还提供了监控API，以方便用户自己构建个性化的Web UI。不做记录了。

##### **Metrics**

好像和REST API作用类似，也不做记录了。

##### **高级工具（Advanced Instrumentation）**

可以使用外部工具来展示Spark jobs的表现：

Cluster-wide monitoring tools,例如：Ganglia

OS profiling tools：dstat、iostat、iotop

JVM工具：如jstack、jmap、jstat、jconsole

