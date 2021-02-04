#### 12、Interaction

`Interaction`是一个`Transformer`，输入向量或者double值列，生成一个包含每个输入列的一个值的组合乘积的向量列。比如，两个3维向量作为输入列，输出则是一个9维向量列。

##### 例

输入数据集：

```
  id1|vec1          |vec2          
  ---|--------------|--------------
  1  |[1.0,2.0,3.0] |[8.0,4.0,5.0] 
  2  |[4.0,3.0,8.0] |[7.0,9.0,8.0] 
  3  |[6.0,1.0,9.0] |[2.0,3.0,6.0] 
  4  |[10.0,8.0,6.0]|[9.0,4.0,5.0] 
  5  |[9.0,2.0,7.0] |[10.0,7.0,3.0]
  6  |[1.0,1.0,4.0] |[2.0,8.0,4.0]    
```

应用`Interaction`后结果为：

```
  id1|vec1          |vec2          |interactedCol                                         
  ---|--------------|--------------|------------------------------------------------------
  1  |[1.0,2.0,3.0] |[8.0,4.0,5.0] |[8.0,4.0,5.0,16.0,8.0,10.0,24.0,12.0,15.0]            
  2  |[4.0,3.0,8.0] |[7.0,9.0,8.0] |[56.0,72.0,64.0,42.0,54.0,48.0,112.0,144.0,128.0]     
  3  |[6.0,1.0,9.0] |[2.0,3.0,6.0] |[36.0,54.0,108.0,6.0,9.0,18.0,54.0,81.0,162.0]        
  4  |[10.0,8.0,6.0]|[9.0,4.0,5.0] |[360.0,160.0,200.0,288.0,128.0,160.0,216.0,96.0,120.0]
  5  |[9.0,2.0,7.0] |[10.0,7.0,3.0]|[450.0,315.0,135.0,100.0,70.0,30.0,350.0,245.0,105.0] 
  6  |[1.0,1.0,4.0] |[2.0,8.0,4.0] |[12.0,48.0,24.0,12.0,48.0,24.0,48.0,192.0,96.0]       
```

代码：

```scala
val df = spark.createDataFrame(Seq(
  (1, 1, 2, 3, 8, 4, 5),
  (2, 4, 3, 8, 7, 9, 8),
  (3, 6, 1, 9, 2, 3, 6),
  (4, 10, 8, 6, 9, 4, 5),
  (5, 9, 2, 7, 10, 7, 3),
  (6, 1, 1, 4, 2, 8, 4)
)).toDF("id1", "id2", "id3", "id4", "id5", "id6", "id7")

val assembler1 = new VectorAssembler().
  setInputCols(Array("id2", "id3", "id4")).
  setOutputCol("vec1")

val assembled1 = assembler1.transform(df)

val assembler2 = new VectorAssembler().
  setInputCols(Array("id5", "id6", "id7")).
  setOutputCol("vec2")

val assembled2 = assembler2.transform(assembled1).select("id1", "vec1", "vec2")

val interaction = new Interaction()
  .setInputCols(Array("id1", "vec1", "vec2"))
  .setOutputCol("interactedCol")

val interacted = interaction.transform(assembled2)

interacted.show(truncate = false)
```

#### 13、Normalizer

`Normalizer`是一个`Transformer`，转换`Vector`行数据集，通过标准化（normalize，归一化）每个`Vector`以具有单位范数（unit norm）。有一个参数`p`（默认2），指定用于标准化的p范数（`p-norm`）。标准化可以使输入数据标准化并且提升学习算法的性能。

##### 例

如下例子展示了加载libsvm格式数据集，然后标准化每行以具有单位`L1`范数和单位`Linf`范数。

