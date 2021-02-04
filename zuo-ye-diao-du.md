## 作业调度（Job Scheduling）

[文档原文](http://spark.apache.org/docs/2.2.0/job-scheduling.html)

首先，每个spark应用（SparkContext实例）作为一系列独立的executor进程运行。Spark运行的集群管理器提供跨应用调度（scheduling across applications）的工具。

其次，在每个Spark应用内部，多个“jobs”（Spark actions）可能同时运行如果它们由不同的线程提交。如果应用通过网络服务请求这是很常见的。Spark包括了一个fair scheduler（公平调度器）来在每个SparkContext内部调度资源。

### 1、应用间调度（Scheduling Across Applicaitons）

多用户共享同一个集群的时候，根据集群管理器的不同会有不同的资源分配方式。

最简单的在所有的集群管理器上都可用的调度模式是资源的静态分区（_static partitioning_），每个应用被给予一个它可以使用的最大数量的资源限制，直到应用的生命周期结束。

根据集群类型，资源分配可以有其它的配置：

* Standalone mode：默认情况下，被提交到standalone mode集群的应用会以FIFO（先进先出）的顺序运行，每个应用会尽可能的使用所有可用的节点。可以通过设置限制应用使用的最大节点数量，还可以通过设置限制应用使用的内存。
* Mesos：使用静态分区（_static partitioning_）时，要设置配置属性spark.mesos.coarse 为true；可以像Standalone mode那样限制节点数和内存的使用。Mesos也支持CPU核心的动态共享（_dynamic sharing_），要设置配置属性spark.mesos.coarse 为false，并使用mesos：//URL；在这种方式下，每个spark应用仍然有固定和独立的内存分配，但是如果应用不再运行tasks，其它应用会在这些核心上运行tasks。
* YARN： --num-executors 控制executors的数量 \(spark.executor.instances as configuration property\),  --executor-memory \(spark.executor.memory configuration property\) 和 --executor-cores \(spark.executor.cores configuration property\) 控制每个executor的资源.

所有的调度模式都**不支持跨应用的内存共享**。

### 1.1、**动态资源分配（Dynamic Resource Allocation）**

Spark提供了根据工作负载动态调整应用占用资源的机制。

这一机制默认是失效的，并且在所有的粗粒度集群管理器上可用。

首先要设置应用的属性spark.dynamicAllocation.enabled为true，其次要在同一集群的各个工作者节点设置一个外部混洗服务（_external shuffle service_），并且设置应用的属性spark.shuffle.service.enabled为true。外部混洗服务的目的是允许executors被删除而不删除它们写的混洗文件。设置外部混洗服务的方式因集群管理器不同而不同：

* standalone模式：设置属性 spark.shuffle.service.enabled为true然后启动工作者即可。
* Mesos粗粒度模式：设置属性 spark.shuffle.service.enabled为true，然后在从节点（slave nodes）运行_$SPARK\_HOME/sbin/start-mesos-shuffle-service.sh_
* YARN模式：比较复杂，[看这里](http://spark.apache.org/docs/2.2.0/running-on-yarn.html#configuring-the-external-shuffle-service)

#### 1.1.1、资源分配策略（Resource Allocation Policy）

需要有确定何时移除executor（Remove Policy）何时加入executor（Request Policy）的策略。

###### 请求策略Request Policy

开启动态分配的Spark应用在有未执行的tasks需要被调度的时候会请求额外的executors。这种情况意味着，已经存在的executors不足以同时满足已经提交但是未完成的tasks。

Spark轮流的请求executors。如果等待的tasks已经等待了_spark.dynamicAllocation.schedulerBacklogTimeout_秒才会真正触发请求，如果等待的tasks仍然在请求executors每隔_spark.dynamicAllocation.sustainedSchedulerBacklogTimeout_秒会再次触发请求。此外，请求的executors的数量相比上一轮的请求成倍增长。例如，一个应用第一次请求一个executor，然后就在随后的请求中请求2、4、8...个executors。

成倍增长策略的动机是twofold。首先，一个应用应该在开始的时候谨慎地请求executors，以防止资源只够启动很少的额外的executors。这与TCP慢启动（TCP slow start）的原理相呼应。其次，应用应该能够及时地扩大自己使用的资源以防止事实上需要更多的executors。

###### 移除策略Remove Policy

移除executors的策略更简单。Spark应用在executor闲置时间超过spark.dynamicAllocation.executorIdleTimeout秒时移除这个executor。Note.在很多情况下，移除条件与请求条件是相互独立的，因为如果有tasks等待调度executor不应该闲置。

#### 1.1.2、优雅的停止Executors（Gracefule Decommission of Executors）

对于非动态分配，一个Spark executor在失败时候退出或者在相关的应用退出时退出。在这两种情况下，executor相关的所有状态都不在需要并且可以安全的丢弃。对于动态分配，executor被移除时应用仍然在运行。如果应用尝试获取这个被移除executor保存或写的状态时，就会再次计算所需要的状态。所以，Spark需要一个优雅的移除executor时保存状态的机制。

这一机制对于混洗尤其重要。在混洗中，Spark executor首先把它的map输出写到本地磁盘，然后当其它executors尝试获取这些输出文件时扮演一个这些输出文件的服务器的角色。如果有些tasks运行时间比同批次tasks运行时间长很多，动态分配可能会在混洗结束之前移除executor，这样，这个executor写的混洗文件就会不必要的再次计算。

保留混洗文件的解决方案是使用外部混洗服务（_external shuffle sevice_）。这个服务只想一个长时间运行的进程，这个进程独立于Spark 应用和应用的executor运行在集群的每一个节点上。如果这个服务是可用的，Spark的executor就会从这个服务获取混洗文件而不是从其它executor获取。这就意味着，任何executor写的混洗state都可以在executor生命周期之外持续存在。

除了混洗文件，executor页缓存数据到磁盘或内存。当executor被移除，所有的缓存数据都将不复存在。为了减轻这种情况，默认情况下executor包含的缓存数据不会被移除，可以通过属性spark.dynamicAllocation.cachedExecutorIdleTimeout配置这个行为。后续版本可能会采用类似外部混洗服务的形式来处理executor的缓存数据。

### 2、应用内部调度（Scheduling Within an Application）

Spark应用内部，多个并行的jobs可能同时运行。Spark的调度器是线程安全的并且支持这种情况让应用服务多个请求。

默认情况下，Spark的调度器以FIFO的方式运行jobs。每个job被划分为stages，第一个job如果有tasks要运行，它有使用所有可用资源的优先级，然后是第二个job获得优先级，等等。如果队列前面的job不需要使用所有的可用资源，之后的jobs可以马上运行，如果前面的jobs很大，后面的jobs可能会很明显的延迟。

从Spark 0.8开始，可以通过配置实现jobs之间公平共享（_fair sharing_）。在fair sharing模式下，Spark以“round robin”方式分配jobs中的tasks，以达到所有的jobs粗略的公平共享集群的资源。这意味着，在耗时长的job在执行的时候，耗时短的job可以被提交而不用等耗时长的job结束。这个模式最合适于多用户设置。

启用fair scheduler，只用设置SparkContext属性_spark.scheduler.mode_为**FAIR**即可：

```scala
val conf = new SparkConf().setMaster(...).setAppName(...)
conf.set("spark.scheduler.mode", "FAIR")
val sc = new SparkContext(conf)
```

#### 2.1、公平调度池（Fair Scheduler Pools）

公平调度也支持把jobs分组到_pools_中，并为不同的pool设置不同的调度选项（例如，权重）。这对于为重要的jobs设置“高优先级”的pool来说很有用，也可以按用户将不同用户的的jobs进行分组并让用户公平共享资源（不管有多少同时进行的jobs）而不是让jobs公平共享资源。这种方式是模仿的Hadoop的Fair Scheduler。

不用任何介入，新提交的jobs进入默认的pool，但是jobs的pools可以在提交它们的线程中通过**SparkContext**的“local property”spark.scheduler.pool来设置：

```scala
// Assuming sc is your SparkContext variable
sc.setLocalProperty("spark.scheduler.pool", "pool1")
```

设置过这个local property之后，这个线程内提交的所有的jobs都会使用这个pool名称。这个设置是针对线程的，这让一个线程代表一个用户运行多个jobs变得简单。如果要清除线程关联的pool，调用如下代码：

```scala
sc.setLocalProperty("spark.scheduler.pool", null)
```

#### 2.2、Pools的默认行为（Default Behavior of Pools）

默认情况下，每个pool公平共享集群的资源，但是在pool内部，jobs按照FIFO的顺序运行。例如，为每个用户创建一个pool，这意味着每个用户公平共享集群资源，每个用户的queries会顺序执行而不是后来的queries从用户较早的查询获取资源。

#### 2.3、配置Pool属性（Configuring Pool Properties）

指定pools的属性也可以通过配置文件修改。每个pool支持三个属性：

* schedulingMode：可以是FIFO或者FAIR，来控制pool内的jobs是按序执行还是公平共享pool的资源。
* weight：这个配置控制pool相对于其它poll分享的集群资源。默认情况下所有的pools的weight都是1。如果给一个pool的比重设置为2，那么它将会获得比其他活跃的pools多两倍的资源。设置一个较高的weight例如1000，在本质上也意味着在pools中拥有较高的优先级，weight-1000的pool如果有jobs要执行它总会优先启动tasks。
* minShare：除了weight，每个pool可以设置一个最小的资源分配。公平调度（fair scheduler）在根据weight分配资源之前总是会尝试满足所有活跃pools的最小分享资源。因此，minShare属性能确保pool快速获取一定数量的资源。

pool的属性可以通过XML文件设置，类似于_conf/fairscheduler.xml.template_，需要设置**SparkConf**的属性spark.scheduler.allocation.file：

```scala
conf.set("spark.scheduler.allocation.file", "/path/to/file")
```

XML文件的格式是每个pool一个&lt;pool&gt;元素，例如：

```xml
<?xml version="1.0"?>
<allocations>
  <pool name="production">
    <schedulingMode>FAIR</schedulingMode>
    <weight>1</weight>
    <minShare>2</minShare>
  </pool>
  <pool name="test">
    <schedulingMode>FIFO</schedulingMode>
    <weight>2</weight>
    <minShare>3</minShare>
  </pool>
</allocations>
```

完整的例子是_conf/fairscheduler.xml.template_。

Note.没有在XML文件中配置的pools的属性都是默认值（schedulingMode:FIFO,weight:1,minShare:0）。

