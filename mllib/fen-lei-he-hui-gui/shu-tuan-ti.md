## 树团体

### 应用：

DataFrame API支持两种主要的树团体算法：随机森林（Random Froests）和梯度提升树（Gradient-Boosted Trees，GBTs）。它们都是用`spark.ml`的决策树作为基本模型。

`spark.ml`树团体API和`spark.mllib`树团体API的主要区别：

- 支持DataFrame和ML Pipelines
- 分类和回归分开
- 使用DataFrame metadata来区分连续特征和类别特征
- 随机森林算法功能更多：评估特征的重要性，对于分类也可以预测每个类型的概率（即，类型的条件概率）

#### 1、随机森林

随机森林是决策树的团体。为了降低过拟合的风险，随机森林组合了很多的决策树。`spark.ml`实现支持将随机森林用于二元和多元的分类以及回归，支持连续特征和类别特征。

##### 1.1、输入和输出

输出列是可选的，通过设置对应的Param为空字符串可以排除掉输出列。

##### 输入列

| Param name  | Type(s) | Default    | Description  |
| ----------- | ------- | ---------- | ------------ |
| labelCol    | Double  | "label"    | 要预测的标签 |
| featuresCol | Vector  | "features" | 特征向量     |

##### 输出列

| Param name       | Type(s) | Default         | Description                                                  | Notes      |
| ---------------- | ------- | --------------- | ------------------------------------------------------------ | ---------- |
| predictionCol    | Double  | "prediction"    | 预测的标签                                                   |            |
| rawPredictionCol | Vector  | "rawPrediction" | 类型长度向量, 包括用于进行预测的位于树节点的训练实例标签的数量 | 仅用于分类 |
| probabilityCol   | Vector  | "probability"   | 类型长度向量，等于归一化为多项式分布的rawPrediction          | 仅用于分类 |

#### 2、梯度提升树

梯度提升树是决策树团体。为了最小化损失函数GBTs迭代地训练决策树。`spark.ml`实现支持将GBTs用于二元和多元的分类以及回归，支持连续特征和类别特征。

##### 2.1、输入和输出

输出列是可选的，通过设置对应的Param为空字符串可以排除掉输出列。

##### 输入列

| Param name  | Type(s) | Default    | Description  |
| ----------- | ------- | ---------- | ------------ |
| labelCol    | Double  | "label"    | 要预测的标签 |
| featuresCol | Vector  | "features" | 特征向量     |

注意，`GBTClassifier`目前只支持二元标签

##### 输出列

| Param name    | Type(s) | Default      | Description | Notes |
| ------------- | ------- | ------------ | ----------- | ----- |
| predictionCol | Double  | "prediction" | 预测的标签  |       |

未来，`GBTClassifier`也会包括输出列`rawPrediction`和`probability`。

### 原理：

所有的团体算法都是创建一个由一组其它基本模型构成的模型的算法。`spark.mllib`支持两种主要的团体算法：`GradientBoostedTrees`和`RandomFroest`，都是使用决策树作为基本模型。

#### 1、梯度提升树vs.随机森林

GBTs和随机森林都是用来学习树团体的算法，但是训练过程不同。有一些实践上的权衡：

- GBTs一次训练一个树，训练时间比随机森林长。随机森林可以并行地训练多个树。
  - 另外，通常将GBTs（而不是随机森林）和比较小（浅）的树一起使用是合理的，训练较小的树耗时更短
- 随机森林比较不倾向于过拟合。在一个随机森林中训练越多的树，过拟合的风险越小，但是训练更多的树反而会增加GBTs的过拟合风险。（用统计学语言来说，随机森林通过使用更多的树减少了方差，而GBTs使用更多的树较小了偏移（bias））
- 随机森林的性能提升与树的数量是单调的，调试更容易（而树的数量过大会降低GBTs的性能）

简而言之，两种算法都可以很高效，应该基于实际的数据集选择算法。

#### 2、随机森林

与决策树类似，随机森林处理类别特征，扩展到多元分类设置，不需要特征缩放，能够达到非线性和特征相互作用。

