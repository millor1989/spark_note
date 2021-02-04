### 特征转换算法（Feature Transformers）

#### 1、Tokenizer

**Tokenization（标记化）**是指将文本（比如句子）切分为单独的词语（term，通常是单词），`Tokenizer`提供了这种功能。

`RegexTokenizer`基于正则表达式匹配实现更高级的标记化。默认情况下，参数“pattern”（正则表达式，默认：“\\\s+”）被用作分隔输入文本的定界符。也可以设置“gaps”参数为false，此时“pattern”正则表达式表示的就是“标记”而不是分隔点，与“pattern”匹配的字符就是标记化的结果。

##### 例

```scala
import org.apache.spark.ml.feature.{RegexTokenizer, Tokenizer}
import org.apache.spark.sql.functions._

val sentenceDataFrame = spark.createDataFrame(Seq(
  (0, "Hi I heard about Spark"),
  (1, "I wish Java could use case classes"),
  (2, "Logistic,regression,models,are,neat")
)).toDF("id", "sentence")

val tokenizer = new Tokenizer().setInputCol("sentence").setOutputCol("words")
val regexTokenizer = new RegexTokenizer()
  .setInputCol("sentence")
  .setOutputCol("words")
  .setPattern("\\W") // alternatively .setPattern("\\w+").setGaps(false)

val countTokens = udf { (words: Seq[String]) => words.length }

val tokenized = tokenizer.transform(sentenceDataFrame)
tokenized.select("sentence", "words")
    .withColumn("tokens", countTokens(col("words"))).show(false)

val regexTokenized = regexTokenizer.transform(sentenceDataFrame)
regexTokenized.select("sentence", "words")
    .withColumn("tokens", countTokens(col("words"))).show(false)
```

#### 2、StopWordsRemover

**Stop words**（停用词）是指应该从输入中排除的单词，一般是指频繁出现但是没有实际意义的词。

`StopWordsRemover`输入为一个字符串序列（比如，`Tokenizer`的输出），输出是从输入字符串序列去除所有停用词的字符串序列。通过`stropWords`参数指定停用词集合。通过调用`StopWordsRemover.loadDefaultStopWords(language)`，可以查看某些语言的停用词，支持的语言参数`language`是： “danish”，“dutch”，“english”，“finnish”，“french”，“german”，“hungarian”，
“italian”， “norwegian”， “portuguese”， “russian”， “spanish”， “swedish” 和 “turkish”。布尔参数`caseSensitive`表示停用词匹配是否是大小写敏感的（默认false）。

##### 例

假如有一个列为`id`和`raw`的DataFrame：

```
 id | raw
----|----------
 0  | [I, saw, the, red, baloon]
 1  | [Mary, had, a, little, lamb]
```

把`raw`列作为`StopWordsRemover`的输入，`filtered`作为输出列，得到的结果是：

```
 id | raw                         | filtered
----|-----------------------------|--------------------
 0  | [I, saw, the, red, baloon]  |  [saw, red, baloon]
 1  | [Mary, had, a, little, lamb]|[Mary, little, lamb]
```

可见，“I”, “the”, “had”, 和 “a”被作为停止词过滤掉了。

```scala
import org.apache.spark.ml.feature.StopWordsRemover

val remover = new StopWordsRemover()
  .setInputCol("raw")
  .setOutputCol("filtered")

val dataSet = spark.createDataFrame(Seq(
  (0, Seq("I", "saw", "the", "red", "balloon")),
  (1, Seq("Mary", "had", "a", "little", "lamb"))
)).toDF("id", "raw")

remover.transform(dataSet).show(false)
```

#### 3、$$n$$-gram

一个`n-gram`是一个`n`个标记（一般是单词）的序列，`n`为整数。`NGram`类可把输入特征转换为`n-gram`特征。

`NGram`输入为字符串序列（比如，`Tokenizer`的输出）。参数`n`用来决定每个`n-gram`中词的个数；输出有`n-gram`序列构成；其中`n-gram`由`n`个连续的单词表示。如果输入序列包含少于`n`个字符串，则不会有输出。

##### 例

