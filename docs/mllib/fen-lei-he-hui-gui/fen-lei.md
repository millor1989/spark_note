### 分类（Classification）

#### 1、逻辑回归（Logistic regression）

逻辑回归是一种预测类别的响应（categorical response）的流行方法。它是广义线性模型（Generalized Linear Models）的一个特殊情况，用于预测结果的概率。在`spark.ml`中，使用二项式逻辑回归（binomial logistic regression）可以预测二元结果，使用多项式逻辑回归（multinomial logistic regression）可以预测多类型结果。使用`family`参数来选择使用的算法，如果不设置Spark会推导正确的变种。

> 通过把`family`参数设置为“multinomial”，多项式逻辑回归可以用于二元分类（binary classification）。它会产生两组稀疏和两个截距。
>
> 当使用常量非零列拟合没有截距的LogisticRegressionModel时，Spark MLlib输出的常量非零列的系数是0。这种行为与R glmnet相同，但是与LIBSVM不同。

##### 1.1、二项式逻辑回归

##### 例

如下例子展示了如何用弹性网正则化（elastic net regularization，elastic net是L1正则化和L2正则化的组合）训练二元分类的二项式和多项式逻辑回归模型。`elasticNetParam`对应于$$\alpha$$，`regParam`对应于$$\lambda$$。

```scala
import org.apache.spark.ml.classification.LogisticRegression

// Load training data
val training = spark.read.format("libsvm").load("data/mllib/sample_libsvm_data.txt")

val lr = new LogisticRegression()
  .setMaxIter(10)
  .setRegParam(0.3)
  .setElasticNetParam(0.8)

// Fit the model
val lrModel = lr.fit(training)

// Print the coefficients and intercept for logistic regression
println(s"Coefficients: ${lrModel.coefficients} Intercept: ${lrModel.intercept}")

// We can also use the multinomial family for binary classification
val mlr = new LogisticRegression()
  .setMaxIter(10)
  .setRegParam(0.3)
  .setElasticNetParam(0.8)
  .setFamily("multinomial")

val mlrModel = mlr.fit(training)

// Print the coefficients and intercepts for logistic regression with multinomial family
println(s"Multinomial coefficients: ${mlrModel.coefficientMatrix}")
println(s"Multinomial intercepts: ${mlrModel.interceptVector}")
```

`spark.ml`的逻辑回归实现也支持提取训练数据集的模型摘要（summary）。注意，`BinaryLogisticRegressionSummary` 中保存在`DataFrame`中的预测（predictions）和指标（metrics）是注解为`@transient`的，因此只在driver上有效。

`LogisticRegressionTrainingSummary`提供了一个`LogisticRegressionModel`的摘要。目前，只支持二元分类，并且摘要必须明确地转换为`BinaryLogisticRegressionTrainingSummary`。当支持多类型分类的时候，可能会有所不同。

继续之前的例子：

```scala
import org.apache.spark.ml.classification.{BinaryLogisticRegressionSummary, LogisticRegression}

// Extract the summary from the returned LogisticRegressionModel instance trained in the earlier
// example
val trainingSummary = lrModel.summary

// Obtain the objective per iteration.
val objectiveHistory = trainingSummary.objectiveHistory
println("objectiveHistory:")
objectiveHistory.foreach(loss => println(loss))

// Obtain the metrics useful to judge performance on test data.
// We cast the summary to a BinaryLogisticRegressionSummary since the problem is a
// binary classification problem.
val binarySummary = trainingSummary.asInstanceOf[BinaryLogisticRegressionSummary]

// Obtain the receiver-operating characteristic as a dataframe and areaUnderROC.
val roc = binarySummary.roc
roc.show()
println(s"areaUnderROC: ${binarySummary.areaUnderROC}")

// Set the model threshold to maximize F-Measure
val fMeasure = binarySummary.fMeasureByThreshold
val maxFMeasure = fMeasure.select(max("F-Measure")).head().getDouble(0)
val bestThreshold = fMeasure.where($"F-Measure" === maxFMeasure)
  .select("threshold").head().getDouble(0)
lrModel.setThreshold(bestThreshold)
```

