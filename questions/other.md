##### `org.apache.spark.ml.linalg.Vectors.sparse(size: Int, elements: Seq[(Int, Double)])` 方法要求 `elements` 参数中的索引不能重复，否则会抛出类似于如下的异常：

```
Caused by: java.lang.IllegalArgumentException: requirement failed: Index 25 follows 25 and is not strictly increasing
	at scala.Predef$.require(Predef.scala:224)
	at org.apache.spark.ml.linalg.SparseVector$$anonfun$1.apply$mcVI$sp(Vectors.scala:580)
	at org.apache.spark.ml.linalg.SparseVector$$anonfun$1.apply(Vectors.scala:579)
	at org.apache.spark.ml.linalg.SparseVector$$anonfun$1.apply(Vectors.scala:579)
	at scala.collection.IndexedSeqOptimized$class.foreach(IndexedSeqOptimized.scala:33)
	at scala.collection.mutable.ArrayOps$ofInt.foreach(ArrayOps.scala:234)
	at org.apache.spark.ml.linalg.SparseVector.<init>(Vectors.scala:579)
	at org.apache.spark.ml.linalg.Vectors$.sparse(Vectors.scala:226)
```

