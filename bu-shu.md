#### Spark应用的部署（Deploying）

[文档原文](http://spark.apache.org/docs/2.2.0/cluster-overview.html)

简单介绍Spark如何在集群上运行

简单理解涉及的组件（component）

#### 词汇解释（Glossary）

* **Application**：构建在Spark上的用户程序。由集群上的_driver program_和_executors_组成。

* **Application jar**：包括用户的Spark应用的Jar文件。有时，jar文件包含spark应用和它的依赖。用户的jar不应该包括hadoop或者Spark库，因为这些文件在运行时会被加入。

* **Driver Program**：运行应用的main\(\)方法并创建SparkContext的进程。

* **Cluster Manager**：用来获取集群资源的外部服务。

* **Deploy mode**：区分驱动进程运行地方。cluster模式，框架在集群内部启动driver；client模式，submitter在集群外启动driver

* **Worker node**：集群中运行应用代码的任意节点

* **Executor**：Worker node上为应用而启动的运行tasks并且在内存或硬盘上保存数据的进程。每个应用都有自己的executors

* **Task**：发送到一个executor一个工作单元

* **Job**：由Spark的行动操作催生，多个tasks组成的并行运算。
* **Stage**：每个job都被划分成由一组的tasks组成的、相互依赖的stages。