`spark.mllib`实现支持将随机森林用于二元和多元的分类以及回归，支持连续特征和类别特征。`spark.mllib`使用既存的决策树实现来实现随机森林。

##### 2.1、基本算法

随机森林单独地训练一组决策树，所以可以并行地进行训练。算法将随机性注入到训练过程中，以便每个决策树有些不同。合并每个树的预测结果减小了预测的方差，提升了在测试数据上的表现。

###### 2.1.1、训练

训练过程随机性的注入包括：

- 每次迭代都从源数据集获取子样本以获得不同的训练集（即，bootstrapping）
- 在每个树节点上考虑不同的分隔特征的随机子集

除了这些随机化，随机森林的决策树的训练方式与单个的决策树相同

###### 2.1.2、预测

要在新的实例上进行预测，随机森林必须聚合它的决策树集合的预测结果。对于分类和回归，聚合方式是不同的。

对于分类：多数票（majority vote）。每个树的预测都被作为某个类型的一票。预测的标签是票数最多的类型。

对于回归：平均（averaging）。每个树预测一个实数，预测的标签是所有树预测结果的平均值。

##### 2.2、使用提示

最重要的两个参数，调试它们常常可以提升性能：

- **numTrees**：森林中树的个数。
  - 增加树的个数会减小预测的方差，提升模型的测试期精确性。
  - 训练时间与树的数量大致呈线性增加
- **maxDepth**：森林中每个树的最大深度。
  - 增加树的深度使模型更强大且富有表现力（expressive）。但是，深树训练时间长且更倾向于过拟合。
  - 通常，使用随机森林是要比使用单个决策树时训练更深的树。单个树比随机森林更可能过拟合（因为随机森林中多个树的叠加减小了方差）

如下两个参数通常不用调试，但是为了提升训练速度可以对它们进行调试：

- **subsamplingRate**：这个参数指定了用于训练森林中每个树的数据集的大小，即原始数据集大小的一个因子。推荐使用默认值（1.0），但是降低这个因子可以加快训练速度。
- **featureSubsetStrategy**：在每个树节点用做候选分裂地特征数量。这个数值被指定为一个特征总数的因子或者一个函数。降低这个数值可以加速训练，但是如果太小有时会影响性能表现。

##### 2.3、例

###### 2.3.1、分类

```scala
import org.apache.spark.mllib.tree.RandomForest
import org.apache.spark.mllib.tree.model.RandomForestModel
import org.apache.spark.mllib.util.MLUtils

// Load and parse the data file.
val data = MLUtils.loadLibSVMFile(sc, "data/mllib/sample_libsvm_data.txt")
// Split the data into training and test sets (30% held out for testing)
val splits = data.randomSplit(Array(0.7, 0.3))
val (trainingData, testData) = (splits(0), splits(1))

// Train a RandomForest model.
// Empty categoricalFeaturesInfo indicates all features are continuous.
val numClasses = 2
val categoricalFeaturesInfo = Map[Int, Int]()
val numTrees = 3 // Use more in practice.
val featureSubsetStrategy = "auto" // Let the algorithm choose.
val impurity = "gini"
val maxDepth = 4
val maxBins = 32

val model = RandomForest.trainClassifier(trainingData, numClasses, categoricalFeaturesInfo,
  numTrees, featureSubsetStrategy, impurity, maxDepth, maxBins)

// Evaluate model on test instances and compute test error
val labelAndPreds = testData.map { point =>
  val prediction = model.predict(point.features)
  (point.label, prediction)
}
val testErr = labelAndPreds.filter(r => r._1 != r._2).count.toDouble / testData.count()
println("Test Error = " + testErr)
println("Learned classification forest model:\n" + model.toDebugString)

// Save and load model
model.save(sc, "target/tmp/myRandomForestClassificationModel")
val sameModel = RandomForestModel.load(sc, "target/tmp/myRandomForestClassificationModel")
```

###### 回归