```scala
import org.apache.spark.ml.feature.Normalizer
import org.apache.spark.ml.linalg.Vectors

val dataFrame = spark.createDataFrame(Seq(
  (0, Vectors.dense(1.0, 0.5, -1.0)),
  (1, Vectors.dense(2.0, 1.0, 1.0)),
  (2, Vectors.dense(4.0, 10.0, 2.0))
)).toDF("id", "features")

// Normalize each Vector using $L^1$ norm.
val normalizer = new Normalizer()
  .setInputCol("features")
  .setOutputCol("normFeatures")
  .setP(1.0)

val l1NormData = normalizer.transform(dataFrame)
println("Normalized using L^1 norm")
l1NormData.show()

// Normalize each Vector using $L^\infty$ norm.
val lInfNormData = normalizer.transform(dataFrame, normalizer.p -> Double.PositiveInfinity)
println("Normalized using L^inf norm")
lInfNormData.show()
```

#### 14、StandardScaler

`StandardScaler`转换`Vector`行数据集，标准化每个特征以具有单位标准偏差和/或0平均值（zero mean）。参数有：

- `withStd`：默认为true，缩放数据为单位标准偏差。
- `withMean`：默认false。缩放之前使用平均值（mean）对数据进行中心化（center the data），输出是密集的，如果输入是稀疏的就需要小心进行处理。

`StandardScaler`是一个`Estimator`，基于数据集`fit`产生一个`StandardScalerModel`；这相当于计算汇总统计数据（summary statistics）。这个模型可以转换数据集中的`Vector`列为具有单位标准偏差和/或0平均值的特征。

注意，如果某个特征的标准偏差是0，对于这个特征会返回默认的0值。

##### 例

加载libsvm格式的数据集，然后标准化每个特征以具有单位标准偏差。

```scala
import org.apache.spark.ml.feature.StandardScaler

val dataFrame = spark.read.format("libsvm").load("data/mllib/sample_libsvm_data.txt")

val scaler = new StandardScaler()
  .setInputCol("features")
  .setOutputCol("scaledFeatures")
  .setWithStd(true)
  .setWithMean(false)

// Compute summary statistics by fitting the StandardScaler.
val scalerModel = scaler.fit(dataFrame)

// Normalize each feature to have unit standard deviation.
val scaledData = scalerModel.transform(dataFrame)
scaledData.show()
```

#### 15、MinMaxScaler

`MinMaxScaler`对`Vector`行数据集进行转换，重新缩放某个特征到一个特定的区间（通常`[0,1]`）。参数为：

- `min`：默认`0.0`。转换的下限，所有特征共享。
- `max`：默认`1.0`。转换的上限，所有特征共享。

`MinMaxScaler`对一个数据集计算汇总统计并生成一个`MinMaxScalerModel`。这个模型可以将每个特征转换为一个区间。

注意，因为0值可能会被转换为非零值，即使输入是稀疏的输出也是`DenseVector`。

##### 例

加载libsvm格式的数据集，然后重新缩放每个特征倒区间`[0, 1]`。

```scala
import org.apache.spark.ml.feature.MinMaxScaler
import org.apache.spark.ml.linalg.Vectors

val dataFrame = spark.createDataFrame(Seq(
  (0, Vectors.dense(1.0, 0.1, -1.0)),
  (1, Vectors.dense(2.0, 1.1, 1.0)),
  (2, Vectors.dense(3.0, 10.1, 3.0))
)).toDF("id", "features")

val scaler = new MinMaxScaler()
  .setInputCol("features")
  .setOutputCol("scaledFeatures")

// Compute summary statistics and generate MinMaxScalerModel
val scalerModel = scaler.fit(dataFrame)

// rescale each feature to range [min, max].
val scaledData = scalerModel.transform(dataFrame)
println(s"Features scaled to range: [${scaler.getMin}, ${scaler.getMax}]")
scaledData.select("features", "scaledFeatures").show()
```

#### 16、MaxAbsScaler

`MaxAbsScaler`对`Vector`行数据集进行转换，通过除以每个特征的最大绝对值，重新缩放某个特征到区间`[-1 , 1]`。它不会对数据进行平移或中心化（shift/center），因此不会破坏任何稀疏性（sparsity）。

