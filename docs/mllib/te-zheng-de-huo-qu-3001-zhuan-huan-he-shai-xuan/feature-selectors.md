### 特征筛选（Feature Selectors）

#### 1、VectorSlicer

`VectorSlicer` 输入一个特征向量输出原特征的子数组作为新的特征向量。可以用于从一个向量列提取特征。

`VectorSlicer`接收一个向量列，通过指定的索引，输出索引对应的元素组成的新的向量列。有两类索引：

1. 表示向量中索引的整数索引，通过`setIndices()`设置。
2. 表示向量中特征名字的字符串索引，通过`setNames()`设置。这需要向量列具有`AttributeGroup`，因为这种实现匹配的是向量`Attribute`的名字。

指定整数索引和字符串索引都是可行的，此外，还可以同时指定整数索引和字符串名称索引。至少要选择一个特征。不允许重复选择特征，以便设置整数索引和名称索引不重复。如果使用字符串索引，而向量的`Attribute`名字是空的会抛出异常。

输出向量会首先按照指定索引的顺序排序，然后按照指定名称排序。

##### 例

输入数据集：

```
 userFeatures
------------------
 [0.0, 10.0, 0.5]
```

`userFeatures`是一个包含三个用户特征的的向量列。假如，`userFeatures`的向量的第一个元素都是0，想要移除第一个元素。使用`setIndices(1, 2)`通过`VectorSlicer`生成一个新的特征列`features`：

```
 userFeatures     | features
------------------|-----------------------------
 [0.0, 10.0, 0.5] | [10.0, 0.5]
```

假如`userFeatures`具有属性，比如`["f1", "f2", "f3"]`，那么也可以使用 `setNames("f2", "f3")`来选择向量元素：

```
 userFeatures     | features
------------------|-----------------------------
 [0.0, 10.0, 0.5] | [10.0, 0.5]
 ["f1", "f2", "f3"] | ["f2", "f3"]
```

代码：

```scala
import java.util.Arrays

import org.apache.spark.ml.attribute.{Attribute, AttributeGroup, NumericAttribute}
import org.apache.spark.ml.feature.VectorSlicer
import org.apache.spark.ml.linalg.Vectors
import org.apache.spark.sql.Row
import org.apache.spark.sql.types.StructType

val data = Arrays.asList(
  Row(Vectors.sparse(3, Seq((0, -2.0), (1, 2.3)))),
  Row(Vectors.dense(-2.0, 2.3, 0.0))
)

val defaultAttr = NumericAttribute.defaultAttr
val attrs = Array("f1", "f2", "f3").map(defaultAttr.withName)
val attrGroup = new AttributeGroup("userFeatures", attrs.asInstanceOf[Array[Attribute]])

val dataset = spark.createDataFrame(data, StructType(Array(attrGroup.toStructField())))

val slicer = new VectorSlicer().setInputCol("userFeatures").setOutputCol("features")

slicer.setIndices(Array(1)).setNames(Array("f3"))
// or slicer.setIndices(Array(1, 2)), or slicer.setNames(Array("f2", "f3"))

val output = slicer.transform(dataset)
output.show(false)
```

#### 2、RFomula

`RFormula`通过指定一个R模型公式来选择列。目前只支持有限的几个R操作符，包括 ‘~’， ‘.’， ‘:’， ‘+’，和‘-‘：

- ‘~’分隔目标（target）和项目（term）
- ‘+’连接项目，“+ 0”意味着移除截距
- ‘-‘移除一个项目，“- 1”意味着移除截距
- ‘:’互操作（interaction，对于数值或者二值化的类别变量是相乘）
- ‘.’除了目标的所有列

假如`a`和`b`是double列，使用如下例子来解释`RFormula`的效果：

- `y ~ a + b` 表示模型 `y ~ w0 + w1 * a + w2 * b` 其中 `w0` 是截距， `w1, w2`是系数。
- `y ~ a + b + a:b - 1` 表示模型 `y ~ w1 * a + w2 * b + w3 * a * b` 其中 `w1, w2, w3` 是系数。

`RFormula`产生一个特征的向量列和一个double或字符串类型的标签列。与R语言中用于线性回归的公式类似，字符串输入列会被进行one-hot编码，数值列会被转换为double。如果标签列是字符串类型，它首先会被使用`StringIndexer`转换为double。如果DataFrame中没有标签列，会根据公式中指定的响应变量创建输出标签列。

