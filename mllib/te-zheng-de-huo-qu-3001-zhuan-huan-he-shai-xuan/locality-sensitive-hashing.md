### Locality Sensitive Hashing

Locality Sensitive Hashing（LSH，局部敏感哈希）是哈希技术的一个重要的类，通常用于大型数据集的聚类（clustering），近似最近邻搜索（approximate nearest neighbor search）和离群值检测（outlier detection）。

LSH的一般思路是使用一组函数（a family of functions，“LSH families”）将数据点哈希进桶中，以便彼此相近的数据点具有很大的概率在同一个桶中，而相差较远的数据点很可能在不同的桶中。

在Spark中，不同的LSH family由不同的类实现（比如，`MinHash`），每个类都提供了用于特征转换（feature transformation），近似相似连接（approximate similarity join）和近似最近邻（approximate nearest neighbor）的APIs。

在LSH中，将被哈希到相同的桶中的相距遥远的两个输入特征定义假阳性（false positive），将被哈希到不同桶中的相近特征定义假阴性（false negative）。

#### 1、LSH操作

LSH大体上可以用于三类操作。拟合出的LSH模型有用于这些操作的方法。

##### 1.1、特征转换

特征转换是基础的功能，增加一个哈希值新列。可以用于降维（demensionality reduction）。可以通过`inputCol`和`outputCol`来指定输入、输出列的名称。

LSH也支持多重LSH哈希表。可以通过`numHashTables`来指定哈希表的数量。这也可以用于近似相似连接和近似最近邻中的OR-放大（OR-amplifiction）。增加哈希标的数量能够增加精确性，但是也会增加通信开销和运行时间。

`outputCol`的类型是`Seq[Vector]`，其中数组的维度与`numHashTables`相等，目前向量的维度是1。未来的版本，将会实现AND-放大（AND-amplification）以便可以指定这些向量的维度。

##### 1.2、近似相似连接

近似相似连接，输入两个数据集，近似地返回数据集中距离小于用户定义的阈值的那些行的配对（pairs）。近似相似连接即支持不同数据集的连接也支持自连接（self-joining）。自连接会产生一些重复地配对。

近似相似连接的输入可以是经过转换的数据集，也可以接收未经转换的数据集。如果使用未经转换的数据集，它会自动地被进行转换。这种情况下，会根据`outputCol`创建哈希签名（hash signature）。

在连接后的数据集中，可以在`datasetA`和`datasetB`中查询源数据集。会有一个表示返回的每对行的真是距离的距离列被添加到输出数据集。

##### 1.3、近似最近邻搜索

近似最近邻搜索的输入为一个数据集（多个特征向量）和一个键（一个特征向量），近似地返回数据集中指定数量的与指定向量最近的行。

近似最近邻搜索可以接收转换过的和为转换过的数据集作为输入。如果使用未转换过的数据集，它会被自动地进行转换。这种情况下，会根据`outputCol`创建哈希签名（hash signature）。

会有一个表示每个输出行与被搜索键（searched key）距离的距离列被添加到输出数据集。

注意：当哈希桶中备选记录不足时，近似最近邻搜索会返回少于`k`行。

#### 2、LSH 算法

##### 2.1、Bucketed Random Projection for Euclidean Distance

Bucketed Random Projection（分桶随机投影）是一个Euclidean distance（欧氏距离）的LSH family。用户可以自定义桶长度。桶长度可以用来控制哈希桶的平均大小（和桶的数量）。更大的桶长度（即，更少的桶）能增加特征被哈希到相同桶中的概率（增加了true的数量和假阳性，increasing the numbers of true and false positives）。

分桶随机投影可以接收任意向量作为输入特征，支持稀疏向量和密集向量。

