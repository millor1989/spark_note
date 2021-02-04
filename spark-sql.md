### [Spark SQL,DataFrames and Datasets Guide](http://spark.apache.org/docs/2.2.0/sql-programming-guide.html#overview)

Spark SQL是用于结构化数据处理的Spark模块。与基础的Spark RDD API不同，Spark SQL提供的接口，为Spark提供了更多的关于数据和执行的运算的结构有关的信息。在内部，Spark SQL使用这些额外的信息进行额外的优化。和Spark SQL进行交互的方式包括SQL和Dataset API。不管使用什么API（或者编程语言）来描述运算，进行运算时使用的是相同的执行引擎。

#### SQL

Spark SQL的一种用途是执行SQL查询。Spark SQL也可以用于从Hive读取数据。从其它编程语言内部运行SQL时，结果会被返回为Dataset/DataFrame。也可以使用命令行或者JDBC/ODBC通过SQL接口进行交互。

#### Datasets和DataFrames

Dataset是一个分布式的数据集合。Dataset是从Spark 1.6开始引入的新接口，既具有RDD优势，又具有Spark SQL优化的执行引擎带来的优势。Dataset可以从JVM对象来构建，并且可以进行函数式的转换（比如`map`、`flatMap`、`filter`等等）。

DataFrame是一个具有命名的列的分布式数据集。在概念上与关系型数据库的表或者R/Python中的data frame等价，但是Spark对它进行了丰富的优化。可以从多个来源构建DataFrame，比如结构化的数据文件、Hive中的表、外部数据库或者RDDs。Spark有Scala、Java、Python和R版本的DataFrame API。在Scala和Java API中，DataFrame由`Row`s的Dataset代表。在Scala API中，`DataFrame`仅仅是`Dataset[Row]`的一个类型别名。而在Java API中，需要使用`Dataset<Row>`来表示`DataFrame`。

