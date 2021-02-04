### 线性方法（Linear Methods）

#### 1、数学公式（Mathematical formulation）

许多标准的机器学习方法都可以被表示为**凸优化**（Convex optimization）问题。例如，有一个$$d$$个元素的变量向量$$w$$（代码中叫作`weights`），查找基于该向量的凸函数（convex function）$$f$$的最小值的任务。可以将这个任务写作一个优化问题$$\min_{w \in R^d} \; f(w)$$，其中**目标函数**（objective function）是$$f(w)  := \lambda\, R(w) + \frac1n \sum_{i=1}^n L(w;x_i,y_i)$$的形式。其中向量$$x_i\in R^d$$是训练数据例子（training data examples），$$1\le i\le n$$，并且$$y_i \in R$$是对应的标签，即要预测的结果。如果$$L(w; x,y)$$可以表示为$$w^T$$和$$y$$的一个函数，则把这个方法称为是**线性的**。

**目标函数**$$f$$有两个部分：正则化部分（regularizer）控制模型的复杂度（避免过拟合）；损失部分（loss）衡量模型在训练数据上的误差（error）。损失函数（loss function，也称代价函数：cost function）$$L(w;.)$$一般是$$w$$的一个凸函数。固定的正则化参数$$\lambda \ge 0$$（代码中的`regParam`）定义了两个目标：最小化损失（即，训练误差，training error）和最小化模型复杂度（即，为了避免过拟合）之间的权衡（trade-off）。

##### 1.1、损失函数

`spark.mllib`支持的损失函数和它们的梯度（gradients，升降率）或者次梯度：

|  | 损失函数$$L(w;x,y)$$ | 梯度或次梯度 |
| --- | --- | --- |
| hinge loss | $$\max \{0, 1-y w^T x \}, \quad y \in \{-1, +1\}$$ | 太复杂的公式，写不粗来 |
| logsitc loss | $$\log(1+\exp( -y w^T x)),\quad y \in \{-1, +1\}$$ | $$-y \left(1-\frac1{1+\exp(-y w^T x)} \right) \cdot x$$ |
| squared loss | $$\frac{1}{2} (w^T x - y)^2, \quad y \in R$$ | $$(w^T x - y) \cdot x$$ |

注意，在上面的数学公式中，二元标签$$y$$的值为$$+1$$（positive）或$$-1$$（negative），对公式来说很方便。但是，在`spark.mllib`中，用$$0$$代替$$-1$$以与多元分类标签保持一致。

##### 1.2、正则化

正则化的目的是使模型简单、和避免过拟合。`spark.mllib`支持的正则化方法有：

|  | 正则化方法$$R(w)$$ | 梯度或次梯度 |
| --- | --- | --- |
| zero（不正则化） | 0 | $$0$$ |
| L2 | $$\frac{1}{2}\|w\|_2^2$$ | $$w$$ |
| L1 | $$\|w\|_1$$ | $$\mathrm{sign}(w)$$ |
| elastic net | $$\alpha \|w\|_1 + (1-\alpha)\frac{1}{2}\|w\|_2^2$$ | $$\alpha \mathrm{sign}(w) + (1-\alpha) w$$ |

其中，$$\mathrm{sign}(w)$$是所有元素$$w$$的signs $$\pm1$$构成的。

因为平滑度（smoothness）的原因，L2正则化问题比L1正则化问题解决起来简单。但是L1正则化可以帮助提升权重稀疏性（sparsity in weights），进而可以是模型更小，更可解释，后者适合特征选择。

Elastic net是L1和L2正则化的组合。在数学上，Elastic Net被定义为$$L_1$$和$$L_2$$正则化项的一个凸组合（convex combination）：

$$\alpha \left( \lambda \|w\|_1 \right) + (1-\alpha) \left( \frac{\lambda}{2}\|w\|_2^2 \right) , \alpha \in [0, 1], \lambda \geq 0$$

通过适当地设置$$\alpha$$，elastic net包含了$$L_1$$和$$L_2$$正则化作为自己的特殊情况。例如，当使用$$\alpha$$为$$1$$的elastic net来训练线性回归模型时，它等价于Lasso模型。当$$\alpha$$被设置为$$0$$时，训练的模型简化为一个岭回归（ridge regression）模型。

不推荐不进行任何正则化就训练模型，尤其是训练样本数量小的时候。

##### 1.3、优化