```scala
import org.apache.spark.ml.feature.NGram

val wordDataFrame = spark.createDataFrame(Seq(
  (0, Array("Hi", "I", "heard", "about", "Spark")),
  (1, Array("I", "wish", "Java", "could", "use", "case", "classes")),
  (2, Array("Logistic", "regression", "models", "are", "neat"))
)).toDF("id", "words")

val ngram = new NGram().setN(2).setInputCol("words").setOutputCol("ngrams")

val ngramDataFrame = ngram.transform(wordDataFrame)
ngramDataFrame.select("ngrams").show(false)
```

输出为：

```
|ngrams                                                                          |
+--------------------------------------------------------------------------------+
|[Hi I heard, I heard about, heard about Spark]                                  |
|[I wish Java, wish Java could, Java could use, could use case, use case classes]|
|[Logistic regression models, regression models are, models are neat]            |
```

#### 4、Binarizer

Binarization（二值化）指将数值特征阈值处理为二元（0/1）特征。

`Binarizer`的参数`threshold`，比它大的特征值被二值化为1.0；小于等于它的特征值被二值化为0.0。Vector类型和Double类型的特征都可以作为输入。

##### 例

```scala
import org.apache.spark.ml.feature.Binarizer

val data = Array((0, 0.1), (1, 0.8), (2, 0.2))
val dataFrame = spark.createDataFrame(data).toDF("id", "feature")

val binarizer: Binarizer = new Binarizer()
  .setInputCol("feature")
  .setOutputCol("binarized_feature")
  .setThreshold(0.5)

val binarizedDataFrame = binarizer.transform(dataFrame)

println(s"Binarizer output with Threshold = ${binarizer.getThreshold}")
binarizedDataFrame.show()
```

#### 5、PCA

PCA（Principal Component Analysis，主成分分析），使用正交变换（orthogonal transformation）将一组可能相关的观测值转换为一组线性不相关的值（称为，主成分）。`PCA`使用PCA训练一个模型把向量投射到低维度的空间

##### 例

把5维特征向量投射为3维的主成分：

```scala
import org.apache.spark.ml.feature.PCA
import org.apache.spark.ml.linalg.Vectors

val data = Array(
  Vectors.sparse(5, Seq((1, 1.0), (3, 7.0))),
  Vectors.dense(2.0, 0.0, 3.0, 4.0, 5.0),
  Vectors.dense(4.0, 0.0, 0.0, 6.0, 7.0)
)
val df = spark.createDataFrame(data.map(Tuple1.apply)).toDF("features")

val pca = new PCA()
  .setInputCol("features")
  .setOutputCol("pcaFeatures")
  .setK(3)
  .fit(df)

val result = pca.transform(df).select("pcaFeatures")
result.show(false)
```

#### 6、PolynomialExpansion

Polynomial expansion（多项式展开），是指把特征展开到多项式空间，由原始维度的n次组合来表示。

以特征向量(x, y)为例，如果进行2次多项式展开，将得到`(x, x * x, y, x * y, y * y)`；如果进行3次多项式展开，那么将得到`(x, y, x * x, y * y, x * y, x * x * y , x * y * y，x * x * x， y * y * y)`。对于特征向量(x, y, z)，如果进行2次多项式展开，那么将得到`(x, y, z, x * x, y * y, z * z, x * y, x * z , y * z)`。

##### 例

把特征展开到3次多项式空间：

```scala
import org.apache.spark.ml.feature.PolynomialExpansion
import org.apache.spark.ml.linalg.Vectors

val data = Array(
  Vectors.dense(2.0, 1.0),
  Vectors.dense(0.0, 0.0),
  Vectors.dense(3.0, -1.0)
)
val df = spark.createDataFrame(data.map(Tuple1.apply)).toDF("features")

val polyExpansion = new PolynomialExpansion()
  .setInputCol("features")
  .setOutputCol("polyFeatures")
  .setDegree(3)

val polyDF = polyExpansion.transform(df)
polyDF.show(false)
```

#### 7、Discrete Cosine Transform （DCT）

Discrete Cosine Transform（离散余弦变换），把时域（time domain）中一个长度为`n`的实数值序列转换为频域（frequency domain）中长度为`n`的实数值序列。

##### 例