##### 1.2、多项式逻辑回归

通过多项式逻辑（softmax）回归可以支持多类型分类。在多项式逻辑回归中，算法生成$$K$$组系数，或者一个$$K \times J$$的矩阵，其中$$K$$是输出类型的数量，$$J$$是特征的数量。如果用截距项（intercept term）来拟合算法，那么还会生成一个长度为$$K$$的截距向量。

>多项式系数为`coefficientMatrix`，截距为`interceptVector`。

>用多项式family训练的逻辑回归模型不支持`coefficients`和`intercept`方法，要用`coefficientMatrix`和`interceptVector`方法替代。

输出结果类型的条件概率（conditional probabilities）$$k \in {1,2,...,K}$$是使用softmax函数建模的。

```scala
import org.apache.spark.ml.classification.LogisticRegression

// Load training data
val training = spark
  .read
  .format("libsvm")
  .load("data/mllib/sample_multiclass_classification_data.txt")

val lr = new LogisticRegression()
  .setMaxIter(10)
  .setRegParam(0.3)
  .setElasticNetParam(0.8)

// Fit the model
val lrModel = lr.fit(training)

// Print the coefficients and intercept for multinomial logistic regression
println(s"Coefficients: \n${lrModel.coefficientMatrix}")
println(s"Intercepts: ${lrModel.interceptVector}")
```

#### 2、决策树分类器

决策树（Decision tree）是一个流行的分类和回归算法家族。

##### 例

如下例子，从LibSVM格式文件加载一个数据集，将它分为训练集合测试集；在训练数据上进行模型训练，在测试数据上进行评估。

```scala
import org.apache.spark.ml.Pipeline
import org.apache.spark.ml.classification.DecisionTreeClassificationModel
import org.apache.spark.ml.classification.DecisionTreeClassifier
import org.apache.spark.ml.evaluation.MulticlassClassificationEvaluator
import org.apache.spark.ml.feature.{IndexToString, StringIndexer, VectorIndexer}

// Load the data stored in LIBSVM format as a DataFrame.
val data = spark.read.format("libsvm").load("data/mllib/sample_libsvm_data.txt")

// Index labels, adding metadata to the label column.
// Fit on whole dataset to include all labels in index.
val labelIndexer = new StringIndexer()
  .setInputCol("label")
  .setOutputCol("indexedLabel")
  .fit(data)
// Automatically identify categorical features, and index them.
val featureIndexer = new VectorIndexer()
  .setInputCol("features")
  .setOutputCol("indexedFeatures")
  .setMaxCategories(4) // features with > 4 distinct values are treated as continuous.
  .fit(data)

// Split the data into training and test sets (30% held out for testing).
val Array(trainingData, testData) = data.randomSplit(Array(0.7, 0.3))

// Train a DecisionTree model.
val dt = new DecisionTreeClassifier()
  .setLabelCol("indexedLabel")
  .setFeaturesCol("indexedFeatures")

// Convert indexed labels back to original labels.
val labelConverter = new IndexToString()
  .setInputCol("prediction")
  .setOutputCol("predictedLabel")
  .setLabels(labelIndexer.labels)

// Chain indexers and tree in a Pipeline.
val pipeline = new Pipeline()
  .setStages(Array(labelIndexer, featureIndexer, dt, labelConverter))

// Train model. This also runs the indexers.
val model = pipeline.fit(trainingData)

// Make predictions.
val predictions = model.transform(testData)

// Select example rows to display.
predictions.select("predictedLabel", "label", "features").show(5)

// Select (prediction, true label) and compute test error.
val evaluator = new MulticlassClassificationEvaluator()
  .setLabelCol("indexedLabel")
  .setPredictionCol("prediction")
  .setMetricName("accuracy")
val accuracy = evaluator.evaluate(predictions)
println("Test Error = " + (1.0 - accuracy))

val treeModel = model.stages(2).asInstanceOf[DecisionTreeClassificationModel]
println("Learned classification tree model:\n" + treeModel.toDebugString)
```

