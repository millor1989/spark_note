### 聚类（Clustering）

聚类是一个非监督式学习问题，基于某些相似性概念将数据分组。聚类通常用于探索性的分析并且（或者）作为一个层级式的监督式学习管道（其中为每个集群训练不同的classifier或者回归模型）的一个组件。

#### 1、K-means

k-means是一个最常用的聚类算法，将数据点聚集到一个预定义了个数的不同集群中。MLlib实现包括了一个并行的`k-means++`算法的变种——叫作`kmeans||`（K Mean Parallel）。

`KMeans`实现为一个`Estimator`，生成`KMeansModel`作为基本模型。

参数：

- `k`：期望的集群数量。注意，可能会返回少于k个集群，比如，只有少于k个的不同数据点
- `maxIter`：运行的最大迭代次数
- `initMode`：指定随机初始化或者通过`kmeans||`初始化
- `initSteps`：指定`k-means||`算法的步数
- `tol`：公差（tolerance）

计算集合内平方误差和（WSSSE，Within Set Sum of Squared Error）。通过增加`k`可以减少这个误差指标。在实际应用中，最优的`k`通常是WSSSE图像“拐点”地方的`k`。

##### 1.1、输入列

| Param name  | Type(s) | Default    | Description |
| ----------- | ------- | ---------- | ----------- |
| featuresCol | Vector  | "features" | 特征向量    |

##### 1.2、输出列

| Param name    | Type(s) | Default      | Description              |
| ------------- | ------- | ------------ | ------------------------ |
| predictionCol | Int     | "prediction" | Predicted cluster center |

##### 1.3、例

```scala
import org.apache.spark.ml.clustering.KMeans

// Loads data.
val dataset = spark.read.format("libsvm").load("data/mllib/sample_kmeans_data.txt")

// Trains a k-means model.
val kmeans = new KMeans().setK(2).setSeed(1L)
val model = kmeans.fit(dataset)

// Evaluate clustering by computing Within Set Sum of Squared Errors.
val WSSSE = model.computeCost(dataset)
println(s"Within Set Sum of Squared Errors = $WSSSE")

// Shows the result.
println("Cluster Centers: ")
model.clusterCenters.foreach(println)
```

#### 2、Latent Dirichlet Allocation（LDA，隐蒂雷克分配）

隐蒂雷克分配（LDA）是一个话题（topic）模型，从一个文本文档的集合中推导话题。LDA可以被想象为一个如下的聚类算法：

- 话题对应于集群中心（cluster centers），文档对应于数据集中的例子（记录）。
- 话题和文档都存在于特征空间中，特征向量是字数（文字包）的向量。
- LDA使用基于文本文档如何产生的统计模型的函数，而不是使用传统的距离来评估一次聚类。

LDA通过`setOptimizer`函数支持不同的推导算法（inference algorithm）。`EMLDAOptimizer`使用基于似然函数（likelihood function）的期望最大化学习聚类，并生成全面的结果，而`OnlineLDAOptimizer`使用在线变分推理（online variational inference）的重复最小批次（mini-batch）抽样，并且通常是内存友好的。

LDA把文档集合作为字数的向量输入，支持如下参数：

- `k`：话题数量（即，几群中心数量）
- `optimizer`：用于学习LDA模型的Optimizer，`EMLDAOptimizer`或`OnlineLDAOptimizer`
- `docConcentration`：优先于文档话题分布的Dirichlet参数。值越大，推断的分布越平滑。
- `topicConcentration`：优先于话题的词条（单词）分布的Dirichlet参数。值越大，推断的分布越平滑。
- `maxIterations`：迭代次数的限制
- `checkpointInterval`：如果使用了checkpointing，这个参数制定了创建checkpoints的频率。如果`maxIterations`很大，使用checkpointing可以帮助降低磁盘上混洗文件的大小并帮助进行故障的恢复

所有的`spark.mllib`LDA模型都支持：

- `describeTopics`：将话题返回为最重要词条和词条权重的数组
- `topicsMatrix`：返回一个`k`矩阵的`vocabSize`，其中每列都是一个话题。

**注意**：LDA还是一个开发中的试验性特性。所以，某些特性仅仅适用于两种优化器（或通过优化器生成的模型）的某一种。目前，分布式模型可以转换为本地模型，但是反之则不行。





`LDA`被实现为`Estimator`，生成一个`LDAModel`作为基本模型。专业的用于可以根据需要将`EMLDAOptimizer`生成的`LDA`模型转换为`DistributedLDAModel`。

```scala
import org.apache.spark.ml.clustering.LDA

// Loads data.
val dataset = spark.read.format("libsvm")
  .load("data/mllib/sample_lda_libsvm_data.txt")

// Trains a LDA model.
val lda = new LDA().setK(10).setMaxIter(10)
val model = lda.fit(dataset)

val ll = model.logLikelihood(dataset)
val lp = model.logPerplexity(dataset)
println(s"The lower bound on the log likelihood of the entire corpus: $ll")
println(s"The upper bound on perplexity: $lp")

// Describe topics.
val topics = model.describeTopics(3)
println("The topics described by their top-weighted terms:")
topics.show(false)

// Shows the result.
val transformed = model.transform(dataset)
transformed.show(false)
```