`MaxAbsScaler`对一个数据集计算汇总统计并生成一个`MaxAbsScalerModel`。这个模型可以将每个特征转换为到区间`[-1, 1]`。

##### 例

加载libsvm格式的数据集，然后重新缩放每个特征倒区间`[-1, 1]`。

```scala
import org.apache.spark.ml.feature.MaxAbsScaler
import org.apache.spark.ml.linalg.Vectors

val dataFrame = spark.createDataFrame(Seq(
  (0, Vectors.dense(1.0, 0.1, -8.0)),
  (1, Vectors.dense(2.0, 1.0, -4.0)),
  (2, Vectors.dense(4.0, 10.0, 8.0))
)).toDF("id", "features")

val scaler = new MaxAbsScaler()
  .setInputCol("features")
  .setOutputCol("scaledFeatures")

// Compute summary statistics and generate MaxAbsScalerModel
val scalerModel = scaler.fit(dataFrame)

// rescale each feature to range [-1, 1]
val scaledData = scalerModel.transform(dataFrame)
scaledData.select("features", "scaledFeatures").show()
```

#### 17、Bucketizer

`Bucketizer`将连续特征列转换为特征桶（feature buckets）列，其中桶是用户指定的。参数是：

- `splits`：有`n+1`个`splits`时，有`n`个桶。切片x，y定义的桶的区间是`[x, y)`，如果是最后一个桶，则包含y。切片应该是严格递增的。必须明确提供`-inf`和`inf`以覆盖所有的Double值；否则，不再切片范围内的值会被当作错误对待。两个`splits`例子：`Array(Double.NegativeInfinity, 0.0, 1.0, Double.PositiveInfinity)` 和`Array(0.0, 1.0, 2.0)`。

如果不清楚目标列数据的边界，那么应该使用`Double.NegativeInfinity` 和`Double.PositiveInfinity` 作为切片的边界以防止发生潜在的超出桶边界异常。

需要注意，提供的切片必须是严格递增的。

##### 例

将`Double`列分桶为索引式（index-wise）列

```scala
import org.apache.spark.ml.feature.Bucketizer

val splits = Array(Double.NegativeInfinity, -0.5, 0.0, 0.5, Double.PositiveInfinity)

val data = Array(-999.9, -0.5, -0.3, 0.0, 0.2, 999.9)
val dataFrame = spark.createDataFrame(data.map(Tuple1.apply)).toDF("features")

val bucketizer = new Bucketizer()
  .setInputCol("features")
  .setOutputCol("bucketedFeatures")
  .setSplits(splits)

// Transform original data into its bucket index.
val bucketedData = bucketizer.transform(dataFrame)

println(s"Bucketizer output with ${bucketizer.getSplits.length-1} buckets")
bucketedData.show()
```

转换结果：

```
|features|bucketedFeatures|
+--------+----------------+
|  -999.9|             0.0|
|    -0.5|             1.0|
|    -0.3|             1.0|
|     0.0|             2.0|
|     0.2|             2.0|
|   999.9|             3.0|
```

#### 18、ElementwiseProduct

ElementwiseProduct用提供的“weight”向量，按照element-wist乘法（按元素相乘）的方式，去乘每个输入向量。即，通过一个缩放乘数（scaler multiplier）对数据集的每列进行缩放。

##### 例

使用转换向量对向量的数据集进行转换：

```scala
import org.apache.spark.ml.feature.ElementwiseProduct
import org.apache.spark.ml.linalg.Vectors

// Create some vector data; also works for sparse vectors
val dataFrame = spark.createDataFrame(Seq(
  ("a", Vectors.dense(1.0, 2.0, 3.0)),
  ("b", Vectors.dense(4.0, 5.0, 6.0)))).toDF("id", "vector")

val transformingVector = Vectors.dense(0.0, 1.0, 2.0)
val transformer = new ElementwiseProduct()
  .setScalingVec(transformingVector)
  .setInputCol("vector")
  .setOutputCol("transformedVector")

// Batch transform the vectors to create new column:
transformer.transform(dataFrame).show()
```

