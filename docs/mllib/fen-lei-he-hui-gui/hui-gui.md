### 回归（Regression）

#### 1、线性回归

使用线性回归（linear regression）模型和模型摘要的接口与逻辑回归类似。

> 当通过“l-bfgs” solver对具有常量非零列的数据集拟合没有截距的LinearRegressionModel时，Spark MLlib会为常量非零列输出0系数。这种行为与Rglmnet相同，当时与LIBSVM不同。

##### 例

训练一个elastic net 正则化的线性回归模型，并获取模型摘要统计数据。

```scala
import org.apache.spark.ml.regression.LinearRegression

// Load training data
val training = spark.read.format("libsvm")
  .load("data/mllib/sample_linear_regression_data.txt")

val lr = new LinearRegression()
  .setMaxIter(10)
  .setRegParam(0.3)
  .setElasticNetParam(0.8)

// Fit the model
val lrModel = lr.fit(training)

// Print the coefficients and intercept for linear regression
println(s"Coefficients: ${lrModel.coefficients} Intercept: ${lrModel.intercept}")

// Summarize the model over the training set and print out some metrics
val trainingSummary = lrModel.summary
println(s"numIterations: ${trainingSummary.totalIterations}")
println(s"objectiveHistory: [${trainingSummary.objectiveHistory.mkString(",")}]")
trainingSummary.residuals.show()
println(s"RMSE: ${trainingSummary.rootMeanSquaredError}")
println(s"r2: ${trainingSummary.r2}")
```

#### 2、广义线性回归

与输出被假设为符合高斯分布（Gaussian distribution）的线性回归相反，广义线性模型（Generalized linear models，GLMs）是响应变量$$Y_i$$符合某种指数分布的线性模型。Spark的`GeneralizedLinearRegression` 接口可以使用多种GLMs，可以用于多种类型的预测问题，包括线性回归，Possion回归，逻辑回归，和其它。目前，在`spark.ml`中只支持某些指数分布。

注意，Spark目前通过`GeneralizedLinearRegression`接口支持最多4096个特征，如果超过这个限制会抛出异常。对于线性回归和逻辑回归，使用`LinearRegression` 和`LogisticRegression` estimators 仍然可以训练具有更多特征的模型。

GLMs需要那些可以写为“典型”（canonical）或者“自然”（natural）形式的指数分布，即自然指数分布。自然指数分布的形式为：

$$f_Y(y|\theta, \tau) = h(y, \tau)\exp{\left( \frac{\theta \cdot y - A(\theta)}{d(\tau)} \right)}$$

其中$$\theta$$是interest参数$$\tau$$是dispersion参数。在GLM中响应变量$$Y_i$$服从自然指数分布：$$Y_i \sim f\left(\cdot|\theta_i, \tau \right)$$

Spark的广义线性回归接口也提供了用于诊断GLM模型拟合的摘要统计，包括residuals，p-values，deviances，和Akaike information criterion，等。

##### 2.1、可用的分布

| Family | Response Type | Supported Links |
| --- | --- | --- |
| Gaussian | Continuous | Identity\*, Log, Inverse |
| Binomial | Binary | Logit\*, Probit, CLogLog |
| Poisson | Count | Log\*, Identity, Sqrt |
| Gamma | Continuous | Inverse\*, Idenity, Log |
| Tweedie | Zero-inflated continuous | Power link function |

\* 表示Canonical Link

##### 例

使用Gaussian响应和identity link函数训练一个GLM，并提取模型摘要统计数据。

```scala
import org.apache.spark.ml.regression.GeneralizedLinearRegression

// Load training data
val dataset = spark.read.format("libsvm")
  .load("data/mllib/sample_linear_regression_data.txt")

val glr = new GeneralizedLinearRegression()
  .setFamily("gaussian")
  .setLink("identity")
  .setMaxIter(10)
  .setRegParam(0.3)

// Fit the model
val model = glr.fit(dataset)

// Print the coefficients and intercept for generalized linear regression model
println(s"Coefficients: ${model.coefficients}")
println(s"Intercept: ${model.intercept}")

// Summarize the model over the training set and print out some metrics
val summary = model.summary
println(s"Coefficient Standard Errors: ${summary.coefficientStandardErrors.mkString(",")}")
println(s"T Values: ${summary.tValues.mkString(",")}")
println(s"P Values: ${summary.pValues.mkString(",")}")
println(s"Dispersion: ${summary.dispersion}")
println(s"Null Deviance: ${summary.nullDeviance}")
println(s"Residual Degree Of Freedom Null: ${summary.residualDegreeOfFreedomNull}")
println(s"Deviance: ${summary.deviance}")
println(s"Residual Degree Of Freedom: ${summary.residualDegreeOfFreedom}")
println(s"AIC: ${summary.aic}")
println("Deviance Residuals: ")
summary.residuals().show()
```

