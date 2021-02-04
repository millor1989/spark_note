### 基础数据统计（Basic Statistics）

#### 1、相关性（Correlation）

数据统计中计算两组（series）数据的相关性是常见的操作。`spark.ml`具有计算多组数据中成对相关性的灵活性。目前支持的相关性方法有**Pearson**和**Spearman**。相关性不表示因果关系。

Pearson相关系数：用于连续变量，假设两组变量均为正态分布、存在线性关系并且等方差，受异常值影响较大。用线性相关系数来反映两组数据线性相关程度，描述的是**线性相关**。

Spearman相关系数：是一种无参数（与分布无关）的检验方法，用于度量变量之间联系的强弱。根据原始数据的排序位置进行求解。无论两组变量数据如何变化，只关心数值在变量内的排列顺序，如果两个变量的对应值在各组内排序相同或类似则具有显著相关性。在没有重复数据的情况下，如果一个变量是另一个变量的严格单调函数，则Spearman相关系数为-1或+1，称为完全Spearman相关。对数据条件要求没Pearson相关严格，对数据错误和极端值反应不敏感。

Pearson相关系数和Spearman相关系数介于-1到+1之间，大于0表示正相关，小于0表示负相关；绝对值越大相关性越强；趋于0则表示两者相关度较弱。

**例**

使用`Correlation`计算输入Vectors的Dataset的相关系数矩阵。输出是一个包含向量的列的相关系数矩阵的DataFrame。

```scala
import org.apache.spark.ml.linalg.{Matrix, Vectors}
import org.apache.spark.ml.stat.Correlation
import org.apache.spark.sql.Row

val data = Seq(
  Vectors.sparse(4, Seq((0, 1.0), (3, -2.0))),
  Vectors.dense(4.0, 5.0, 0.0, 3.0),
  Vectors.dense(6.0, 7.0, 0.0, 8.0),
  Vectors.sparse(4, Seq((0, 9.0), (3, 1.0)))
)

val df = data.map(Tuple1.apply).toDF("features")

// Pearson相关
val Row(coeff1: Matrix) = Correlation.corr(df, "features").head
println("Pearson correlation matrix:\n" + coeff1.toString)

// Spearman相关
val Row(coeff2: Matrix) = Correlation.corr(df, "features", "spearman").head
println("Spearman correlation matrix:\n" + coeff2.toString)
```

#### 2、假设检验（Hypothesis testing）

假设检验是数据统计的一个强大工具，可以用来决定结果是否具有统计意义（statistically significant），结果是否是偶尔产生的。`spark.ml`目前支持Pearson的Chi-square卡方（$\chi^2$）独立性测试。如果总体的某一假设是真实的，那么能推翻这一假设的事件几乎不可能发生，但是如果几乎不可能的事件发生了，那么可以怀疑假设的真实性，拒绝这一假设。

`ChiSquareTest`针对标签对每个特征进行Pearson独立性测试。对于每个特征，“特征-标签”对都通过卡方统计计算被转换为偶然性（contingency）矩阵。所有的标签和特征值都必须是分类的（categorical）。

**例**

```scala
import org.apache.spark.ml.linalg.{Vector, Vectors}
import org.apache.spark.ml.stat.ChiSquareTest

val data = Seq(
  (0.0, Vectors.dense(0.5, 10.0)),
  (0.0, Vectors.dense(1.5, 20.0)),
  (1.0, Vectors.dense(1.5, 30.0)),
  (0.0, Vectors.dense(3.5, 30.0)),
  (0.0, Vectors.dense(3.5, 40.0)),
  (1.0, Vectors.dense(3.5, 40.0))
)

val df = data.toDF("label", "features")
val chi = ChiSquareTest.test(df, "features", "label").head
println("pValues = " + chi.getAs[Vector](0))
println("degreesOfFreedom = " + chi.getSeq[Int](1).mkString("[", ",", "]"))
println("statistics = " + chi.getAs[Vector](2))
```

