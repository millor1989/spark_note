# 硬件配置（Hardware Provisioning）

Spark开发者的一个常见问题是怎样为Spark配置硬件。而正确的硬件配置要视情况而定，有如下推荐。

## 1、存储系统（Storage Systems）

大多数Spark的jobs要从外部系统读取输入数据（例如，Hadoop文件系统，或者HBase），把Spark配置得尽可能离文件系统近。建议是：

* 如果可能，在HDFS的同一个节点上运行Spark。最简单的方式是在同一个节点上设置起一个Spark standalone模式的集群，并且配置Spark和Hadoop的内存和CPU使用以避免干扰（对于Hadoop，相关的配置是为每个task内存设置mapred.child.jar.opts，为task的数量设置mapreduce.tasktracker.map.tasks.maximum和mapreduce.tasktracker.reduce.tasks.maximum）。也可以在像Mesos或者Hadoop YARN这样的常见集群管理器上运行Hadoop和Spark。
* 如果不行，在HDFS同一个机架的不同节点上运行Spark。
* 对于像HBase这样的低延时数据存储，为了比埋你干扰更倾向于在和存储系统不同的节点上运行运算jobs。

## 2、本地磁盘（Local Disks）

尽管Spark可以在内存中运行大量的运算，它仍然需要使用本地磁盘来存储（不适合在RAM的）数据和保存stages之间的中间输出数据。推荐每个节点有4-8个磁盘，不使用RAID（只是独立挂载separate mount points）。在Linux中，使用noatime选项挂载磁盘以减少不必要的写操作。在Spark中，配置spark.local.dir变量为逗号分割的本地磁盘列表。如果使用HDFS，使用和HDFS相同的磁盘也可以

## 3、内存（Memory）

通常，每个机器有8G-数百G内存情况下Spark都能运行良好。在所有情况下，都推荐只分配给Spark至多75%的内存，把剩余的内存留给操作系统和缓存。

使用多少内存要视应用程序而定。要确定应用为某个特定数据集使用的内存大小，把数据集的一部分加载到Spark RDD并且使用Spark的监控UI的Storage页面查看它在内存中的大小。注意，内存的使用受到存储级别（Storage Level）和序列化格式的极大影响----查看[调试指南](http://spark.apache.org/docs/2.2.0/tuning.html)以了解如何减少内存使用。

最后，要注意的是，在超过200G的RAM的情况下，Java的虚拟机不总是运行良好。如果使用超过这个数量的RAM的机器，可以在每个节点上运行多个工作者JVMs。在Spark的Standalone模式，可以在conf/spark-env.sh文件中设置`SPARK_WORKER_INSTANCES`参数来设置每个节点工作者（workers）的数量，通过设置SPARK\_WORKER\_CORES参数来设置每个工作者（workers）的核心数量。

## 4、网络（Network）

当数据已经在内存中时，许多Spark应用是网络绑定（network-bound）的。使用10G或者更高的网络是使这些应用运行更快的最好方法。对于像group-by、reduce-by和SQL join这些“分布式 reduce”应用尤其如此。对于任何给定的应用，都可以通过应用的监控UI查看Spark通过网络混洗的数据量。

## 5、CPU核心数（CPU Cores）

Spark可以很好的扩展到每台机器数十个核心，因为它线程间进行最小的共享。每台机器应该配置至少8-16个核心。根据CPU的工作负载，可能需要更多的核心：一旦数据已经在内存中，大多数应用不是CPU-bound就是network-bound。