输出结果：

```
| id|       vector|transformedVector|
+---+-------------+-----------------+
|  a|[1.0,2.0,3.0]|    [0.0,2.0,6.0]|
|  b|[4.0,5.0,6.0]|   [0.0,5.0,12.0]|
```

#### 19、SQLTransformer

`SQLTransformer` 用来实现通过SQL语句定义的转换。目前只支持像`"SELECT ... FROM __THIS__ ..."`这样的语法，其中 `"__THIS__"` 表示底层的输入数据集的表。select子句指定结果中要展示的字段，常量和表达式，并且可以是任意Spark SQL支持的select子句。还可以使用Spark SQL的内置函数和UDFs。例如：

- `SELECT a, a + b AS a_b FROM __THIS__`
- `SELECT a, SQRT(b) AS b_sqrt FROM __THIS__ where a > 5`
- `SELECT a, b, SUM(c) AS c_sum FROM __THIS__ GROUP BY a, b`

##### 例

输入数据集：

```
 id |  v1 |  v2
----|-----|-----
 0  | 1.0 | 3.0  
 2  | 2.0 | 5.0
```

通过`SQLTransformer`使用`"SELECT *, (v1 + v2) AS v3, (v1 * v2) AS v4 FROM __THIS__"`进行转换后的输出为：

```
 id |  v1 |  v2 |  v3 |  v4
----|-----|-----|-----|-----
 0  | 1.0 | 3.0 | 4.0 | 3.0
 2  | 2.0 | 5.0 | 7.0 |10.0
```

代码：

```scala
import org.apache.spark.ml.feature.SQLTransformer

val df = spark.createDataFrame(
  Seq((0, 1.0, 3.0), (2, 2.0, 5.0))).toDF("id", "v1", "v2")

val sqlTrans = new SQLTransformer().setStatement(
  "SELECT *, (v1 + v2) AS v3, (v1 * v2) AS v4 FROM __THIS__")

sqlTrans.transform(df).show()
```

#### 20、VectorAssembler

`VectorAssembler`是一个把一组指定的列连结为一个向量列的转换器。可以用来将原始特征和不同特征转换器生成的特征连结为一个特征向量，以用来训练像逻辑回归和决策树这样的ML模型。`VectorAssembler`的输入列类型可以是：所有的数值类型，布尔类型，向量类型。在每行中，输入列会按照指定的顺序被连结到一个向量。

##### 例

输入数据集：

```
 id | hour | mobile | userFeatures     | clicked
----|------|--------|------------------|---------
 0  | 18   | 1.0    | [0.0, 10.0, 0.5] | 1.0
```

使用`VectorAssembler`连结输入列`hour`，`moblie`，`userFeatures`，输出结果为：

```
 id | hour | mobile | userFeatures     | clicked | features
----|------|--------|------------------|---------|-----------------------------
 0  | 18   | 1.0    | [0.0, 10.0, 0.5] | 1.0     | [18.0, 1.0, 0.0, 10.0, 0.5]
```

代码：

```scala
import org.apache.spark.ml.feature.VectorAssembler
import org.apache.spark.ml.linalg.Vectors

val dataset = spark.createDataFrame(
  Seq((0, 18, 1.0, Vectors.dense(0.0, 10.0, 0.5), 1.0))
).toDF("id", "hour", "mobile", "userFeatures", "clicked")

val assembler = new VectorAssembler()
  .setInputCols(Array("hour", "mobile", "userFeatures"))
  .setOutputCol("features")

val output = assembler.transform(dataset)
println("Assembled columns 'hour', 'mobile', 'userFeatures' to vector column 'features'")
output.select("features", "clicked").show(false)
```

