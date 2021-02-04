### ML Pipelines

ML Pipelines（机器学习管道）提供了一组高级别的基于DataFrames的APIs，可以用来创建和调试实际的机器学习管道。

#### 1、管道中的主要概念

MLlib将用于机器学习算法标准化，以更简单地将多个算法在一个管道（或者工作流）中结合起来。MLlib的管道深受scikit-learn项目的启发。

- **DataFrame**：ML API使用Spark SQL的`DataFrame`作为ML数据集，`DataFrame`可以持有多种数据类型。比如，`DataFrame`可以包含不同的列——保存文本，特征向量，true标签，和预测值。
- **Transformer**：`Transformer`是将一个`DataFrame`转换为另一个`DataFrame`的一个算法。比如，一个ML模型是一个将带有特征的`DataFrame`转换为一个带有预测结果的`DataFrame`。
- **Estimator**：`Estimator`是基于`DataFrame`拟合出一个`Transformer`的一个算法。比如，一个`Estimator`是一个基于`DataFrame`训练出一个模型的学习算法。
- **Pipeline**：`Pipeline`链将`Transformer`s和`Estimator`s组合在一起构成一个ML工作流。
- **Parameter**：所有的`Transformer`s和`Estimator`s分享同一个API来指定参数。

#### 2、DataFrame

机器学习可以应用到多种数据类型，比如向量，文本，图像，和有结构的数据。`spark.ml`使用来自Spark SQL的`DataFrame`来支持多种数据类型。

`DataFrame`支持多种基本和结构化的数据类型；除了Spark SQL guide中的数据类型，`DataFrame`还可以使用ML的`Vector`类型。

可以隐式地或者明确地使用普通`RDD`创建`DataFrame`。

`DataFrame`中的列是有名称的。

#### 3、管道组件

##### 3.1、Transformers

`Transformer`是一个包含特征转换器和学习得出的模型的抽象概念。在技术上，`Transformer`实现了一个方法`transfrom`，将一个`DataFrame`转换为另一个`DataFrame`，一般是通过追加一列或者多列实现的。比如：

- 一个特征转换器可能用一个`DataFrame`，读取一列（比如，文本）将它映射为新的一列（比如，特征向量），并且输出追加了映射的新列的`DataFrame`。
- 一个学习模型可能用一个`DataFrame`，读取包含特征向量的列，预测每个特征向量对应的标签，并且输出带有预测标签列的新的`DataFrame`。

##### 3.2、Estimators

`Estimator`抽象了拟合或者训练数据的算法的概念。技术上，`Estimator`实现了一个方法`fit()`，接收一个`DataFrame`产生一个`Model`（是一个`Transformer`）。比如，学习算法`LogisticRegression`是一个`Estimator`，调用`fit()`方法训练一个`LogisticRegressionModel`（是一个`Model`，所以是一个`Transformer`）。

#### 3.3、管道组建的属性

`Transformer.transform()`s和`Estimator.fit()`s都是无状态的。未来，可能会通过其它的概念支持有状态的算法。

`Transformer`或者`Estimator`的每个实例都有一个唯一的ID，在指定参数时有用。

#### 4、管道

在机器学习中，运行一系列的算法对数据进行处理和学习是很常见的。比如，一个简单的文本文件处理工作流可能包括如下一个阶段：

- 将文件的文本切分为单词
- 将文件的单词转换为数值特征向量
- 使用特征向量和标签学习一个预测模型

MLlib将这样的一个工作流表示为一个`Pipeline`，`Pipeline`由以指定的顺序运行的一系列`PipelineStage`s（`Transformer`s和`Estimator`s）构成。

##### 4.1、工作原理

`Pipeline`包含一系列的阶段，每个阶段是`Transformer`或者`Estimator`。这些阶段按照顺序运行，输入`DataFrame`随着在管道中的流动在每个阶段被转换。对于`TransFormer`阶段，对输入`DataFrame`调用`transfrom()`方法。对于`Estimator`阶段，调用`fit()`方法产生一个`Transformer`（它是`PipelineModel`或者拟合出的`Pipeline`的一部分），并对`DataFrame`调用这个`Transformer`的`transform()`方法。

用简单的文本文件工作流进一步阐释。下图是`Pipeline`的训练时（**traning time**）用法。

![ML Pipeline Example](/assets/ml-Pipeline.png)