#### 3、随即森林分类器

随机森林（Random forest）是一个流行的分类和回归算法家族。

##### 例

```scala
import org.apache.spark.ml.Pipeline
import org.apache.spark.ml.classification.{RandomForestClassificationModel, RandomForestClassifier}
import org.apache.spark.ml.evaluation.MulticlassClassificationEvaluator
import org.apache.spark.ml.feature.{IndexToString, StringIndexer, VectorIndexer}

// Load and parse the data file, converting it to a DataFrame.
val data = spark.read.format("libsvm").load("data/mllib/sample_libsvm_data.txt")

// Index labels, adding metadata to the label column.
// Fit on whole dataset to include all labels in index.
val labelIndexer = new StringIndexer()
  .setInputCol("label")
  .setOutputCol("indexedLabel")
  .fit(data)
// Automatically identify categorical features, and index them.
// Set maxCategories so features with > 4 distinct values are treated as continuous.
val featureIndexer = new VectorIndexer()
  .setInputCol("features")
  .setOutputCol("indexedFeatures")
  .setMaxCategories(4)
  .fit(data)

// Split the data into training and test sets (30% held out for testing).
val Array(trainingData, testData) = data.randomSplit(Array(0.7, 0.3))

// Train a RandomForest model.
val rf = new RandomForestClassifier()
  .setLabelCol("indexedLabel")
  .setFeaturesCol("indexedFeatures")
  .setNumTrees(10)

// Convert indexed labels back to original labels.
val labelConverter = new IndexToString()
  .setInputCol("prediction")
  .setOutputCol("predictedLabel")
  .setLabels(labelIndexer.labels)

// Chain indexers and forest in a Pipeline.
val pipeline = new Pipeline()
  .setStages(Array(labelIndexer, featureIndexer, rf, labelConverter))

// Train model. This also runs the indexers.
val model = pipeline.fit(trainingData)

// Make predictions.
val predictions = model.transform(testData)

// Select example rows to display.
predictions.select("predictedLabel", "label", "features").show(5)

// Select (prediction, true label) and compute test error.
val evaluator = new MulticlassClassificationEvaluator()
  .setLabelCol("indexedLabel")
  .setPredictionCol("prediction")
  .setMetricName("accuracy")
val accuracy = evaluator.evaluate(predictions)
println("Test Error = " + (1.0 - accuracy))

val rfModel = model.stages(2).asInstanceOf[RandomForestClassificationModel]
println("Learned classification forest model:\n" + rfModel.toDebugString)
```

#### 4、梯度提升树分类器

梯度提升树（Gradient-boosted tree，GBT）是一个流行的使用决策树团体的分类和回归算法。

##### 例

