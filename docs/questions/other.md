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

##### 分区字段应该用字符串类型，不然会去获取所有的分区元数据。

是一个警告信息，是获取所有的**分区元数据**，并不是扫描全表、更不是读取全表的数据。

只要加了分区过滤的 filter 应该就会过滤分区。只是如果分区特别多——获取所有的分区元数据——也是会降低性能的。

```
WARN client.Shim_v1_1: Caught Hive MetaException attempting to get partition metadata by filter from Hive. Falling back to fetching all partition metadata, which will degrade performance. Modifying your Hive metastoreconfiguration to set hive.metastore.try.direct.sql to true may resolve this problem.
...
...
...
Caused by: MetaException(message:Filtering is supported only on partition keys of type string)
....
...
```

##### Impala JDBC

如果不是在 Hadoop 集群中使用 Impala JDBC，除了 impala jdbc 本身的驱动外，还需要[引入额外的的依赖包](https://docs.cloudera.com/documentation/enterprise/5-5-x/topics/impala_jdbc.html)：

```tex
commons-logging-X.X.X.jar
  hadoop-common.jar
  hive-common-X.XX.X-cdhX.X.X.jar
  hive-jdbc-X.XX.X-cdhX.X.X.jar
  hive-metastore-X.XX.X-cdhX.X.X.jar
  hive-service-X.XX.X-cdhX.X.X.jar
  httpclient-X.X.X.jar
  httpcore-X.X.X.jar
  libfb303-X.X.X.jar
  libthrift-X.X.X.jar
  log4j-X.X.XX.jar
  slf4j-api-X.X.X.jar
  slf4j-logXjXX-X.X.X.jar
```

另外，如果需要用户名密码认证（LDAP），jdbc 连接字符串中应该加入 `AuthMech=3`，具体参考[官方文档](https://docs.cloudera.com/documentation/other/connectors/impala-jdbc/latest/Cloudera-JDBC-Driver-for-Impala-Install-Guide.pdf)。

##### Hive JDBC

Hive JDBC 查询需要依赖 `guava`，否则会报类似如下的错误：

```tex
Caused by: java.lang.NoClassDefFoundError: com/google/common/primitives/Ints
	at org.apache.hive.service.cli.Column.<init>(Column.java:150)
	at org.apache.hive.service.cli.ColumnBasedSet.<init>(ColumnBasedSet.java:51)
	at org.apache.hive.service.cli.RowSetFactory.create(RowSetFactory.java:37)
	at org.apache.hive.jdbc.HiveQueryResultSet.next(HiveQueryResultSet.java:367)
```