线性方法使用凸优化（convex optimization）方法来优化目标函数。目前大多数算法APIs支持随机梯度下降（Stochastic Gradient Descent ，SGD），有些支持L-BFGS。

#### 2、分类

分类（Classification）的目的是把条目划分为类别（categories）。最常见的分类是二元分类（binary classification），有两种类别。如果有超过两种类别，就叫作多类型分类（multiclass classification）。`spark.mllib`支持两种可以用于分类的线性方法：线性支持向量机（linear Support Vector Machines，SVMs）和逻辑回归。线性支持向量机只支持二元分类，而逻辑回归支持二元分类和多元分类。对这两种方法，`spark.mllib`都支持L1和L2正则化变种。在MLlib中，训练数据集合由LabeledPoint的RDD代表，其中标签是从零开始的类型索引：$$0,1,2,...$$。

##### 2.1、线性支持向量机

线性支持向量机是一个用于大规模分类任务的标准方法。他是一个线性方法，损失函数是higne loss。默认的是使用L2正则化，也支持L1正则化。

线性SVMs算法的输出是SVM模型。用$$x$$表示新的数据点（new data point），模型基于$$w^Tx$$的值进行预测。默认情况下，如果$$w^Tx \ge 0$$结果为positive，否则为negative。

##### 例

```scala
import org.apache.spark.mllib.classification.{SVMModel, SVMWithSGD}
import org.apache.spark.mllib.evaluation.BinaryClassificationMetrics
import org.apache.spark.mllib.util.MLUtils

// Load training data in LIBSVM format.
val data = MLUtils.loadLibSVMFile(sc, "data/mllib/sample_libsvm_data.txt")

// Split data into training (60%) and test (40%).
val splits = data.randomSplit(Array(0.6, 0.4), seed = 11L)
val training = splits(0).cache()
val test = splits(1)

// Run training algorithm to build the model
val numIterations = 100
val model = SVMWithSGD.train(training, numIterations)

// Clear the default threshold.
model.clearThreshold()

// Compute raw scores on the test set.
val scoreAndLabels = test.map { point =>
  val score = model.predict(point.features)
  (score, point.label)
}

// Get evaluation metrics.
val metrics = new BinaryClassificationMetrics(scoreAndLabels)
val auROC = metrics.areaUnderROC()

println("Area under ROC = " + auROC)

// Save and load model
model.save(sc, "target/tmp/scalaSVMWithSGDModel")
val sameModel = SVMModel.load(sc, "target/tmp/scalaSVMWithSGDModel")
```

`SVMWithSGD.train()`方法默认把正则化参数设置为1.0执行L2正则化。如果要配置这个算法，可以通过直接创建一个新的对象并嗲用setter方法来自定义`SVMWithSGD`。所有其它的`spark.mllib`算法也都支持这种自定义方式。比如，如下代码生成一个正则化参数为0.1的L1正则化的SVMs变种，并且用200次迭代训练算法：

```scala
import org.apache.spark.mllib.optimization.L1Updater

val svmAlg = new SVMWithSGD()
svmAlg.optimizer
  .setNumIterations(200)
  .setRegParam(0.1)
  .setUpdater(new L1Updater)
val modelL1 = svmAlg.run(training)
```

##### 2.2、逻辑回归

逻辑回归广发用于预测二元响应，损失函数是logistic loss。

对于二元分类问题，这个算法输出一个二元逻辑回归模型。用$$x$$表示新的数据点（new data point），$$z = w^T x$$，模型通过应用逻辑函数$$\mathrm{f}(z) = \frac{1}{1 + e^{-z}}$$进行预测。默认情况下，如果$$\mathrm{f}(z) \gt 0.5$$结果为positive，否则为negative。

二元逻辑回归可以被一般化为多项式逻辑回归，以进行多元分类训练和预测。例如，对于$$K$$个可能的输出结果，其中某个结果可以被设定为一个“pivot”，其它的$$K - 1$$个结果会分别地相对于（against）pivot结果进行回归。在`spark.mllib`中第一个类别$$0$$被设定为“pivot”类。

对于多元分类问题，算法会输出一个多项式逻辑回归模型，它包含$$K - 1$$个相对于（against）第一个类别的二元逻辑回归模型。输入新的数据点，$$K-1$$个模型会运行，概率最大的类别会被作为预测结果类别。

`spark.mllib`实现了两种解决逻辑回归的算法：mini-batch gradient descent和L-BFGS。推荐使用后者，因为后者收敛的更快。