##### 例

输入数据集：

```
id | country | hour | clicked
---|---------|------|---------
 7 | "US"    | 18   | 1.0
 8 | "CA"    | 12   | 0.0
 9 | "NZ"    | 15   | 0.0
```

通过`RFormula`使用字符串公式`clicked ~ country + hour`，表示基于`country`和`hour`来预测`clicked`，转换后的结果：

```
id | country | hour | clicked | features         | label
---|---------|------|---------|------------------|-------
 7 | "US"    | 18   | 1.0     | [0.0, 0.0, 18.0] | 1.0
 8 | "CA"    | 12   | 0.0     | [0.0, 1.0, 12.0] | 0.0
 9 | "NZ"    | 15   | 0.0     | [1.0, 0.0, 15.0] | 0.0
```

代码：

```scala
import org.apache.spark.ml.feature.RFormula

val dataset = spark.createDataFrame(Seq(
  (7, "US", 18, 1.0),
  (8, "CA", 12, 0.0),
  (9, "NZ", 15, 0.0)
)).toDF("id", "country", "hour", "clicked")

val formula = new RFormula()
  .setFormula("clicked ~ country + hour")
  .setFeaturesCol("features")
  .setLabelCol("label")

val output = formula.fit(dataset).transform(dataset)
output.select("features", "label").show()
```

#### 3、ChiSqSelector

`ChiSqSelector`表示卡方（Chi-Squared）特征筛选。用于处理具有类别特征的有标签的数据（labled data）。`ChiSqSelector`使用独立卡方测试（Chi-Squared test of independence）来决定选择哪些特征。它支持5中筛选方法——`numTopFeatures`，`percentile`， `fpr`， `fdr`， `fwe`：

- `numTopFeatures` 根据卡方测试选择固定数量的前几个特征，这类似于选择最具预测力（most predictive power）的那些特征
- `percentile` 与`numTopFeatures` 类似，只是选择固定因子（而不是固定数量）的特征
- `fpr` 选择`p`值（p-values）低于阈值的所有特征，这能控制了筛选的误报率（false positive rate ，假阳率）。
- `fdr` 使用Benjamini-Hochberg procedure筛选错误发现率（false discovery rate）低于阈值的所有特征。
- `fwe`选择`p`值（p-values）低于阈值的所有特征，使用`1/numFeatures`来缩放阈值，这控制了筛选的族系错误率（family-wise error rate ）。

默认的的筛选方法是`numTopFeatures`，默认的固定筛选特征数量是50。使用`setSelectorType`选择筛选方法。

##### 例

输入数据集：

```
id | features              | clicked
---|-----------------------|---------
 7 | [0.0, 0.0, 18.0, 1.0] | 1.0
 8 | [0.0, 1.0, 12.0, 0.0] | 0.0
 9 | [1.0, 0.0, 15.0, 0.1] | 0.0
```

使用`ChiSqSelector` ，设置`numTopFeatures = 1`，根据标签`clicked`，筛选出最有用的特征`selectedFeatures`：

```
id | features              | clicked | selectedFeatures
---|-----------------------|---------|------------------
 7 | [0.0, 0.0, 18.0, 1.0] | 1.0     | [1.0]
 8 | [0.0, 1.0, 12.0, 0.0] | 0.0     | [0.0]
 9 | [1.0, 0.0, 15.0, 0.1] | 0.0     | [0.1]
```

代码：

```scala
import org.apache.spark.ml.feature.ChiSqSelector
import org.apache.spark.ml.linalg.Vectors

val data = Seq(
  (7, Vectors.dense(0.0, 0.0, 18.0, 1.0), 1.0),
  (8, Vectors.dense(0.0, 1.0, 12.0, 0.0), 0.0),
  (9, Vectors.dense(1.0, 0.0, 15.0, 0.1), 0.0)
)

val df = spark.createDataset(data).toDF("id", "features", "clicked")

val selector = new ChiSqSelector()
  .setNumTopFeatures(1)
  .setFeaturesCol("features")
  .setLabelCol("clicked")
  .setOutputCol("selectedFeatures")

val result = selector.fit(df).transform(df)

println(s"ChiSqSelector output with top ${selector.getNumTopFeatures} features selected")
result.show()
```