```scala
import org.apache.spark.ml.feature.BucketedRandomProjectionLSH
import org.apache.spark.ml.linalg.Vectors
import org.apache.spark.sql.functions.col

val dfA = spark.createDataFrame(Seq(
  (0, Vectors.dense(1.0, 1.0)),
  (1, Vectors.dense(1.0, -1.0)),
  (2, Vectors.dense(-1.0, -1.0)),
  (3, Vectors.dense(-1.0, 1.0))
)).toDF("id", "features")

val dfB = spark.createDataFrame(Seq(
  (4, Vectors.dense(1.0, 0.0)),
  (5, Vectors.dense(-1.0, 0.0)),
  (6, Vectors.dense(0.0, 1.0)),
  (7, Vectors.dense(0.0, -1.0))
)).toDF("id", "features")

val key = Vectors.dense(1.0, 0.0)

val brp = new BucketedRandomProjectionLSH()
  .setBucketLength(2.0)
  .setNumHashTables(3)
  .setInputCol("features")
  .setOutputCol("hashes")

val model = brp.fit(dfA)

// Feature Transformation
println("The hashed dataset where hashed values are stored in the column 'hashes':")
model.transform(dfA).show()

// Compute the locality sensitive hashes for the input rows, then perform approximate
// similarity join.
// We could avoid computing hashes by passing in the already-transformed dataset, e.g.
// `model.approxSimilarityJoin(transformedA, transformedB, 1.5)`
println("Approximately joining dfA and dfB on Euclidean distance smaller than 1.5:")
model.approxSimilarityJoin(dfA, dfB, 1.5, "EuclideanDistance")
  .select(col("datasetA.id").alias("idA"),
    col("datasetB.id").alias("idB"),
    col("EuclideanDistance")).show()

// Compute the locality sensitive hashes for the input rows, then perform approximate nearest
// neighbor search.
// We could avoid computing hashes by passing in the already-transformed dataset, e.g.
// `model.approxNearestNeighbors(transformedA, key, 2)`
println("Approximately searching dfA for 2 nearest neighbors of the key:")
model.approxNearestNeighbors(dfA, key, 2).show()
```

#### 2.2、MinHash for Jaccard Distance

MinHash是一个Jaccard Distance（提卡距离）的LSH family，输入特征是自然数集合。两个集合的提卡距离由它们交集和并集的基数（cardinality）定义。MinHash对集合中的每个元素应用一个随机哈希函数并获取所有哈希值的最小值。

MinHash的输入集合由二进制向量表示，向量的索引表示元素本身，向量中的非零值表示集合中存在的元素。稀疏向量和密集向量都支持，为了高效一般推荐使用稀疏向量。比如，`Vectors.sparse(10, Array[(2, 1.0), (3, 1.0), (5, 1.0)])` 表示得向量有10个元素，集合包含元素2，元素3，和元素5.所有的非零值都被当作二进制值“1”。

注意：MinHash不能转换空集，这意味着任意的数据向量必须至少有一个非零元素。

```scala
import org.apache.spark.ml.feature.MinHashLSH
import org.apache.spark.ml.linalg.Vectors
import org.apache.spark.sql.functions.col

val dfA = spark.createDataFrame(Seq(
  (0, Vectors.sparse(6, Seq((0, 1.0), (1, 1.0), (2, 1.0)))),
  (1, Vectors.sparse(6, Seq((2, 1.0), (3, 1.0), (4, 1.0)))),
  (2, Vectors.sparse(6, Seq((0, 1.0), (2, 1.0), (4, 1.0))))
)).toDF("id", "features")

val dfB = spark.createDataFrame(Seq(
  (3, Vectors.sparse(6, Seq((1, 1.0), (3, 1.0), (5, 1.0)))),
  (4, Vectors.sparse(6, Seq((2, 1.0), (3, 1.0), (5, 1.0)))),
  (5, Vectors.sparse(6, Seq((1, 1.0), (2, 1.0), (4, 1.0))))
)).toDF("id", "features")

val key = Vectors.sparse(6, Seq((1, 1.0), (3, 1.0)))

val mh = new MinHashLSH()
  .setNumHashTables(5)
  .setInputCol("features")
  .setOutputCol("hashes")

val model = mh.fit(dfA)

// Feature Transformation
println("The hashed dataset where hashed values are stored in the column 'hashes':")
model.transform(dfA).show()

// Compute the locality sensitive hashes for the input rows, then perform approximate
// similarity join.
// We could avoid computing hashes by passing in the already-transformed dataset, e.g.
// `model.approxSimilarityJoin(transformedA, transformedB, 0.6)`
println("Approximately joining dfA and dfB on Jaccard distance smaller than 0.6:")
model.approxSimilarityJoin(dfA, dfB, 0.6, "JaccardDistance")
  .select(col("datasetA.id").alias("idA"),
    col("datasetB.id").alias("idB"),
    col("JaccardDistance")).show()

// Compute the locality sensitive hashes for the input rows, then perform approximate nearest
// neighbor search.
// We could avoid computing hashes by passing in the already-transformed dataset, e.g.
// `model.approxNearestNeighbors(transformedA, key, 2)`
// It may return less than 2 rows when not enough approximate near-neighbor candidates are
// found.
println("Approximately searching dfA for 2 nearest neighbors of the key:")
model.approxNearestNeighbors(dfA, key, 2).show()
```