#### 3、决策树回归

##### 例

```scala
import org.apache.spark.ml.Pipeline
import org.apache.spark.ml.evaluation.RegressionEvaluator
import org.apache.spark.ml.feature.VectorIndexer
import org.apache.spark.ml.regression.DecisionTreeRegressionModel
import org.apache.spark.ml.regression.DecisionTreeRegressor

// Load the data stored in LIBSVM format as a DataFrame.
val data = spark.read.format("libsvm").load("data/mllib/sample_libsvm_data.txt")

// Automatically identify categorical features, and index them.
// Here, we treat features with > 4 distinct values as continuous.
val featureIndexer = new VectorIndexer()
  .setInputCol("features")
  .setOutputCol("indexedFeatures")
  .setMaxCategories(4)
  .fit(data)

// Split the data into training and test sets (30% held out for testing).
val Array(trainingData, testData) = data.randomSplit(Array(0.7, 0.3))

// Train a DecisionTree model.
val dt = new DecisionTreeRegressor()
  .setLabelCol("label")
  .setFeaturesCol("indexedFeatures")

// Chain indexer and tree in a Pipeline.
val pipeline = new Pipeline()
  .setStages(Array(featureIndexer, dt))

// Train model. This also runs the indexer.
val model = pipeline.fit(trainingData)

// Make predictions.
val predictions = model.transform(testData)

// Select example rows to display.
predictions.select("prediction", "label", "features").show(5)

// Select (prediction, true label) and compute test error.
val evaluator = new RegressionEvaluator()
  .setLabelCol("label")
  .setPredictionCol("prediction")
  .setMetricName("rmse")
val rmse = evaluator.evaluate(predictions)
println("Root Mean Squared Error (RMSE) on test data = " + rmse)

val treeModel = model.stages(1).asInstanceOf[DecisionTreeRegressionModel]
println("Learned regression tree model:\n" + treeModel.toDebugString)
```

#### 4、随机森林回归

##### 例

```scala
import org.apache.spark.ml.Pipeline
import org.apache.spark.ml.evaluation.RegressionEvaluator
import org.apache.spark.ml.feature.VectorIndexer
import org.apache.spark.ml.regression.{RandomForestRegressionModel, RandomForestRegressor}

// Load and parse the data file, converting it to a DataFrame.
val data = spark.read.format("libsvm").load("data/mllib/sample_libsvm_data.txt")

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
val rf = new RandomForestRegressor()
  .setLabelCol("label")
  .setFeaturesCol("indexedFeatures")

// Chain indexer and forest in a Pipeline.
val pipeline = new Pipeline()
  .setStages(Array(featureIndexer, rf))

// Train model. This also runs the indexer.
val model = pipeline.fit(trainingData)

// Make predictions.
val predictions = model.transform(testData)

// Select example rows to display.
predictions.select("prediction", "label", "features").show(5)

// Select (prediction, true label) and compute test error.
val evaluator = new RegressionEvaluator()
  .setLabelCol("label")
  .setPredictionCol("prediction")
  .setMetricName("rmse")
val rmse = evaluator.evaluate(predictions)
println("Root Mean Squared Error (RMSE) on test data = " + rmse)

val rfModel = model.stages(1).asInstanceOf[RandomForestRegressionModel]
println("Learned regression forest model:\n" + rfModel.toDebugString)
```

#### 5、梯度提升树（GBT，Gradient-boosted tree）回归

##### 例

注意：本例中`GBTRegressor`实际上只进行了一次迭代，实践中，通常会需要更多次数迭代。

```scala
import org.apache.spark.ml.Pipeline
import org.apache.spark.ml.evaluation.RegressionEvaluator
import org.apache.spark.ml.feature.VectorIndexer
import org.apache.spark.ml.regression.{GBTRegressionModel, GBTRegressor}

// Load and parse the data file, converting it to a DataFrame.
val data = spark.read.format("libsvm").load("data/mllib/sample_libsvm_data.txt")

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
val gbt = new GBTRegressor()
  .setLabelCol("label")
  .setFeaturesCol("indexedFeatures")
  .setMaxIter(10)

// Chain indexer and GBT in a Pipeline.
val pipeline = new Pipeline()
  .setStages(Array(featureIndexer, gbt))

// Train model. This also runs the indexer.
val model = pipeline.fit(trainingData)

// Make predictions.
val predictions = model.transform(testData)

// Select example rows to display.
predictions.select("prediction", "label", "features").show(5)

// Select (prediction, true label) and compute test error.
val evaluator = new RegressionEvaluator()
  .setLabelCol("label")
  .setPredictionCol("prediction")
  .setMetricName("rmse")
val rmse = evaluator.evaluate(predictions)
println("Root Mean Squared Error (RMSE) on test data = " + rmse)

val gbtModel = model.stages(1).asInstanceOf[GBTRegressionModel]
println("Learned regression GBT model:\n" + gbtModel.toDebugString)
```