##### 例

```scala
import org.apache.spark.mllib.classification.{LogisticRegressionModel, LogisticRegressionWithLBFGS}
import org.apache.spark.mllib.evaluation.MulticlassMetrics
import org.apache.spark.mllib.regression.LabeledPoint
import org.apache.spark.mllib.util.MLUtils

// Load training data in LIBSVM format.
val data = MLUtils.loadLibSVMFile(sc, "data/mllib/sample_libsvm_data.txt")

// Split data into training (60%) and test (40%).
val splits = data.randomSplit(Array(0.6, 0.4), seed = 11L)
val training = splits(0).cache()
val test = splits(1)

// Run training algorithm to build the model
val model = new LogisticRegressionWithLBFGS()
  .setNumClasses(10)
  .run(training)

// Compute raw scores on the test set.
val predictionAndLabels = test.map { case LabeledPoint(label, features) =>
  val prediction = model.predict(features)
  (prediction, label)
}

// Get evaluation metrics.
val metrics = new MulticlassMetrics(predictionAndLabels)
val accuracy = metrics.accuracy
println(s"Accuracy = $accuracy")

// Save and load model
model.save(sc, "target/tmp/scalaLogisticRegressionWithLBFGSModel")
val sameModel = LogisticRegressionModel.load(sc,
  "target/tmp/scalaLogisticRegressionWithLBFGSModel")
```

#### 3、回归

##### 3.1、Linear least squares，Lasso，和ridge regression

线性最小平方（Linear least squares）是最常用的回归问题公式。它是一个使用squared loss作为损失函数的的线性方法。

不同回归方法按照使用的正则化类型不同进行区分：ordinary least squares或者linear least squares不使用正则化；ridge regression使用L2正则化；Lasso使用L1正则化。对于所有这些模型，平均损失或者训练误差，$$\frac{1}{n} \sum_{i=1}^n (w^T x_i - y_i)^2$$被称为均方误差（mean squared error）。

##### 例

```scala
import org.apache.spark.mllib.linalg.Vectors
import org.apache.spark.mllib.regression.LabeledPoint
import org.apache.spark.mllib.regression.LinearRegressionModel
import org.apache.spark.mllib.regression.LinearRegressionWithSGD

// Load and parse the data
val data = sc.textFile("data/mllib/ridge-data/lpsa.data")
val parsedData = data.map { line =>
  val parts = line.split(',')
  LabeledPoint(parts(0).toDouble, Vectors.dense(parts(1).split(' ').map(_.toDouble)))
}.cache()

// Building the model
val numIterations = 100
val stepSize = 0.00000001
val model = LinearRegressionWithSGD.train(parsedData, numIterations, stepSize)

// Evaluate model on training examples and compute training error
val valuesAndPreds = parsedData.map { point =>
  val prediction = model.predict(point.features)
  (point.label, prediction)
}
val MSE = valuesAndPreds.map{ case(v, p) => math.pow((v - p), 2) }.mean()
println("training Mean Squared Error = " + MSE)

// Save and load model
model.save(sc, "target/tmp/scalaLinearRegressionWithSGDModel")
val sameModel = LinearRegressionModel.load(sc, "target/tmp/scalaLinearRegressionWithSGDModel")
```

##### 3.2、流式线性回归（Streaming linear regression）

当数据以流的形式到达，在线拟合回归模型是很有用的，当新的数据到达时更新模型的参数。`spark.mllib`目前支持使用ordinary least squares的流线性回归。在线拟合与离线拟合类似，除了拟合发生在每个批次的数据上，以便模型持续地更新以从流反映数据（reflect the data from the stream）。

##### 例

`args(0)`训练数据集文本文件，`args(1)`用于预测的数据集文本文件。

```scala
import org.apache.spark.mllib.linalg.Vectors
import org.apache.spark.mllib.regression.LabeledPoint
import org.apache.spark.mllib.regression.StreamingLinearRegressionWithSGD

val trainingData = ssc.textFileStream(args(0)).map(LabeledPoint.parse).cache()
val testData = ssc.textFileStream(args(1)).map(LabeledPoint.parse)

val numFeatures = 3
val model = new StreamingLinearRegressionWithSGD()
  .setInitialWeights(Vectors.zeros(numFeatures))

model.trainOn(trainingData)
model.predictOnValues(testData.map(lp => (lp.label, lp.features))).print()

ssc.start()
ssc.awaitTermination()
```