```scala
import org.apache.spark.ml.Pipeline
import org.apache.spark.ml.classification.{GBTClassificationModel, GBTClassifier}
import org.apache.spark.ml.evaluation.MulticlassClassificationEvaluator
import org.apache.spark.ml.feature.{IndexToString, StringIndexer, VectorIndexer}

// Load and parse the data file, converting it to a DataFrame.
val data = spark.read.format("libsvm").load("data/mllib/sample_libsvm_data.txt")

// Index labels, adding metadata to the label column.
// Fit on whole dataset to include all labels in index.
val labelIndexer = new StringIndexer()
  .setInputCol("label")
  .setOutputCol("indexedLabel")
  .fit(data)
// Automatically identify categorical features, and index them.
// Set maxCategories so features with > 4 distinct values are treated as continuous.
val featureIndexer = new VectorIndexer()
  .setInputCol("features")
  .setOutputCol("indexedFeatures")
  .setMaxCategories(4)
  .fit(data)

// Split the data into training and test sets (30% held out for testing).
val Array(trainingData, testData) = data.randomSplit(Array(0.7, 0.3))

// Train a GBT model.
val gbt = new GBTClassifier()
  .setLabelCol("indexedLabel")
  .setFeaturesCol("indexedFeatures")
  .setMaxIter(10)

// Convert indexed labels back to original labels.
val labelConverter = new IndexToString()
  .setInputCol("prediction")
  .setOutputCol("predictedLabel")
  .setLabels(labelIndexer.labels)

// Chain indexers and GBT in a Pipeline.
val pipeline = new Pipeline()
  .setStages(Array(labelIndexer, featureIndexer, gbt, labelConverter))

// Train model. This also runs the indexers.
val model = pipeline.fit(trainingData)

// Make predictions.
val predictions = model.transform(testData)

// Select example rows to display.
predictions.select("predictedLabel", "label", "features").show(5)

// Select (prediction, true label) and compute test error.
val evaluator = new MulticlassClassificationEvaluator()
  .setLabelCol("indexedLabel")
  .setPredictionCol("prediction")
  .setMetricName("accuracy")
val accuracy = evaluator.evaluate(predictions)
println("Test Error = " + (1.0 - accuracy))

val gbtModel = model.stages(2).asInstanceOf[GBTClassificationModel]
println("Learned classification GBT model:\n" + gbtModel.toDebugString)
```

#### 5、多层感知机分类器

多层感知机分类器（Multilayer perceptron classifier，MLPC）是一个基于前馈人工神经网络（feedforward artifical neural network）的分类器。MLPC由多层节点构成。网络中的每层都全部地连接到下一层。输入层中的节点代表输入数据。所有其它的节点通过输入与节点权重$$w$$和偏差$$b$$线性组合并应用激活函数（activation function），将输入映射到输出。可以将$$K+1$$层MLPC写为矩阵的形式如下：

$$\mathrm{y}(x) = \mathrm{f_K}(...\mathrm{f_2}(w_2^T\mathrm{f_1}(w_1^T x+b_1)+b_2)...+b_K)$$

中间层节点使用sigmoid（logistic）函数：$$\mathrm{f}(z_i) = \frac{1}{1 + e^{-z_i}}$$

输出层节点使用softmax函数：$$\mathrm{f}(z_i) = \frac{e^{z_i}}{\sum_{k=1}^N e^{z_k}}$$

输出层的节点数量$$N$$与类型的数量相对应。

MLPC通过反向传播（backpropagation）学习模型。使用逻辑损失函数进行优化，并使用L-BFGS作为优化例程。

```scala
import org.apache.spark.ml.classification.MultilayerPerceptronClassifier
import org.apache.spark.ml.evaluation.MulticlassClassificationEvaluator

// Load the data stored in LIBSVM format as a DataFrame.
val data = spark.read.format("libsvm")
  .load("data/mllib/sample_multiclass_classification_data.txt")

// Split the data into train and test
val splits = data.randomSplit(Array(0.6, 0.4), seed = 1234L)
val train = splits(0)
val test = splits(1)

// specify layers for the neural network:
// input layer of size 4 (features), two intermediate of size 5 and 4
// and output of size 3 (classes)
val layers = Array[Int](4, 5, 4, 3)

// create the trainer and set its parameters
val trainer = new MultilayerPerceptronClassifier()
  .setLayers(layers)
  .setBlockSize(128)
  .setSeed(1234L)
  .setMaxIter(100)

// train the model
val model = trainer.fit(train)

// compute accuracy on the test set
val result = model.transform(test)
val predictionAndLabels = result.select("prediction", "label")
val evaluator = new MulticlassClassificationEvaluator()
  .setMetricName("accuracy")

println("Test set accuracy = " + evaluator.evaluate(predictionAndLabels))
```

#### 6、线性支持向量机