上图中，顶行表示`Pipeline`的三个阶段。前两个阶段（`Tokenizer`和`HashingTF`）是`Transformer`s（蓝色），第三个（`LogisticRegression`）是一个`Estimator`（红色）。底行表示管道中数据的流动，圆柱表示`DataFrame`s。对源`DataFrame`（包含原始的文本文件和标签）调用`Pipeline.fit()`方法。`Tokenizer.transform()`方法把原始的文本文件切分为单词，将带有切分后单词的新列添加到`DataFrame`。`HashingTF.transform()`方法将单词列转换为特征向量，并往`DataFrame`添加新的特征向量列。因为`LogisticRegression`是一个`Estimaotr`，`Pipeline`首先调用`LogisticRegression.fit()`来产生一个`LogisticRegressionModel`。如果`Pipeline`有更多的`Estimator`s，它会在把`DataFrame`传递到下一个阶段之前，对`DataFrame`调用`LogisticRegressionModel`的`transform()`方法。

`Pipeline`是一个`Estimator`。因此，调用`Pipeline`的`fit()`方法后，它会产生一个`PipelineModel`（是一个`Transformer`）。这个`PipelineModel`是测试时（**test time**）使用的，下图是它的用法。

![ML PipelineModel Example](/assets/ml-PipelineModel.png)

在上图中，`PipelineModel`与原始的`Pipeline`具有相同数量的阶段，但是原始`Pipeline`中所有的`Estimator`s都变成了`Transformer`s。当对测试数据集调用`PipelineModel`的`transform()`方法时，数据按照顺序在拟合出的管道中传递。每个阶段的`transform()`方法对数据集进行更新并将其传递到下一个阶段。

`Pipeline`s和`PipelineModel`s帮助保证了训练数据和测试数据经过的是相同的特征处理步骤。

##### 4.2、详细

**DAG `Pipeline`s**：`Pipeline`的阶段被指定为一个有序的数组。这里的例子都是线性的`Pipeline`s，即，`Pipeline`中每个阶段都是用前一阶段产生的数据。只要数据流图能够撑一个有向无环图（DAG）也可以创建非线性的`Pipeline`s。目前，数据流图是基于每个阶段输入输出列的名字（一般作为参数指定）隐式地指定的。如果`Pipeline`构成了一个DAG，那么必须按照拓扑结构顺序指定每个阶段。

**运行时检查**（Runtime checking）：因为`Pipeline`s可以操作不同类型的`DataFrame`s，所以`Pipeline`s不能使用编译时类型检查。`Pipeline`s和`PipelineModel`s在运行`Pipeline`之前执行的是运行时检查。这种类型检查是通过使用`DataFrame`的*schema*（`DataFrame`中列的数据类型描述）进行的。

**Unique Pipeline stages**：`Pipeline`的每个阶段都应该是唯一实例的。比如，相同的实例`myHashingTF`不能插入`Pipeline`两次，因为`Pipeline`的阶段必须具有唯一的IDs。但是不同的实例`myHashingTF1`和`myHashingTF2`（都是`HashingTF`类型）可以放进相同的`Pipeline`，因为不同的实例会产生不同的IDs。

#### 5、参数

MLlib`Estimator`s和`Transformer`s使用统一的API来指定参数。

`Param`是一个具有独立文档的命名参数。`ParamMap`是一个(参数，值)对的集合。

有两种给算法传递参数的主要方式：

1. 为一个实例设置参数。比如，如果`lr`是一个`LogisticRegression`实例，可以调用`lr.setMaxIter(10)`使`lr.fit()`使用最多10次遍历。
2. 传递`ParamMap`到`fit()`或`transform()`。`ParamMap`中的任何参数都会覆盖之前通过setter方法设置的参数。

参数是属于特定的`Estimator`s和`Transformer`s的。比如，如果有两个`LogisticalRegression`实例`lr1`和`lr2`，那么可以构建一个`ParamMap(lr1.maxIter -> 10, lr2.maxIter -> 20)`来为二者设置`maxIter`参数。如果`Pipeline`中有两个具有`maxIter`参数的算法时，这种方式是有用的。

#### 6、保存和加载管道

参考下面的[例子：Pipeline]()

#### 7、代码示例

##### 例子：Estimator，Transformer，和Param