```scala
import org.apache.spark.mllib.tree.RandomForest
import org.apache.spark.mllib.tree.model.RandomForestModel
import org.apache.spark.mllib.util.MLUtils

// Load and parse the data file.
val data = MLUtils.loadLibSVMFile(sc, "data/mllib/sample_libsvm_data.txt")
// Split the data into training and test sets (30% held out for testing)
val splits = data.randomSplit(Array(0.7, 0.3))
val (trainingData, testData) = (splits(0), splits(1))

// Train a RandomForest model.
// Empty categoricalFeaturesInfo indicates all features are continuous.
val numClasses = 2
val categoricalFeaturesInfo = Map[Int, Int]()
val numTrees = 3 // Use more in practice.
val featureSubsetStrategy = "auto" // Let the algorithm choose.
val impurity = "variance"
val maxDepth = 4
val maxBins = 32

val model = RandomForest.trainRegressor(trainingData, categoricalFeaturesInfo,
  numTrees, featureSubsetStrategy, impurity, maxDepth, maxBins)

// Evaluate model on test instances and compute test error
val labelsAndPredictions = testData.map { point =>
  val prediction = model.predict(point.features)
  (point.label, prediction)
}
val testMSE = labelsAndPredictions.map{ case(v, p) => math.pow((v - p), 2)}.mean()
println("Test Mean Squared Error = " + testMSE)
println("Learned regression forest model:\n" + model.toDebugString)

// Save and load model
model.save(sc, "target/tmp/myRandomForestRegressionModel")
val sameModel = RandomForestModel.load(sc, "target/tmp/myRandomForestRegressionModel")
```

#### 3、梯度提升树

GBTs是决策树的团体。GBTs为了最小化损失函数迭代地训练决策树。与决策树类似，GBTs处理类别特征，扩展到多元分类设置，不需要特征缩放，能够达到非线性和特征相互作用。

`spark.mllib`支持GBTs进行二元分类和回归，可以使用连续特征和类别特征。`spark.mllib`使用既存的决策树实现来实现GBTs。

注意：GBTs目前不支持多元分类。多元分类问题，可以使用决策树或随机森林。

##### 3.1、基本算法

梯度提升迭代地训练一系列的决策树。每次迭代，算法使用当前的团体预测每个训练实例的标签，然后和实际的标签（true label）进行比较。数据集被重新打标签（re-labeled）以更加侧重于预测结果不理想的训练实例。这样，在下次迭代中，决策树会帮助纠正此前的预测失误。

用于为实例重打标签（re-labeling）的专用机制通过损失函数定义。每次迭代，GBTs基于训练数据进一步减小这个损失函数。

###### 3.1.1、损失函数

如下表格是`spark.mllib`的GBTs目前支持的损失函数。注意，每个损失函数适用于分类或者回归，而不是都适用。

其中：$$N$$ = 实例数量。$$y_i$$ = 实例$$i$$的标签。$$x_i$$ = 实例$$i$$的特征。$$F(x_i)$$ = 模型的对实例$$i$$的预测标签。

| Loss           | Task | Formula                                          | Description                                    |
| -------------- | ---- | ------------------------------------------------ | ---------------------------------------------- |
| Log Loss       | 分类 | $$2 \sum_{i=1}^{N} \log(1+\exp(-2 y_i F(x_i)))$$ | Twice binomial negative log likelihood         |
| Squared Error  | 回归 | $$\sum_{i=1}^{N} (y_i - F(x_i))^2$$              | 也叫L2 loss。回归任务的默认损失函数            |
| Absolute Error | 回归 | $$\sum_{i=1}^{N} \|y_i - F(x_i)\|$$              | 也叫L1 loss。对于离群值比Squared Error更健壮。 |

##### 3.2、使用提示

一些参数：

- **loss**：不同的数据，选择不同的损失函数，会有显著的不同结果
- **numIterations**：设置团体中树的数量。每次迭代产生一个树。增加迭代次数会使模型更具表现力，提升训练数据精度。但是，如果太大测试期的精度可能会反而更小。
- **learningRate**：这个参数不必调试。如果算法行为似乎不稳定，减小这个值可能提升稳定性。
- **algo**：使用树【战略，Strategy】参数设置算法或者任务（分类 vs. 回归）

###### 3.2.1、边训练边验证

训练更多的树的时候梯度提升会过拟合。为了防止过拟合，边训练边验证是有用的。使用方法`runWithValidation`来使用这个选项。它有一对RDD参数，第一个是训练数据集，第二个是验证数据集。

