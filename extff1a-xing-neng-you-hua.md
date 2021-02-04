来自美团技术团队

[https://tech.meituan.com/tags/spark.html](https://tech.meituan.com/tags/spark.html)



##### Spark相较于MapReduce的优势：

一方面，MapReduce计算模型对多轮迭代的DAG作业支持不好，每轮迭代都需要将数据存储到硬盘，极大地影响了作业执行效率；另外只提供Map和Reduce两种算子，使得用户在实现迭代式计算（比如，机器学习算法）时成本高且效率低。

另一方面，Spark可以结合SQL查询和复杂的过程式逻辑处理高效地进行半结构化或者非结构化数据的ETL处理。

在美团的应用：

- Spark结合Zeepelin的交互式开发平台

  Zeepelin支持多种解释器：Spark、Pyspark、SQL。Zeepelin可以连接线上集群

- 基于Spark的ETL

- 基于Spark的用户特征平台

- 基于Spark的数据挖掘平台

- Spark用于交互式用户行为分析系统

- Spark用于SEM（Search Engine Marketing，搜索引擎营销）投放服务



##### Spark调优方案:

- 开发调优

- 资源调优

- 数据倾斜调优

- shuffle调优


