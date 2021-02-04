## 决策树

### 应用：

`spark.ml`的决策树API与`spark.mllib`中基于RDD的决策树API的主要区别于：

- 支持ML Pipelines
- 用于分类和回归的决策树算法的分离
- 使用DataFrame metadata来区分连续特征和类别特征

Pipelines API对`spark.ml`决策树算法的支持，相比于基于RDD的API更丰富。

特别是，对于分类，用户可以获得每种类型的预测的概率（即，class conditional probabilities，类型条件概率）；对于回归，用户可以获得预测的有偏样本方差（biased sample variance）。

#### 1、输入和输出

这里列出了输入和输出（预测）列类型。所有的输出列都是可选的，通过设置对应的Param为空字符串可以排除输出列。

##### 1.1、输入列

| 参数名      | 类型   | Default    | 描述         |
| ----------- | ------ | ---------- | ------------ |
| labelCol    | Double | "label"    | 要预测的标签 |
| featuresCol | Vector | "features" | 特征向量     |

##### 1.2、输出列

| 参数名           | 类型   | Default         | 描述                                                         | 注意事项 |
| ---------------- | ------ | --------------- | ------------------------------------------------------------ | -------- |
| predictionCol    | Double | "prediction"    | 预测的标签                                                   |          |
| rawPredictionCol | Vector | "rawPrediction" | 类型长度向量, 包括用于进行预测的位于树节点的训练实例标签的数量 | 分类专用 |
| probabilityCol   | Vector | "probability"   | 类型长度向量，等于归一化为多项式分布的rawPrediction          | 分类专用 |
| varianceCol      | Double |                 | 预测的有偏样本方差                                           | 回归专用 |

### 原理：

决策树和它们的团体是机器学习分类和回归任务的流行方法。决策树应用广泛，因为它们易于解释，易于处理类别特征，易于扩展到多元分类设置，不需要特征缩放，并且可以捕获非线性和特征相互作用。树团体算法，例如随即森林和boosting，它们是分类和回归任务中表现一流的算法。

`spark.ml`实现支持将决策树用于二元和多元分类，以及回归，可以用于连续特征和类别特征。它的实现按行分区数据，可以对百万级甚至十亿级的数据进行分布式的训练。

#### 1、**基础算法**

决策树是贪婪的算法，对特征空间进行递归二元分区。决策树为每个最底层（叶子）分区预测相同的标签。为了最大化树节点增加的信息，通过从一个可能的分裂集合中选择最佳的分裂（best split）贪婪地进行每个分区选择。换句话说，在每个树节点，是从集合$\underset{s}{\operatorname{argmax}} IG(D,s)$选择分裂的，

其中$$IG(D,s)$$是将分裂$$s$$应用到数据集$$D$$时增加的信息。

##### 1.1、节点的不纯度（Node impurity）和信息增加（information gain）

节点的不纯度（node impurity）是在某个节点，对标签的同质性（homogeneity）的衡量。`spark.mllib`的当前实现为分类提供了两种不纯度衡量方法（Gini impurity，和entropy——熵），为回归提供了一种不纯度衡量方法（variance——方差）。

| Impurity      | Task           | Formula                                      | Description                                                  |
| ------------- | -------------- | -------------------------------------------- | ------------------------------------------------------------ |
| Gini impurity | Classification | $$\sum_{i=1}^{C} f_i(1-f_i)$$                | $$f_i$$是某个节点的标签$$i$$的频率，$$C$$是不同标签的个数    |
| Entropy       | Classification | $$\sum_{i=1}^{C} -f_ilog(f_i)$$              | $$f_i$$是某个节点的标签$$i$$的频率，$$C$$是不同标签的个数    |
| Variance      | Regression     | $$\frac{1}{N} \sum_{i=1}^{N} (y_i - \mu)^2$$ | $$y_i$$是某个实例的标签，$$N$$是实例的数量，$$\mu$$是由$$\frac{1}{N} \sum_{i=1}^N y_i$$决定的平均值 |

信息增加（information gain）是父节点的不纯度和两个子节点的不纯度的加权和（weighted sum）的差（difference）。假如，分裂$$s$$将大小为$$N$$的数据集$$D$$分为两个数据集$$D_{left}$$（大小为$$N_{left}$$）和$$D_{right}$$（大小为$$N_{right}$$），那么信息增加为：

$$IG(D,s) = Impurity(D) - \frac{N_{left}}{N} Impurity(D_{left}) - \frac{N_{right}}{N} Impurity(D_{right})$$

##### 1.2、候选分裂（Split candidates）

**连续特征**

对于在单台机器中小数据集的实现，每个连续特征的候选分裂对于特征来说一般是唯一的。为了更快的树计算，某些实现首先对特征值进行排序，然后把有序的唯一值作为候选分裂。

