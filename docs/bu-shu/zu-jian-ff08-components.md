Spark应用作为一系列独立进程（_processes_）运行在集群上，这些进程由主程序（驱动程序，_driver program_ ）的_SparkContext_对象进行协调。

![](/assets/cluster-overview.png)

SparkContext可以和多种类型的集群管理器（包括Spark的standalone集群管理器、Mesos、YARN）进行连接，由集群管理器分配应用的资源。和集群连接后，Spark在集群中的节点上获取_executors_，_executors_是为Spark应用运行运算和保存数据的进程（_processes_）。接下来，把应用代码（由传送给_SparkContext_的Jar文件或者Python文件定义）发送给_executors_。最后，_SparkContext_把_tasks_发送给_executors_来运行。

**Notes**：

1、每个应用都有自己的_executor_进程，_executor_以多线程运行_tasks_并且在应用持续周期内一直存在。这样，应用之间在调度方面（每个_driver_调度自己的_tasks_）和执行方面（不同应用的tasks在不同的JVM中运行）都可以相互独立。可是，这也意味着不同Spark应用（_SparkContext_实例）之间无法共享数据，除非把数据写到外部存储系统。

2、Spark对于底层的集群管理器是无关的（agnostic不可知的）。只要可以获取_executor_进程，这些进程可以相互通信，即使在也支持其他应用程序的集群管理器上运行它也相对容易。

3、驱动程序（driver program）必须在整个生命周期中监听和接收他的executors的连接。所以，驱动程序对于工作节点（_worker nodes_）必须是网络可寻址的。

4、因为驱动程序调度集群上的任务，它应该运行在离工作节点较近的地方，最好是在相同的本地网络环境。如果要向远程集群发送请求，最好为驱动程序打开一个RPC并在它附近执行提交操作，而不是在远离工作节点的地方运行驱动程序。