```scala
import org.apache.spark.ml.feature.DCT
import org.apache.spark.ml.linalg.Vectors

val data = Seq(
  Vectors.dense(0.0, 1.0, -2.0, 3.0),
  Vectors.dense(-1.0, 2.0, 4.0, -7.0),
  Vectors.dense(14.0, -2.0, -5.0, 1.0))

val df = spark.createDataFrame(data.map(Tuple1.apply)).toDF("features")

val dct = new DCT()
  .setInputCol("features")
  .setOutputCol("featuresDCT")
  .setInverse(false)

val dctDf = dct.transform(df)
dctDf.select("featuresDCT").show(false)
```

#### 8、StringIndexer

`StringIndexer`将标签的字符串列转换为标签索引列。索引介于区间`[0, numLabels)`，按照标签频率排序，所以最频繁的标签的索引是0。如果用户选择保留未出现过的标签，这些标签会被赋予索引`numLables`。如果输入列是数值，则将它转为字符串并对字符串值进行索引。当下游的管道组件（比如`Estimator`或者`Transformer`）使用这种字符串到索引的标签时，必须指定组件的输入列为这个字符串到索引的列的名称。在许多情况下，都可以使用`setInputCol`设置输入列。

##### 例

假如有列为`id`和`category`的DataFrame：

```
 id | category
----|----------
 0  | a
 1  | b
 2  | c
 3  | a
 4  | a
 5  | c
```

使用`StringIndexer`对`category`列进行转换后：

```
 id | category | categoryIndex
----|----------|---------------
 0  | a        | 0.0
 1  | b        | 2.0
 2  | c        | 1.0
 3  | a        | 0.0
 4  | a        | 0.0
 5  | c        | 1.0
```

“a”出现频率最高，它的索引为0，“c”次之索引为1，“b”索引为2。

当使用基于一个数据集拟合出的`StringIndexer`去转换另一个数据集时，有三种处理未出现过的标签的策略：

- error：抛出异常（默认）
- skip：跳过包含未出现过的标签的整行
- keep：将未出现过的标签放到一个特殊的桶中，赋予索引`numLabels`

比如，使用上面数据集的`StringIndexer`处理下面的数据集：

```
 id | category
----|----------
 0  | a
 1  | b
 2  | c
 3  | d
 4  | e
```

如果未设置处理未出现标签的策略或者设置处理策略为“error”，那么会抛出异常。但是，如果调用了`setHandleInvalid("skip")`，那么会生成如下数据集：

```
 id | category | categoryIndex
----|----------|---------------
 0  | a        | 0.0
 1  | b        | 2.0
 2  | c        | 1.0
```

结果中没有包含“d”或“e”的行。

如果调用`setHandleInvalid("keep")`，生成的数据集如下：

```
 id | category | categoryIndex
----|----------|---------------
 0  | a        | 0.0
 1  | b        | 2.0
 2  | c        | 1.0
 3  | d        | 3.0
 4  | e        | 3.0
```

其中“d”和“e”被映射为索引“3.0”。

```scala
import org.apache.spark.ml.feature.StringIndexer

val df = spark.createDataFrame(
  Seq((0, "a"), (1, "b"), (2, "c"), (3, "a"), (4, "a"), (5, "c"))
).toDF("id", "category")

val indexer = new StringIndexer()
  .setInputCol("category")
  .setOutputCol("categoryIndex")

val indexed = indexer.fit(df).transform(df)
indexed.show()
```

#### 9、IndexToString

与`StringIndexer`对称，`IndexToString`将标签索引列映射为原始标签字符串列。通常的使用场景是，使用`StringIndexer`把标签转换为索引，使用这些索引训练模型，然后使用`IndexToString`将预测的结果索引转换为原始的标签。

##### 例

基于前例的`StringIndexer`，假如有数据集：

```
 id | categoryIndex
----|---------------
 0  | 0.0
 1  | 2.0
 2  | 1.0
 3  | 0.0
 4  | 0.0
 5  | 1.0
```

对它应用`IndexToString`，则能获取到对应的原始标签（通过`StringIndexer`的输出可以推导出列的metadata）：

```
 id | categoryIndex | originalCategory
----|---------------|-----------------
 0  | 0.0           | a
 1  | 2.0           | b
 2  | 1.0           | c
 3  | 0.0           | a
 4  | 0.0           | a
 5  | 1.0           | c
```

代码：