对于大型的分布式数据集排序是高开销的。此时的实现是，通过对部分抽样的数据执行分位数计算（quantile calculation），计算出一个候选分裂地近似集合。使用有序的分裂创建“箱子”（“bins”），通过参数`maxBins`可以指定这些箱子的最大数量。

注意，箱子的数量不能大于实例$$N$$的数量（一种罕见的情况，因为默认的`maxBins`值是32）。如果条件不满足，树算法会自动地减少箱子的数量。

**类别特征**

对于有$$M$$个可能值（类别）的类别特征，可能会有最多$$2^{M-1} - 1$$个候选分裂。对于二元（0/1）分类和回归，可以通过按照平均标签（average label）排序类别特征值将候选分裂的数量减少到$$M - 1$$。例如，对于一个二元分类问题，它的一个类别特征具有三个类别A，B，C并且它们与标签1的对应比例是0.2，0.6，0.4，这个类别特征会被排序为A，C，B。两种候选分裂是A|C,B和A,C|B其中|表示分裂。

在多元分类中，所有$$2^{M - 1} - 1$$个可能分裂会在任何可能的时间使用。当$$2^{M-1} - 1$$大于参数`maxBins`时，使用的是一个与二元分类和回归使用的方法类似的（启发式的）方法。$$M$$个类别特征值按照不纯度排序，结果是需要考虑的$$M - 1$$个候选分裂。

##### 1.3、停止规则（stopping rule）

当如下条件中的某一个被满足时，递归的树结构会在某个节点停止：

1. 节点深度等于`maxDepth`训练参数；
2. 没有候选分裂能够使信息增加大于`minInfoGain`；
3. 没有候选分裂产生每个节点都有最少`minInstancesPerNode`个训练实例的子节点

#### 2、使用提示（Usage tips）

##### 2.1、问题规范参数（Problem specification parameters）

这些参数描述了要解决的问题和数据集。只用指定，不用调试。

- **algo**：决策树类型，`Classification`或者`Regression`
- **numClasses**：类型的个数（`Classification`专用）
- **categoricalFeaturesInfo**：指明那些特征是类别的，并且这些每个特征有多少个类别值。这个参数以特征索引对特征arity（类别数量）的map的形式指定。不在这个map中的特征会被当作连续特征对待。
  - 例如，`Map(0->2,4->10)`指明特征`0`是二元的（值为`0`或`1`），特征`4`有10个类别（值为`{0,1,2,3,4,5,6,7,8,9}`）。注意，特征的索引是从0开始的：特征`0`和`4`是一个特征向量实例的第一个和第五个元素。
  - 注意，`categoricalFeaturesInfo`不是必须指定的。不指定，算法也能运行，并且可能得到合理的结果。但是，如果类别特征指定的合适，性能会更好。

##### 2.2、停止准则（stopping criteria）

这些参数决定了树什么时间停止构建（增加新的节点）。当调试这些参数的时候，要仔细的在预留的测试数据上进行验证以避免过拟合。

- **maxDepth**：树的最大深度。树越深开销越大（潜在地，更加精确），但是训练成本更高并且更可能过拟合
- **minInstancesPerNode**：对于一个要进一步分裂的节点，它的每个子节点必须有最少这个数量的训练实例。这个参数通常用于`RandomForest`，因为它常常训练深度比单个的树更深。
- **minInfoGain**：对于一个要进一步分裂的节点，分裂必须增加最少这个数量（就信息增加而言）

##### 2.3、可调试参数（Tunable parameters）

这些参数是可调试的。当调试这些参数的时候，要仔细的在预留的测试数据上进行验证以避免过拟合。

- **maxBins**：离散化连续特征时使用的箱子个数。
  - 增加`maxBins`让算法可以考虑更多的候选分裂并且使分裂决定更细粒度。但是增加了运算量和通信量
  - 注意，对于任何的类别特征，`maxBins`参数必须至少是类别$$M$$的最大数量。
- **maxMemoryInMB**：用于收集充足统计数据的内存的量。
  - 默认值被保守地设置为256MB，以让决策算法能够在大多数场景下运行。增加`maxMemoryInMB`可以通过减少数据的传递导致更快的训练（如果内存足够）。但是，随着`maxMemoryInMB`的增加，由于每次迭代的通信量与`maxMemoryInMB`是成比例的，收益可能会减少。
  - 实现详情：为了更快的处理，决策树算法会收集要分裂的节点分组（而不是每次一个节点）的统计数据。一组要被处理的节点的数量由需要的内存决定（随每个特征而变化）。`maxMemoryInMB`参数指明了每个工作者可以用于这些数据统计的内存限制（MB）。
