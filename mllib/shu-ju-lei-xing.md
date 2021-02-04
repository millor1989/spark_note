### 数据类型

MLlib支持位于一台机器上的本地向量（local vectors）和矩阵（matrices），也支持基于一个或者多个RDD的分布式矩阵。本地向量和本地矩阵都是作为公开接口提供的简单数据模型。底层的线性代数操作由Breeze提供。

#### 1、本地向量（Local vector）

本地向量的索引是0开始的integer类型，值是double类型，存储在一台机器上。MLlib支持两种类型的本地向量：密集向量和稀疏向量。密集向量基于一个double数组，表示它的元素值，而稀疏向量基于两个并行的数组：索引数组和值数组。比如向量`(1.0, 0.0, 3.0)` 可以用密集向量形式`[1.0, 0.0, 3.0]`表示；也可以用稀疏向量形式`(3, [0, 2], [1.0, 3.0])`表示，其中`3`是向量的大小，第一个数组为非零索引数组，第二个数组为与索引对应的非零值。

本地向量的基础类是`Vector`，有两种实现`DenseVector`和`SparseVector`。推荐使用`Vectors`类中的工厂方法创建本地向量。

```scala
import org.apache.spark.ml.linalg.{Vector, Vectors}

// Create a dense vector (1.0, 0.0, 3.0).
val dv: Vector = Vectors.dense(1.0, 0.0, 3.0)
// Create a sparse vector (1.0, 0.0, 3.0) by specifying its indices and values corresponding to nonzero entries.
val sv1: Vector = Vectors.sparse(3, Array(0, 2), Array(1.0, 3.0))
// Create a sparse vector (1.0, 0.0, 3.0) by specifying its nonzero entries.
val sv2: Vector = Vectors.sparse(3, Seq((0, 1.0), (2, 3.0)))
```

注意，Scala默认导入`scala.collection.immutable.Vector` ，所以要明确地导入`org.apache.spark.ml.linalg.Vector` 以使用MLlib的`Vector`。

#### 2、标记点（Labeled point）

标记点是一个本地向量（密集向量或稀疏向量）和一个相关的标签/响应（label/response）。在MLlib中，标记点用于监督式学习算法。使用double来存储标签，所以分类和回归都可以使用标记点。对于二元分类，标签应该是0（negative）或1（positive）。对于多类型分类，标签应该是从0开始的类型索引：0，1，2，...。

```scala
import org.apache.spark.ml.linalg.Vectors
import org.apache.spark.ml.feature.LabeledPoint

// Create a labeled point with a positive label and a dense feature vector.
val pos = LabeledPoint(1.0, Vectors.dense(1.0, 0.0, 3.0))

// Create a labeled point with a negative label and a sparse feature vector.
val neg = LabeledPoint(0.0, Vectors.sparse(3, Array(0, 2), Array(1.0, 3.0)))
```

##### 2.1、稀疏数据

实践中经常会用到稀疏训练数据。MLlib支持读取以`LIBSVM`格式保存的训练数据，这种格式是`LIBSVM`和`LIBLINEAR`默认使用的格式。这种格式是一种文本格式，每行表示一个带标签的稀疏特征向量，格式如下：

```
label index1:value1 index2:value2 ...
```

其中索引是按照升序从1开始的。加载之后，特征索引被转换为从0开始的。

```scala
val data = spark.read.format("libsvm").load("data/mllib/sample_libsvm_data.txt")
```

#### 3、本地矩阵（Local matrix）

本地矩阵的行和列索引是integer类型的，值是double类型的，保存在一台机器上。MLlib支持密集矩阵，元素值按照**列主序**（column-major order）保存在一个double数组中；MLlib也支持稀疏矩阵，非零值按照列主序以压缩稀疏列（CSC，Compressed Sparse Column）格式保存。

本地矩阵的基础类是`Matrix`，有两种实现：`DenseMatrix`和`SparseMatrix`。推荐使用`Matricies`的工厂方法创建本地矩阵。MLlib中的本地矩阵是按照列主序保存的。

```scala
import org.apache.spark.ml.linalg.{Matrices, Matrix}

// Create a dense matrix ((1.0, 2.0), (3.0, 4.0), (5.0, 6.0))
val dm: Matrix = Matrices.dense(3, 2, Array(1.0, 3.0, 5.0, 2.0, 4.0, 6.0))

// Create a sparse matrix ((9.0, 0.0), (0.0, 8.0), (0.0, 6.0))
val sm: Matrix = Matrices.sparse(3, 2, Array(0, 1, 3), Array(0, 2, 1), Array(9, 6, 8))
```