```scala
import org.apache.spark.ml.attribute.Attribute
import org.apache.spark.ml.feature.{IndexToString, StringIndexer}

val df = spark.createDataFrame(Seq(
  (0, "a"),
  (1, "b"),
  (2, "c"),
  (3, "a"),
  (4, "a"),
  (5, "c")
)).toDF("id", "category")

val indexer = new StringIndexer()
  .setInputCol("category")
  .setOutputCol("categoryIndex")
  .fit(df)
val indexed = indexer.transform(df)

println(s"Transformed string column '${indexer.getInputCol}' " +
    s"to indexed column '${indexer.getOutputCol}'")
indexed.show()

val inputColSchema = indexed.schema(indexer.getOutputCol)
println(s"StringIndexer will store labels in output column metadata: " +
    s"${Attribute.fromStructField(inputColSchema).toString}\n")

val converter = new IndexToString()
  .setInputCol("categoryIndex")
  .setOutputCol("originalCategory")

val converted = converter.transform(indexed)

println(s"Transformed indexed column '${converter.getInputCol}' back to original string " +
    s"column '${converter.getOutputCol}' using labels in metadata")
converted.select("id", "categoryIndex", "originalCategory").show()
```

#### 10、OneHotEncoder

One-hot编码是将标签索引列映射为二进制向量（binary vector，仅仅由`0`和`1`两种值构成的向量）列，每个向量最多有一个值为`1`。这种编码方式适用于需要连续特征的算法，比如逻辑回归；当逻辑回归要使用类别特征（categorical features，使用类别特征时，一般都先用`StringIndexer`进行编码）时，要使用one-hot编码对类别特征进行编码。

##### 例

```scala
import org.apache.spark.ml.feature.{OneHotEncoder, StringIndexer}

val df = spark.createDataFrame(Seq(
  (0, "a"),
  (1, "b"),
  (2, "c"),
  (3, "a"),
  (4, "a"),
  (5, "c"),
  (6, "d")
)).toDF("id", "category")

val indexer = new StringIndexer()
  .setInputCol("category")
  .setOutputCol("categoryIndex")
  .fit(df)
val indexed = indexer.transform(df)

val encoder = new OneHotEncoder()
  .setInputCol("categoryIndex")
  .setOutputCol("categoryVec")

val encoded = encoder.transform(indexed)
encoded.show()
```

输出结果：

```
| id|category|categoryIndex|  categoryVec|
+---+--------+-------------+-------------+
|  0|       a|          0.0|(3,[0],[1.0])|
|  1|       b|          3.0|    (3,[],[])|
|  2|       c|          1.0|(3,[1],[1.0])|
|  3|       a|          0.0|(3,[0],[1.0])|
|  4|       a|          0.0|(3,[0],[1.0])|
|  5|       c|          1.0|(3,[1],[1.0])|
|  6|       d|          2.0|(3,[2],[1.0])|
```

#### 11、VectorIndexer

`VectorIndexer`用于对`Vector`数据集中的类别特征进行索引。它可以自动地决定哪些特征是类别特征，并且能够将原始值转换为类别索引（category indices）。特别是，它进行了如下处理：

1. 输入为`Vector`类型列，指定参数`maxCategories`。
2. 基于不同值（distinct values）的数量决定那些特征是类别特征，其中声明了最多`maxCategorices`个特征是类别的。
3. 为每个类别特征计算基于0的索引。
4. 对类别特征进行索引并将特征值转换为索引。

对类别特征进行索引可以让算法（比如，决策树Decision Trees，树团体Tree Ensembles）恰当地处理类别特征，提升性能。

##### 例

读取标记点（labled points）数据集，使用`VectorIndex`来决定哪些特征是类别特征。将类别特征转换为索引。转换后的数据可以传递给处理类别特征的算法（比如`DecisionTreeRegressor`）。

```scala
import org.apache.spark.ml.feature.VectorIndexer

val data = spark.read.format("libsvm").load("data/mllib/sample_libsvm_data.txt")

val indexer = new VectorIndexer()
  .setInputCol("features")
  .setOutputCol("indexed")
  .setMaxCategories(10)

val indexerModel = indexer.fit(data)

val categoricalFeatures: Set[Int] = indexerModel.categoryMaps.keys.toSet
println(s"Chose ${categoricalFeatures.size} categorical features: " +
  categoricalFeatures.mkString(", "))

// Create new column "indexed" with categorical values transformed to indices
val indexedData = indexerModel.transform(data)
indexedData.show()
```

