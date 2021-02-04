### [机器学习库（MLlib）导读](http://spark.apache.org/docs/2.2.0/ml-guide.html)

MLlib是Spark的机器学习库。它的目标是让实际的机器学习可扩展并且简单。它提供的工具包括：

- ML算法：常用的学习算法，比如分类（classification），回归（regression），聚类（clustering），和协同过滤（collaborative filtering）
- 特征工程（Featurization）：特征获取（extraction），转换（transformation），降维（dimensionality reduction），和选择（selection）
- Pipelines：用于构建（constructing），评估（evaluating），和调试（tuning）ML Pipelines的工具
- 持久化（Persistence）：保存和加载算法，模型和Pipelines
- 实用工具（Utilities）：线性代数（linear algebra），统计（statistics），数据处理（data handling），等等

#### 声明：基于DataFrame的API是主要API

##### 基于RDD的MLlib API处于维护模式

从Spark 2.0开始，包`spark.mllib`中基于RDD的APIs进入维护模式。Spark主要的机器学习API是`spark.ml`包中的基于DataFrame的API。

这意味着：

- MLlib仍然会对`spark.mllib`中的基于RDD的API进行bug fixes。
- MLlib不会往基于RDD的API添加新的特性
- 在Spark 2.x版本中，MLlib会往基于DataFrame的API中添加新特性以使它的特性与基于RDD的API达到均势
- 当基于DataFrame的API与基于RDD的API特性均势后（粗略估计在Spark 2.3），基于RDD的API会被废弃
- 基于RDD的API未来可能会被移除

MLlib转向基于DataFrame API的原因：

- DataFrame API比RDDs更加用户友好。DataFrames的好处包括Spark Datasources，SQL/DataFrame查询，Tungsten和Catalyst优化，跨语言统一的APIs。
- 基于DataFrame的MLlib API提供了跨ML算法和跨多语言统一的API。
- DataFrames有实际的ML Pipelines，尤其是特征转换

#### 依赖

MLlib使用了线性代数包Breeze，Breeze依赖netlib-java进行数值的处理。MLlib默认不包括netlib-java，要使用netlib-java可以添加`com.github.fommil.netlib:all:1.1.2` 依赖。