- **subsamplingRate**：用于学习决策树的训练数据的因子。这个参数对于训练树团体（使用`RandomForest`和`GradientBoostedTrees`）最相关，获取源数据的子样本（subsample）是很有用的。对于训练单个决策树，这个参数没那么有用，因为训练实例的数量通常不是主要的限制。
- **impurity**：用于候选分裂选择的不存度衡量。必须与`algo`参数匹配。

##### 2.4、缓存和检查点（caching and checkpointing）

MLlib 1.2增加了一些用于扩展到更大（更深）树和树团体的一些特性。当`maxDepth`被设置为很大时，开启节点ID缓存和检查点是有用的。这些参数对于`numTrees`设置的很大的`RandomForest`也是很有用的。

- **useNodeIdCache**：如果设置为true，算法会避免在每次迭代时把当前的模型（tree or trees）传递到executors。
  - 对于深树（提升工作者的运算速度）和大型随机森林（减少每次迭代的通信量）有用
  - 事项详情：默认情况下，算法将当前模型传达到executors以便executors可以将训练实例和树节点匹配。当开启这项设置，算法将缓存当前模型。

节点ID缓存会生成一系列的RDDs（每次迭代1个）。这个长的血缘关系会导致性能问题，但是对中间RDDs进行checkpointing可以减轻这些问题。注意，checkpointing仅仅适用于`useNodeIdCache`设置为true的场合。

- **checkpointDir**：用于checkpointing节点ID缓存RDDs的目录
- **checkpointInterval**：checkpointing节点ID缓存RDDs的频率。设置的太低回导致额外的写HDFS开销；设置的太高会导致当executors故障时，RDD需要重新计算。

#### 3、缩放（scaling）

运算量随着训练实例数量、特征数量和`maxBins`参数近似线性地缩放。

通信量随着特征的数量和`maxBins`参数近似线性地缩放。

MLlib决策树实现的算法可以读取稀疏数据和密集数据，但是，没有优化稀疏输入。

#### 4、例

##### 4.1、分类

```scala
import org.apache.spark.mllib.tree.DecisionTree
import org.apache.spark.mllib.tree.model.DecisionTreeModel
import org.apache.spark.mllib.util.MLUtils

// Load and parse the data file.
val data = MLUtils.loadLibSVMFile(sc, "data/mllib/sample_libsvm_data.txt")
// Split the data into training and test sets (30% held out for testing)
val splits = data.randomSplit(Array(0.7, 0.3))
val (trainingData, testData) = (splits(0), splits(1))

// Train a DecisionTree model.
//  Empty categoricalFeaturesInfo indicates all features are continuous.
val numClasses = 2
val categoricalFeaturesInfo = Map[Int, Int]()
val impurity = "gini"
val maxDepth = 5
val maxBins = 32

val model = DecisionTree.trainClassifier(trainingData, numClasses, categoricalFeaturesInfo,
  impurity, maxDepth, maxBins)

// Evaluate model on test instances and compute test error
val labelAndPreds = testData.map { point =>
  val prediction = model.predict(point.features)
  (point.label, prediction)
}
val testErr = labelAndPreds.filter(r => r._1 != r._2).count().toDouble / testData.count()
println("Test Error = " + testErr)
println("Learned classification tree model:\n" + model.toDebugString)

// Save and load model
model.save(sc, "target/tmp/myDecisionTreeClassificationModel")
val sameModel = DecisionTreeModel.load(sc, "target/tmp/myDecisionTreeClassificationModel")
```

##### 4.2、回归

```scala
import org.apache.spark.mllib.tree.DecisionTree
import org.apache.spark.mllib.tree.model.DecisionTreeModel
import org.apache.spark.mllib.util.MLUtils

// Load and parse the data file.
val data = MLUtils.loadLibSVMFile(sc, "data/mllib/sample_libsvm_data.txt")
// Split the data into training and test sets (30% held out for testing)
val splits = data.randomSplit(Array(0.7, 0.3))
val (trainingData, testData) = (splits(0), splits(1))

// Train a DecisionTree model.
//  Empty categoricalFeaturesInfo indicates all features are continuous.
val categoricalFeaturesInfo = Map[Int, Int]()
val impurity = "variance"
val maxDepth = 5
val maxBins = 32

val model = DecisionTree.trainRegressor(trainingData, categoricalFeaturesInfo, impurity,
  maxDepth, maxBins)

// Evaluate model on test instances and compute test error
val labelsAndPredictions = testData.map { point =>
  val prediction = model.predict(point.features)
  (point.label, prediction)
}
val testMSE = labelsAndPredictions.map{ case (v, p) => math.pow(v - p, 2) }.mean()
println("Test Mean Squared Error = " + testMSE)
println("Learned regression tree model:\n" + model.toDebugString)

// Save and load model
model.save(sc, "target/tmp/myDecisionTreeRegressionModel")
val sameModel = DecisionTreeModel.load(sc, "target/tmp/myDecisionTreeRegressionModel")
```