#### 6、生存回归（Survival regression）

在`spark.ml`中，实现了加速故障（Accelerated failure time， AFT）模型，这是一个用于修正过数据（censored data）的参数的生存回归（parametric survival regression）模型。它描述了一个生存时间的日志的模型，所以通常它被称为生存分析的日志线性模型（log-linear model for survival analysis）。与用于相同目的的比例危害（Proportinal hazards）模型不同，AFT模型更易于并行化，因为它的每个实例都独立地为目标函数做贡献。

###### 公式，算法，省略。

##### 例

```scala
import org.apache.spark.ml.linalg.Vectors
import org.apache.spark.ml.regression.AFTSurvivalRegression

val training = spark.createDataFrame(Seq(
  (1.218, 1.0, Vectors.dense(1.560, -0.605)),
  (2.949, 0.0, Vectors.dense(0.346, 2.158)),
  (3.627, 0.0, Vectors.dense(1.380, 0.231)),
  (0.273, 1.0, Vectors.dense(0.520, 1.151)),
  (4.199, 0.0, Vectors.dense(0.795, -0.226))
)).toDF("label", "censor", "features")
val quantileProbabilities = Array(0.3, 0.6)
val aft = new AFTSurvivalRegression()
  .setQuantileProbabilities(quantileProbabilities)
  .setQuantilesCol("quantiles")

val model = aft.fit(training)

// Print the coefficients, intercept and scale parameter for AFT survival regression
println(s"Coefficients: ${model.coefficients}")
println(s"Intercept: ${model.intercept}")
println(s"Scale: ${model.scale}")
model.transform(training).show(false)
```

#### 7、保序回归（Isotonic regression）

在形式上，保序回归的定义是：给定了一个代表观测响应的有限的实数集合$$Y = {y_1, y_2,...,y_n}$$，$$X= {x_1, x_2,...,x_n}$$表示未知的响应值，拟合出一个函数使

$$f(x) = \sum_{i=1}^n w_i (y_i - x_i)^2$$

最小，其中，整体顺序服从$$x_1\le x_2\le ...\le x_n$$，$$w_i$$是正的权重。结果函数被称为保序回归，并且它是唯一的。它可以被看作是顺序限制下的最小平方问题。本质上，保序回归是最符合原始数据点的一个单调函数（monotonic function）。

`spark.ml`实现了一个pool adjacent violators algorithm（PAVA）来并行化保序回归。训练输入是一个包含标签（label）、特征（features）和权重（weight）三列的DataFrame。此外，`IsotonicRegression`算法有一个名为_**isotonic**_的默认值为**true**的可选参数，这个参数决定保序回归是保序的（单调递增）还是antitonic（单调递减）。

训练返回一个`IsotonicRegressionModel`，可以用来预测已知和未知特征的标签。保序回归的结果被作为分段线性函数（piecewise linear function）对待。预测的规则是：

- 如果预测输入精确地匹配一个训练特征，那么返回对应的预测结果。如果相同特征对应多个预测结果，则返回其中的一个，至于返回哪一个是不确定的（与`java.util.Arrays.binarySearch`相同）。
- 如果预测输入比所有的训练特征低或高，则分别返回最低或最高特征对应的预测结果。如果相同特征对应多个预测结果，则分别返回最低或最高的预测结果。
- 如果预测输入落在两个训练特征之间，预测会被作为分段线性函数对待，并通过两个最近的特征的预测结果来计算插入的值的预测结果。如果相同特征对应多个预测结果，则使用与上一条相同的规则。

##### 例

```scala
import org.apache.spark.ml.regression.IsotonicRegression

// Loads data.
val dataset = spark.read.format("libsvm")
  .load("data/mllib/sample_isotonic_regression_libsvm_data.txt")

// Trains an isotonic regression model.
val ir = new IsotonicRegression()
val model = ir.fit(dataset)

println(s"Boundaries in increasing order: ${model.boundaries}\n")
println(s"Predictions associated with the boundaries: ${model.predictions}\n")

// Makes predictions.
model.transform(dataset).show()
```