当验证误差（validation error）的改善不超过一定的公差（tolerance，由`BoostingStrategy`的`validationTol`参数提供）时，训练终止。在实践中，验证误差先减少，后增加。可能存在验证误差不是单调地改变的情况，建议用户设置足够大的负公差（negative tolerance）并使用`evaluateEachIteration`（获取每次迭代的误差或损失，error or loss）检查验证曲线以调试迭代次数。

##### 3.3、例

###### 3.3.1、分类

```scala
import org.apache.spark.mllib.tree.GradientBoostedTrees
import org.apache.spark.mllib.tree.configuration.BoostingStrategy
import org.apache.spark.mllib.tree.model.GradientBoostedTreesModel
import org.apache.spark.mllib.util.MLUtils

// Load and parse the data file.
val data = MLUtils.loadLibSVMFile(sc, "data/mllib/sample_libsvm_data.txt")
// Split the data into training and test sets (30% held out for testing)
val splits = data.randomSplit(Array(0.7, 0.3))
val (trainingData, testData) = (splits(0), splits(1))

// Train a GradientBoostedTrees model.
// The defaultParams for Classification use LogLoss by default.
val boostingStrategy = BoostingStrategy.defaultParams("Classification")
boostingStrategy.numIterations = 3 // Note: Use more iterations in practice.
boostingStrategy.treeStrategy.numClasses = 2
boostingStrategy.treeStrategy.maxDepth = 5
// Empty categoricalFeaturesInfo indicates all features are continuous.
boostingStrategy.treeStrategy.categoricalFeaturesInfo = Map[Int, Int]()

val model = GradientBoostedTrees.train(trainingData, boostingStrategy)

// Evaluate model on test instances and compute test error
val labelAndPreds = testData.map { point =>
  val prediction = model.predict(point.features)
  (point.label, prediction)
}
val testErr = labelAndPreds.filter(r => r._1 != r._2).count.toDouble / testData.count()
println("Test Error = " + testErr)
println("Learned classification GBT model:\n" + model.toDebugString)

// Save and load model
model.save(sc, "target/tmp/myGradientBoostingClassificationModel")
val sameModel = GradientBoostedTreesModel.load(sc,
  "target/tmp/myGradientBoostingClassificationModel")
```

###### 3.3.2、回归

```scala
import org.apache.spark.mllib.tree.GradientBoostedTrees
import org.apache.spark.mllib.tree.configuration.BoostingStrategy
import org.apache.spark.mllib.tree.model.GradientBoostedTreesModel
import org.apache.spark.mllib.util.MLUtils

// Load and parse the data file.
val data = MLUtils.loadLibSVMFile(sc, "data/mllib/sample_libsvm_data.txt")
// Split the data into training and test sets (30% held out for testing)
val splits = data.randomSplit(Array(0.7, 0.3))
val (trainingData, testData) = (splits(0), splits(1))

// Train a GradientBoostedTrees model.
// The defaultParams for Regression use SquaredError by default.
val boostingStrategy = BoostingStrategy.defaultParams("Regression")
boostingStrategy.numIterations = 3 // Note: Use more iterations in practice.
boostingStrategy.treeStrategy.maxDepth = 5
// Empty categoricalFeaturesInfo indicates all features are continuous.
boostingStrategy.treeStrategy.categoricalFeaturesInfo = Map[Int, Int]()

val model = GradientBoostedTrees.train(trainingData, boostingStrategy)

// Evaluate model on test instances and compute test error
val labelsAndPredictions = testData.map { point =>
  val prediction = model.predict(point.features)
  (point.label, prediction)
}
val testMSE = labelsAndPredictions.map{ case(v, p) => math.pow((v - p), 2)}.mean()
println("Test Mean Squared Error = " + testMSE)
println("Learned regression GBT model:\n" + model.toDebugString)

// Save and load model
model.save(sc, "target/tmp/myGradientBoostingRegressionModel")
val sameModel = GradientBoostedTreesModel.load(sc,
  "target/tmp/myGradientBoostingRegressionModel")
```