```scala
import org.apache.spark.ml.classification.LogisticRegression
import org.apache.spark.ml.linalg.{Vector, Vectors}
import org.apache.spark.ml.param.ParamMap
import org.apache.spark.sql.Row

// Prepare training data from a list of (label, features) tuples.
val training = spark.createDataFrame(Seq(
  (1.0, Vectors.dense(0.0, 1.1, 0.1)),
  (0.0, Vectors.dense(2.0, 1.0, -1.0)),
  (0.0, Vectors.dense(2.0, 1.3, 1.0)),
  (1.0, Vectors.dense(0.0, 1.2, -0.5))
)).toDF("label", "features")

// Create a LogisticRegression instance. This instance is an Estimator.
val lr = new LogisticRegression()
// Print out the parameters, documentation, and any default values.
println("LogisticRegression parameters:\n" + lr.explainParams() + "\n")

// We may set parameters using setter methods.
lr.setMaxIter(10)
  .setRegParam(0.01)

// Learn a LogisticRegression model. This uses the parameters stored in lr.
val model1 = lr.fit(training)
// Since model1 is a Model (i.e., a Transformer produced by an Estimator),
// we can view the parameters it used during fit().
// This prints the parameter (name: value) pairs, where names are unique IDs for this
// LogisticRegression instance.
println("Model 1 was fit using parameters: " + model1.parent.extractParamMap)

// We may alternatively specify parameters using a ParamMap,
// which supports several methods for specifying parameters.
val paramMap = ParamMap(lr.maxIter -> 20)
  .put(lr.maxIter, 30)  // Specify 1 Param. This overwrites the original maxIter.
  .put(lr.regParam -> 0.1, lr.threshold -> 0.55)  // Specify multiple Params.

// One can also combine ParamMaps.
val paramMap2 = ParamMap(lr.probabilityCol -> "myProbability")  // Change output column name.
val paramMapCombined = paramMap ++ paramMap2

// Now learn a new model using the paramMapCombined parameters.
// paramMapCombined overrides all parameters set earlier via lr.set* methods.
val model2 = lr.fit(training, paramMapCombined)
println("Model 2 was fit using parameters: " + model2.parent.extractParamMap)

// Prepare test data.
val test = spark.createDataFrame(Seq(
  (1.0, Vectors.dense(-1.0, 1.5, 1.3)),
  (0.0, Vectors.dense(3.0, 2.0, -0.1)),
  (1.0, Vectors.dense(0.0, 2.2, -1.5))
)).toDF("label", "features")

// Make predictions on test data using the Transformer.transform() method.
// LogisticRegression.transform will only use the 'features' column.
// Note that model2.transform() outputs a 'myProbability' column instead of the usual
// 'probability' column since we renamed the lr.probabilityCol parameter previously.
model2.transform(test)
  .select("features", "label", "myProbability", "prediction")
  .collect()
  .foreach { case Row(features: Vector, label: Double, prob: Vector, prediction: Double) =>
    println(s"($features, $label) -> prob=$prob, prediction=$prediction")
  }
```

##### 例子：Pipeline

```scala
import org.apache.spark.ml.{Pipeline, PipelineModel}
import org.apache.spark.ml.classification.LogisticRegression
import org.apache.spark.ml.feature.{HashingTF, Tokenizer}
import org.apache.spark.ml.linalg.Vector
import org.apache.spark.sql.Row

// Prepare training documents from a list of (id, text, label) tuples.
val training = spark.createDataFrame(Seq(
  (0L, "a b c d e spark", 1.0),
  (1L, "b d", 0.0),
  (2L, "spark f g h", 1.0),
  (3L, "hadoop mapreduce", 0.0)
)).toDF("id", "text", "label")

// Configure an ML pipeline, which consists of three stages: tokenizer, hashingTF, and lr.
val tokenizer = new Tokenizer()
  .setInputCol("text")
  .setOutputCol("words")
val hashingTF = new HashingTF()
  .setNumFeatures(1000)
  .setInputCol(tokenizer.getOutputCol)
  .setOutputCol("features")
val lr = new LogisticRegression()
  .setMaxIter(10)
  .setRegParam(0.001)
val pipeline = new Pipeline()
  .setStages(Array(tokenizer, hashingTF, lr))

// Fit the pipeline to training documents.
val model = pipeline.fit(training)

// Now we can optionally save the fitted pipeline to disk
model.write.overwrite().save("/tmp/spark-logistic-regression-model")

// We can also save this unfit pipeline to disk
pipeline.write.overwrite().save("/tmp/unfit-lr-model")

// And load it back in during production
val sameModel = PipelineModel.load("/tmp/spark-logistic-regression-model")

// Prepare test documents, which are unlabeled (id, text) tuples.
val test = spark.createDataFrame(Seq(
  (4L, "spark i j k"),
  (5L, "l m n"),
  (6L, "spark hadoop spark"),
  (7L, "apache hadoop")
)).toDF("id", "text")

// Make predictions on test documents.
model.transform(test)
  .select("id", "text", "probability", "prediction")
  .collect()
  .foreach { case Row(id: Long, text: String, prob: Vector, prediction: Double) =>
    println(s"($id, $text) --> prob=$prob, prediction=$prediction")
  }
```

##### 模型选择（超参调试）

使用ML管道的一个益处是超参（hyperparameter）优化。