一个支持向量机（Support vector machine，SVM）在高维或者无限维空间中构造一个超平面（hyperplane）或一组超平面，可以用于分类，回归，或者其它任务。直觉上说，与任何类别距离最近的训练数据点的距离（所谓的函数间隔，functional margin）最大的超平面，能获得好的区分；因为一般间隔越大分类器的一般化误差（generalization error）越小。Spark ML中的`LinearSVC`支持使用线性SVM进行二元分类。在内部，使用OWLQN优化器优化Hinge Loss。

##### 例

```scala
import org.apache.spark.ml.classification.LinearSVC

// Load training data
val training = spark.read.format("libsvm").load("data/mllib/sample_libsvm_data.txt")

val lsvc = new LinearSVC()
  .setMaxIter(10)
  .setRegParam(0.1)

// Fit the model
val lsvcModel = lsvc.fit(training)

// Print the coefficients and intercept for linear svc
println(s"Coefficients: ${lsvcModel.coefficients} Intercept: ${lsvcModel.intercept}")
```

#### 7、一比其余分类器（即，一比所有）

一比其余（OneVsRest）是，通过给定一个可以高效地执行二元分类基础的分类器，来进行多元分类的机器学习简化例子。也被称为“一比所有”（one-vs-all）

`OneVsRest`被实现为一个`Estimator`。对基础的分类器，它用`Classifier`实例并未`k`个类别中的每一个创建一个而元分类问题。类别`i`的分类器被训练为预测标签是否为`i`，将类别`i`与其它类别区分。

通过评估每个分类器来完成预测，最可信的（most confident）分类器的索引被当作标签输出。

##### 例

加载Iris数据集，将其解析为一个DataFrame并使用`OneVsRest`进行多元分类。

```scala
import org.apache.spark.ml.classification.{LogisticRegression, OneVsRest}
import org.apache.spark.ml.evaluation.MulticlassClassificationEvaluator

// load data file.
val inputData = spark.read.format("libsvm")
  .load("data/mllib/sample_multiclass_classification_data.txt")

// generate the train/test split.
val Array(train, test) = inputData.randomSplit(Array(0.8, 0.2))

// instantiate the base classifier
val classifier = new LogisticRegression()
  .setMaxIter(10)
  .setTol(1E-6)
  .setFitIntercept(true)

// instantiate the One Vs Rest Classifier.
val ovr = new OneVsRest().setClassifier(classifier)

// train the multiclass model.
val ovrModel = ovr.fit(train)

// score the model on test data.
val predictions = ovrModel.transform(test)

// obtain evaluator.
val evaluator = new MulticlassClassificationEvaluator()
  .setMetricName("accuracy")

// compute the classification error on test data.
val accuracy = evaluator.evaluate(predictions)
println(s"Test Error = ${1 - accuracy}")
```

#### 8、朴素贝叶斯

朴素贝叶斯（Naive Bayes）分类器，是基于特征之间具有强（朴素，navie）独立性假设应用贝叶斯定理的简单概率分类器家族。`spark.ml`实现目前吃吃多项式朴素贝叶斯和（multinomial naive Bayes）伯努利朴素贝叶斯（Bernoulli naive Bayes）。

##### 例

```scala
import org.apache.spark.ml.classification.NaiveBayes
import org.apache.spark.ml.evaluation.MulticlassClassificationEvaluator

// Load the data stored in LIBSVM format as a DataFrame.
val data = spark.read.format("libsvm").load("data/mllib/sample_libsvm_data.txt")

// Split the data into training and test sets (30% held out for testing)
val Array(trainingData, testData) = data.randomSplit(Array(0.7, 0.3), seed = 1234L)

// Train a NaiveBayes model.
val model = new NaiveBayes()
  .fit(trainingData)

// Select example rows to display.
val predictions = model.transform(testData)
predictions.show()

// Select (prediction, true label) and compute test error
val evaluator = new MulticlassClassificationEvaluator()
  .setLabelCol("label")
  .setPredictionCol("prediction")
  .setMetricName("accuracy")
val accuracy = evaluator.evaluate(predictions)
println("Test set accuracy = " + accuracy)
```

