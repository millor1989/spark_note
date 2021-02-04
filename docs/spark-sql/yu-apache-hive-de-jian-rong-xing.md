### 与Apache Hive的兼容性

Spark SQL被设计为兼容Hive Metastore，SerDes和UDFs。目前Hive SerDes和UDFs是基于Hive 1.2.1，并且Spark SQL可以连接不同版本的Hive Metastore（从0.12.0到2.1.1）。

#### 1、在既存的Hive仓库部署

Spark SQL的Thrift JDBC服务器是对既存Hive开箱即用的。不用修改Hive Metastore或者改变表的数据存放和分区。

#### 2、支持的Hive特征

Spark SQL supports大多数的Hive features, 比如：

- Hive查询语句，包括：
  - `SELECT`
  - `GROUP BY`
  - `ORDER BY`
  - `CLUSTER BY`
  - `SORT BY`
- 所有的Hive操作符，包括：
  - 关系运算符 (`=`, `⇔`, `==`, `<>`, `<`, `>`, `>=`, `<=`, etc)
  - 算术运算符 (`+`, `-`, `*`, `/`, `%`, etc)
  - 逻辑运算符 (`AND`, `&&`, `OR`, `||`, etc)
  - 复杂类型构造函数
  - 数学函数 (`sign`, `ln`, `cos`, etc)
  - 字符串函数 (`instr`, `length`, `printf`, etc)
- 用户定义函数 (UDF)
- 用户定义聚合函数 (UDAF)
- 用户定义序列化格式 (SerDes)
- 窗口函数
- 连接（Joins）
  - `JOIN`
  - `{LEFT|RIGHT|FULL} OUTER JOIN`
  - `LEFT SEMI JOIN`
  - `CROSS JOIN`
- 合并（union）
- 子查询    
  - `SELECT col FROM ( SELECT a + b AS col from t1) t2`
- 抽样（Sampling）
- 分析（Explain）
- 分区表包括动态分区插入（Partitioned tables including dynamic partition insertion）
- 视图（View）
- 所有的Hive DDL语句，包括:     
  - `CREATE TABLE`
  - `CREATE TABLE AS SELECT`
  - `ALTER TABLE`
- 大多数的 Hive数据类型，包括：
  - `TINYINT`
  - `SMALLINT`
  - `INT`
  - `BIGINT`
  - `BOOLEAN`
  - `FLOAT`
  - `DOUBLE`
  - `STRING`
  - `BINARY`
  - `TIMESTAMP`
  - `DATE`
  - `ARRAY<>`
  - `MAP<>`
  - `STRUCT<>`

#### 3、不支持的Hive功能

Spark SQL不支持的Hive功能，大多都是不常用的。

##### 3.1、重要Hive特征

- 分桶的表（Tables with buckets）：桶（bucket）是Hive表分区内部的哈希分布（hash partitioning）。Spark SQL目前还不支持。

##### 3.2、深奥的Hive特征

- `UNION`类型
- Unique join
- 列统计数据收集（Column statistics collecting）：

##### 3.3、Hive输入输出格式

- CLI的格式：对于返回给CLI展示的结果，Spark SQL只支持`TextOutputFormat`
- Hadoop封装（Hadoop archive）

##### 3.4、Hive优化

很少的几个Hive优化还没有被Spark SQL包括。有些因为Spark SQL的在内存运算模型而不重要，其他的可能会在Spark SQL未来的版本中实现。

- block级别的bitmap索引和虚拟列（用于构建索引）
- 自动地确定用于joins和groupbys的reducers的数量：在当前的Spark SQL中，需要通过“`SET spark.sql.shuffle.partitions=[num_tasks];`”控制混洗（post-shuffle）并行化级别。
- 只针对元数据的查询（Meta-data only query）：对于仅仅使用meta data的查询，Spark SQL仍然会启动tasks来运算结果。
- 倾斜数据标识：Spark SQL不关注Hive中的数据倾斜标识。
- 连接中的`STREAMTABLE`暗示（hint）：Spark SQL不关注`STREAMTABLE`暗示。
- 合并查询结果的多个小文件：如果结果输出包含多个小文件，Hive可以选择性的把小文件合并为更大的文件，以避免HDFS metadata溢出。Spark SQL则不支持。