#### 21、QuantileDiscretizer

`QuantileDiscretizer` 将连续特征列转换为装箱的（binned）类别特征列。通过`numBuckets`参数指定箱子的数量。如果输入的不同值得数量太少，少于`numBuckets`，结果中的箱子数会也会少于`numBuckets`。

`NaN`值：在`QuantileDiscretizer`拟合期间，`NaN`值会被移除。拟合后会生成一个`Bucketizer`模型来进行预测。在转换期间，`Bucketizer`发现数据集中有`NaN`值时，会抛出错误，但是，用户还可以通过设置`handleInvalid`选择保留或者移除数据集中的`NaN`值。如果用户选择保留`NaN`值，`NaN`值会被特殊对待并放入特定的箱子，比如，如果使用4个箱子，非`NaN`的数据会被放入箱子`[0-3]`，`NaN`的数据则放入箱子`[4]`。

算法：使用近似算法（approxQuantile）来选择箱子区间。可以使用`relativeError`参数来控制近似算法的精度。设置为0时，计算精确地分位数（quantile），计算精确的分位数是高开销地操作。为了覆盖所有的实数值箱子的上限下限分别是 `-Infinity` 和 `+Infinity`。

##### 例

输入数据集，`hour`是一个连续的特征：

```
 id | hour
----|------
 0  | 18.0
----|------
 1  | 19.0
----|------
 2  | 8.0
----|------
 3  | 5.0
----|------
 4  | 2.2
```

设置`numBuckets`为3，进行转换的结果为：

```
 id | hour | result
----|------|------
 0  | 18.0 | 2.0
----|------|------
 1  | 19.0 | 2.0
----|------|------
 2  | 8.0  | 1.0
----|------|------
 3  | 5.0  | 1.0
----|------|------
 4  | 2.2  | 0.0
```

代码：

```scala
import org.apache.spark.ml.feature.QuantileDiscretizer

val data = Array((0, 18.0), (1, 19.0), (2, 8.0), (3, 5.0), (4, 2.2))
val df = spark.createDataFrame(data).toDF("id", "hour")

val discretizer = new QuantileDiscretizer()
  .setInputCol("hour")
  .setOutputCol("result")
  .setNumBuckets(3)

val result = discretizer.fit(df).transform(df)
result.show()
```

#### 22、Imputer

`Imputer`用来补全数据集中的缺失值，使用缺失值所在列的平均值或者中位值进行补全。输入列应该是`DoubleType`或者`FloatType`。目前`Imputer`不支持类别特征，对于包含类别的特征的列进行补全可能会产生错误的值。

注意，所有的`null`值都会被当作缺失值，都会被补全。

##### 例

输入数据集：

```
      a     |      b      
------------|-----------
     1.0    | Double.NaN
     2.0    | Double.NaN
 Double.NaN |     3.0   
     4.0    |     4.0   
     5.0    |     5.0   
```

`Imputer`会替换所有的`Double.NaN` ，使用平均值（默认的补全策略）。

```
      a     |      b     | out_a | out_b   
------------|------------|-------|-------
     1.0    | Double.NaN |  1.0  |  4.0 
     2.0    | Double.NaN |  2.0  |  4.0 
 Double.NaN |     3.0    |  3.0  |  3.0 
     4.0    |     4.0    |  4.0  |  4.0
     5.0    |     5.0    |  5.0  |  5.0 
```

代码：

```scala
import org.apache.spark.ml.feature.Imputer

val df = spark.createDataFrame(Seq(
  (1.0, Double.NaN),
  (2.0, Double.NaN),
  (Double.NaN, 3.0),
  (4.0, 4.0),
  (5.0, 5.0)
)).toDF("a", "b")

val imputer = new Imputer()
  .setInputCols(Array("a", "b"))
  .setOutputCols(Array("out_a", "out_b"))

val model = imputer.fit(df)
model.transform(df).show()
```

