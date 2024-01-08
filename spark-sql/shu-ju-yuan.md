### 数据源

Spark SQL通过DataFrame接口支持多种多样的数据源。可以通过使用关系型的转换或者通过创建临时视图来操作DataFrame。把DataFrame注册为一个临时视图后，可以使用SQL查询对他的数据进行操作。

#### 1、通用的Load/Save函数

最简单的形式，默认的数据源（`parquet`，除非通过`spark.sql.sources.default`另外指定）将被用于所有的操作。

```scala
val usersDF = spark.read.load("examples/src/main/resources/users.parquet")
usersDF.select("name", "favorite_color").write.save("namesAndFavColors.parquet")
```

##### 1.1、指定选项（Manually Specifying Options）

也可以为数据源指定一些需要的选项。数据源一般通过全限定名（比如，`org.apache.spark.sql.parquet`）来指定，对于内置的数据源可以通过简称（`json`, `parquet`, `jdbc`, `orc`, `libsvm`, `csv`, `text`）指定。使用这种DataFrame语法可以将任意数据源类型转换为其它类型。

```scala
val peopleDF = spark.read.format("json").load("examples/src/main/resources/people.json")
peopleDF.select("name", "age").write.format("parquet").save("namesAndAges.parquet")
```

##### 1.2、直接对文件运行SQL

除了通过`read` API将文件导入DataFrame然后进行查询，还可以直接使用SQL查询文件。

```scala
val sqlDF = spark.sql("SELECT * FROM parquet.`examples/src/main/resources/users.parquet`")
```

##### 1.3、Save Modes

`save`操作有一个可选的选项`mode`使用`SaveMode`做参数——指定如果数据已经存在该如何处理。这些保存模式既不使用锁机制也不是原子操作。另外，使用`Overwrite`保存模式时，新的数据在执行保存之前会删除已有的数据。

| Scala/Java | Any Language | Meaning |
| --- | --- | --- |
| `SaveMode.ErrorIfExists` （**默认**） | `"error"` \(default\) | 如果数据已经存在，则期望抛出异常。 |
| `SaveMode.Append` | `"append"` | When saving a DataFrame to a data source, if data/table already exists,     contents of the DataFrame are expected to be appended to existing data. |
| `SaveMode.Overwrite` | `"overwrite"` | 如果数据/表已经存在，则期望存在的数据会被DataFrame中的数据覆盖 |
| `SaveMode.Ignore` | `"ignore"` | 如果数据已经存在，则期望不保存DataFrame的内容，并且不改变已经存在的数据。 |

##### 1.4、保存到持久化表（Saving to Persistent Tables）

`DataFrame`还可以通过`saveAsTable`命令保存为Hive metastore中的持久化表。注意，即使没有部署Hive也可以使用这个特征。Spark会创建一个默认的本地Hive metastore（使用Derby）。与`createOrReplaceTempView`不同，`saveAsTable` 会把DataFrame的内容进行物化（materialize）并且在Hive metastore中创建一个数据的指针。即使Spark程序重启持久化表仍然存在，只要能把连接指向相同的metastore。可以通过`SparkSession`的`table`方法，用持久化表的名称，就能创建持久化表的DataFrame。

对于基于文件的数据源，比如text, parquet, json,等等，可以通过`path`选项指定表路径。比如，`df.write.option("path", "/some/path").saveAsTable("t")`；删除表后，表路径和表数据仍然存在。如果不指定表路径，Spark会把数据保存到仓库目录下的一个默认表路径；此时，如果删除表，默认表路径也会被删除。

从Spark 2.1开始，持久化的数据源表把每个分区的metadata保存在了Hive metastore中。这带来了几点好处：

* metastore可以只为某个查询返回必要的分区，不用再查询整个表的全部分区。
* Hive的DDLs比如`ALTER TABLE PARTITION ... SET LOCATION`对DataSource API创建的表也可用了。

注意，当创建外部数据源表（使用`path`选项创建的）默认不会收集分区信息。可以调用`MSCK REPAIR TABLE`来同步metastore中的分区信息。

##### 1.5、Bucketing，Sorting和Partitioning

对于基于文件的数据源，还可以对输出进行bucket、sort、partition。bucket和sort只能用于持久化表。

```scala
peopleDF.write.bucketBy(42, "name").sortBy("age").saveAsTable("people_bucketed")
```

而使用Dataset API时，partition操作可以用于`save`和`saveAsTable`。

```scala
usersDF.write.partitionBy("favorite_color").format("parquet").save("namesPartByColor.parquet")
```

可以对一个表同时进行partition和bucket：

```scala
peopleDF
  .write
  .partitionBy("favorite_color")
  .bucketBy(42, "name")
  .saveAsTable("people_partitioned_bucketed")
```

`partitionBy`会创建分区目录结构。所以，对于高基数（high cardinality）的列适用性有限。而`bucketBy`把数据分散到固定数量的buckets中，并且可以在唯一值得数量不确定（unbounded）时使用。

#### 2、Parquet文件

Parquet是一种被许多其它数据处理系统支持的面向列的格式。Spark SQL提供了对Parquet文件的读写支持，并且会自动保留原始数据的schema。当写Parquet文件时，为了兼容性（compatibility）的原因所有的列都被自动转换为可以为NULL的（nullable）。

##### 2.1、程式化的加载数据

```scala
// Encoders for most common types are automatically provided by importing spark.implicits._
import spark.implicits._

val peopleDF = spark.read.json("examples/src/main/resources/people.json")

// DataFrames can be saved as Parquet files, maintaining the schema information
peopleDF.write.parquet("people.parquet")

// Read in the parquet file created above
// Parquet files are self-describing so the schema is preserved
// The result of loading a Parquet file is also a DataFrame
val parquetFileDF = spark.read.parquet("people.parquet")

// Parquet files can also be used to create a temporary view and then used in SQL statements
parquetFileDF.createOrReplaceTempView("parquetFile")
val namesDF = spark.sql("SELECT name FROM parquetFile WHERE age BETWEEN 13 AND 19")
namesDF.map(attributes => "Name: " + attributes(0)).show()
// +------------+
// |       value|
// +------------+
// |Name: Justin|
// +------------+
```

##### 2.2、分区发现

在像Hive这种系统中，表分区是一个通用的优化方法。在分区表中，数据通常保存在不同的目录中，分区列的值被编码到每个分区目录的路径。Parquet数据源可以自动地发现和推导分区信息。比如用`gender` 和 `country` 作为分区字段：

```reStructuredText
path
└── to
    └── table
        ├── gender=male
        │   ├── ...
        │   │
        │   ├── country=US
        │   │   └── data.parquet
        │   ├── country=CN
        │   │   └── data.parquet
        │   └── ...
        └── gender=female
            ├── ...
            │
            ├── country=US
            │   └── data.parquet
            ├── country=CN
            │   └── data.parquet
            └── ...
```

把`path/to/table`传递给`SparkSession.read.parquet`或者`SparkSession.read.load`，Spark SQL都能够从路径中获取分区信息。返回的DataFrame的schema将会是：

```reStructuredText
root
|-- name: string (nullable = true)
|-- age: long (nullable = true)
|-- gender: string (nullable = true)
|-- country: string (nullable = true)
```

注意，分区字段的数据类型时自动推导的。数值数据类型和字符串类型都是支持的。如果不想让Spark自动推导分区字段的数据类型，可以把`spark.sql.sources.partitionColumnTypeInference.enabled`配置项（默认为`true`）设置为`false`，类型推导关闭后，将会使用字符串类型作为分区字段类型。

从Spark 1.6开始，分区发现默认只会查找指定的路径下的分区。对于上面的例子，如果用户传递的路径是`path/to/table/gender=male`，`gender`将不会被当作一个分区字段。如果需要指定分区发现的起始目录，可以设置数据源的`basePath`选项。例如，当设置`path/to/table/gender=male`作为数据的路径并且设置`path/to/table/`为`basePath` 时，`gender`将会被作为分区字段。

##### 2.3、Schema合并（Schema Merging）

与ProtocolBuffer，Avro和Thrift，Parquet也支持schema演变。Parquet文件可以从简单的schema开始，渐渐地根据需要增加更多的列到schema。这样，用户可能最后会有多个Parquet文件，这些文件拥有不同的但是相互兼容的（different but mutually compatible）schema。Parquet数据源可以自动侦测这种情况并且合并全部这些文件的schema。

schema合并是相对高开销的操作，在大多数情况下并不必要，Spark从1.5.0开始默认将这个功能关闭。若要开启可以：1、在读取Parquet文件时，设置数据源选项`mergeSchema` 为`true`（如下面代码示例），或者2、设置全局的SQL选项`spark.sql.parquet.mergeSchema` 为`true`。

```scala
// This is used to implicitly convert an RDD to a DataFrame.
import spark.implicits._

// Create a simple DataFrame, store into a partition directory
val squaresDF = spark.sparkContext.makeRDD(1 to 5).map(i => (i, i * i)).toDF("value", "square")
squaresDF.write.parquet("data/test_table/key=1")

// Create another DataFrame in a new partition directory,
// adding a new column and dropping an existing column
val cubesDF = spark.sparkContext.makeRDD(6 to 10).map(i => (i, i * i * i)).toDF("value", "cube")
cubesDF.write.parquet("data/test_table/key=2")

// Read the partitioned table
val mergedDF = spark.read.option("mergeSchema", "true").parquet("data/test_table")
mergedDF.printSchema()

// The final schema consists of all 3 columns in the Parquet files together
// with the partitioning column appeared in the partition directory paths
// root
//  |-- value: int (nullable = true)
//  |-- square: int (nullable = true)
//  |-- cube: int (nullable = true)
//  |-- key: int (nullable = true)
```

##### 2.4、Hive metastore Parquet表转换

读写Hive metastore中的Parquet表时，Spark SQL为了更好的性能会使用自己的Parquet支持替代Hive SerDe。这个行为是由`spark.sql.hive.convertMetastoreParquet` 配置控制的，默认是开启的。

###### 2.4.1、Hive/Parquet Schema协调（Hive/Parquet Schema Reconciliation）

在schema处理的角度，Hive和Parquet有两个关键的不同：

1. Hive是大小写不敏感的，Parquet不是；
2. Hive认为所有的列都是可以为null的（nullable），Parquet中可空性是重要的（while nullability in Parquet is significant）。

因此，当把Hive metastore Parquet 表转换为Spark SQL Parquet表时，必须要协调Hive metastore schema和Parquet schema。协调规则是：

1. 两边schema中具有相同名称的字段必须有相同的数据类型，不管可控性（regardless of nullability）。协调后的字段应该与Parquet侧schema有相同的数据类型，以便重视可空性（nullability is respected）。

2. 协调后的schema必须包含Hive metastore schema中的字段。

   1）只在Parquet schema中出现的字段在协调后的shcema中已被删除

   2）只在Hive metastore schema中出现的字段，被加到协调后的schema中并且是可空的。

###### 2.4.2、Metadata刷新

为了更好的性能，Spark SQL缓存了Parquet metadata。当开启了Hive metastore Parquet表转换，这些转换后的表metadata也会被缓存。如果这些表被Hive或者其它外部工具更新，需要手动的刷新以确保metadata一致。

```scala
// spark is an existing SparkSession
spark.catalog.refreshTable("my_table")
```

##### 2.5、配置

通过`SparkSession`的`setConf`方法或者使用SQL运行`SET key=value`可以对Parquet进行配置。

| Property Name | Default | Meaning |
| --- | --- | --- |
| `spark.sql.parquet.binaryAsString` | false | 某些其它的Parquet生成系统，特别是Impala，Hive，和低版本的Spark SQL，在写出Parquet schema时并不区分二进制数据和字符串。这个配置告诉Spark SQL把二进制数据作为字符串以和这些系统进行兼容。 |
| `spark.sql.parquet.int96AsTimestamp` | true | 某些其它的Parquet生成系统，特别是Impala，Hive，把时间戳保存在INT96中。这个配置告诉Spark SQL把INT96数据作为时间戳以和这些系统兼容。 |
| `spark.sql.parquet.cacheMetadata` | true | Turns on caching of Parquet schema metadata. Can speed up querying of static data. |
| `spark.sql.parquet.compression.codec` | snappy | 设置写Parquet文件时使用的压缩codec。可用参数包括uncompressed, snappy, gzip, lzo。 |
| `spark.sql.parquet.filterPushdown` | true | 设置为true时，开启Parquet过滤器push-down优化。 |
| `spark.sql.hive.convertMetastoreParquet` | true | 设置为false时, Spark SQL 会为parquet表使用 Hive SerDer而不是内置的支持。 |
| `spark.sql.parquet.mergeSchema` | false | 设置为true时，Parquet数据源合并从全部数据文件收集的schema，否则从汇总文件（如果没有可用的汇总文件，则选择一个随机的数据文件）获取schema。 |
| `spark.sql.optimizer.metadataOnly` | true | 设置为true时，开启metadata-only查询优化（使用表的metadata来产生分区字段，而不是通过扫表产生）。当扫描的字段都是分区字段并且查询中有一个符合distinct语义的聚合操作符时它才起作用。 |

#### 3、JSON数据集

Spark SQL可以自动推测JSON数据集的schema，并且把它加载为`Dataset[Row]`。对JSON文件或者`Dataset[String]`应用`SparkSession.read.json()`即可实现这个转换。

需要注意的是，这里的json文件**并不是**典型的JSON文件。每行都必须包含一个单独的（separate）、独立有效的（self-contained valid）JSON对象。对于常规的多行JSON文件，需要通过`option`将`multiLine`选项设置为`true`。

```scala
// Primitive types (Int, String, etc) and Product types (case classes) encoders are
// supported by importing this when creating a Dataset.
import spark.implicits._

// A JSON dataset is pointed to by path.
// The path can be either a single text file or a directory storing text files
val path = "examples/src/main/resources/people.json"
val peopleDF = spark.read.json(path)

// The inferred schema can be visualized using the printSchema() method
peopleDF.printSchema()
// root
//  |-- age: long (nullable = true)
//  |-- name: string (nullable = true)

// Creates a temporary view using the DataFrame
peopleDF.createOrReplaceTempView("people")

// SQL statements can be run by using the sql methods provided by spark
val teenagerNamesDF = spark.sql("SELECT name FROM people WHERE age BETWEEN 13 AND 19")
teenagerNamesDF.show()
// +------+
// |  name|
// +------+
// |Justin|
// +------+

// Alternatively, a DataFrame can be created for a JSON dataset represented by
// a Dataset[String] storing one JSON object per string
val otherPeopleDataset = spark.createDataset(
  """{"name":"Yin","address":{"city":"Columbus","state":"Ohio"}}""" :: Nil)
val otherPeople = spark.read.json(otherPeopleDataset)
otherPeople.show()
// +---------------+----+
// |        address|name|
// +---------------+----+
// |[Columbus,Ohio]| Yin|
// +---------------+----+
```

#### 4、Hive表

Spark SQL也支持对Apache Hive中的表进行读写。但是Hive有大量的依赖，而这些依赖不存在于Spark默认的发行版本中。如果classpath中有这些Hive依赖，Spark会自动地加载它们。需要注意的是，这些Hive依赖必须存在于所有的工作节点上，因为为了访问Hive中保存的数据，需要访问Hive的序列化和反序列化库（SerDes）。

把`hive-site.xml`, `core-site.xml` \(for security configuration\),和 `hdfs-site.xml` \(for HDFS configuration\) 文件放在`/conf`目录中完成Hive的配置。

使用Hive时，必须实例化支持Hive的`SparkSession`，包括到Hive metastore的连接、Hive serdes的支持、和Hive的用户自定义函数。没有部署Hive的情况下也可以开启Hive支持。没有通过`hive-site.xml`进行配置的情况下，Spark的context会自动地在当前目录中创建`metastore_db`，并且创建一个`spark.sql.warehouse.dir`配置的目录（默认的目录是Spark应用启动时的当前目录中的`spark-warehouse`目录）。从Spark 2.0.0开始，`hive-site.xml` 中的`hive.metastore.warehouse.dir` 属性就被弃用了，使用`spark.sql.warehouse.dir` 来指定仓库中的数据库的默认位置。可能需要为启动Spark应用的用户授权写权限。

```scala
import java.io.File

import org.apache.spark.sql.Row
import org.apache.spark.sql.SparkSession

case class Record(key: Int, value: String)

// warehouseLocation points to the default location for managed databases and tables
val warehouseLocation = new File("spark-warehouse").getAbsolutePath

val spark = SparkSession
  .builder()
  .appName("Spark Hive Example")
  .config("spark.sql.warehouse.dir", warehouseLocation)
  .enableHiveSupport()
  .getOrCreate()

import spark.implicits._
import spark.sql

sql("CREATE TABLE IF NOT EXISTS src (key INT, value STRING) USING hive")
sql("LOAD DATA LOCAL INPATH 'examples/src/main/resources/kv1.txt' INTO TABLE src")

// Queries are expressed in HiveQL
sql("SELECT * FROM src").show()
// +---+-------+
// |key|  value|
// +---+-------+
// |238|val_238|
// | 86| val_86|
// |311|val_311|
// ...

// Aggregation queries are also supported.
sql("SELECT COUNT(*) FROM src").show()
// +--------+
// |count(1)|
// +--------+
// |    500 |
// +--------+

// The results of SQL queries are themselves DataFrames and support all normal functions.
val sqlDF = sql("SELECT key, value FROM src WHERE key < 10 ORDER BY key")

// The items in DataFrames are of type Row, which allows you to access each column by ordinal.
val stringsDS = sqlDF.map {
  case Row(key: Int, value: String) => s"Key: $key, Value: $value"
}
stringsDS.show()
// +--------------------+
// |               value|
// +--------------------+
// |Key: 0, Value: val_0|
// |Key: 0, Value: val_0|
// |Key: 0, Value: val_0|
// ...

// You can also use DataFrames to create temporary views within a SparkSession.
val recordsDF = spark.createDataFrame((1 to 100).map(i => Record(i, s"val_$i")))
recordsDF.createOrReplaceTempView("records")

// Queries can then join DataFrame data with data stored in Hive.
sql("SELECT * FROM records r JOIN src s ON r.key = s.key").show()
// +---+------+---+------+
// |key| value|key| value|
// +---+------+---+------+
// |  2| val_2|  2| val_2|
// |  4| val_4|  4| val_4|
// |  5| val_5|  5| val_5|
// ...
```

##### 4.1、指定Hive表的存储格式

默认，以普通文本的格式读取Hive表的数据。如下选项可以用来指定Hive的存储格式，比如 `CREATE TABLE src(id int) USING hive OPTIONS(fileFormat 'parquet')`。注意，Spark SQL还不支持创建表时对Hive存储格式进行处理，可以在Hive侧使用存储格式创建表，然后使用Spark SQL来读取它。

| Property Name | Meaning |
| --- | --- |
| `fileFormat` | 指定文件存储格式，它包含了 "serde", "input format"和 "output format"信息。目前支持6中格式: 'sequencefile', 'rcfile', 'orc', 'parquet', 'textfile' and 'avro'。 |
| `inputFormat, outputFormat` | 指定对应 `InputFormat` and `OutputFormat`类的名称字符串，比如 `org.apache.hadoop.hive.ql.io.orc.OrcInputFormat`. 这两个选项成对出现，如果指定了 `fileFormat` 选项，就不能这两个选项。 |
| `serde` | 指定serde class的名字。如果指定的`fileformat`包含了serde的信息，就不想再指定这个选项。目前，"sequencefile", "textfile" 和 "rcfile"这三种文件格式不包含serde信息，所以它们需要指定该选项。 |
| `fieldDelim, escapeDelim, collectionDelim, mapkeyDelim, lineDelim` | 这些选项只能用于“textfile”文件格式。定义如何把分隔符分隔的文件读取为行。 |

所有这些属性都通过`OPTIONS`定义，会被当作Hive serde属性。

##### 4.2、与不同版本的Hive metastore交互

Spark SQL的Hive支持中有些很重要部分是与Hive metastore的交互，它们使Spark SQL可以访问Hive表的metadata。从Spark 1.4.0开始，使用如下配置，Spark SQL可以查询不同版本的Hive metastores。注意，与和metastore交互Hive的版本不同，Spark SQL内部对照Hive 1.2.1编译并且把这些类用于内部执行（serdes、UDFs、UDAFs，等）。

如下选项可以用于配置用来获取metadata的Hive版本：

| Property Name | Default | Meaning |
| --- | --- | --- |
| `spark.sql.hive.metastore.version` | `1.2.1` | Hive metastore版本.可选的是 `0.12.0` t到 `1.2.1`的版本。 |
| `spark.sql.hive.metastore.jars` | `builtin` | 用来实例化HiveMetastoreClient的jars的位置。有3个选项: 1） `builtin`         使用Hive 1.2.1, 当`-Phive`被开启时，它和Spark assembly 是绑定的. 使用这个选项时，`spark.sql.hive.metastore.version` 必须是 `1.2.1` 或者未定义的。2） `maven`      使用从Maven仓库下载的指定版本的Hive jars。通常不建议在生产环境使用这个配置。3）JVM标准格式的classpath    这个classpath必须包含整个Hive和Hive的依赖，包括正确版本的Hadoop。这些jars只需要存在于driver上，但是，如果要在yarn集群模式运行，必须确保它们打包在应用中。 |
| `spark.sql.hive.metastore.sharedPrefixes` | `com.mysql.jdbc`， `org.postgresql`， `com.microsoft.sqlserver`，  `oracle.jdbc` | 逗号分隔的，需要使用classloader加载的，Spark SQL和某个指定版本的Hive之间共享的，类的前缀的集合。需要共享的类的一个例子是与metastore进行交流的JDBC drivers。其它需要共享是那些和已经共享的类进行交互的类。比如，log4j使用的自定义的appenders。 |
| `spark.sql.hive.metastore.barrierPrefixes` | `(empty)` | 逗号分隔的，需要明确地为和Spark SQL通信的每个Hive版本重新加载的，类的前缀的集合。比如，以同一个前缀声明的需要共享的Hive UDFs（比如，`org.apache.spark.*`）。 |

#### 5、其它数据库的 JDBC

Spark SQL 也包括有一个使用 JDBC 从其它数据库读取数据的数据源。这个功能比使用 `JdbcRDD` 更好。它返回的结果是 DataFrame，进而并且可以轻松地被 Spark SQL 处理或者和其它数据源进行连接。JDBC 数据源不需要用户提供 ClassTag，使用 Java 或者 Python 操作起来更简单。（注意 JDBC 数据源和允许其他应用使用 Spark SQL 运行查询的 Spark SQL JDBC server 是不同的。）

使用 jdbc 数据源需要 spark classpath 中有与数据对应的 JDBC driver。比如，要通过spark-shell连接到 postgres，需要运行如下命令：

```bash
bin/spark-shell --driver-class-path postgresql-9.4.1207.jar --jars postgresql-9.4.1207.jar
```

使用JDBC数据源API可以把远程的数据库加载为一个DataFrame或者Spark SQL临时视图。在数据源的选项中可以指定JDBC连接属性。`user` 和 `password` 一般作为连接属性来提供，以登录到数据源。除了连接属性，Spark还支持如下的不区分大小写的选项：

| Property Name | Meaning |
| --- | --- |
| `url` | JDBC URL。不同数据源可能不同。比如, `jdbc:postgresql://localhost/test?user=fred&password=secret` |
| `dbtable` | 表名。可以是SQL查询的`FROM`子句中合法的所有语句。比如，除了表名，还可以使用括号中的子查询。 |
| `driver` | JDBC driver类名 |
| `partitionColumn, lowerBound, upperBound` | 这几个属性必须同时指定或者都不指定。指定时，`numPartitions`也必须同时指定。这些选项描述从多个工作者并行读取数据时如何对表进行分区。`partitionColumn`必须是表的数值类型列。注意，`lowerBound`和`upperBound` 只是用来决定分区的步长，不是过滤表中的记录。所有的记录都会被分区然后返回。这个选项只用于读取数据。 |
| `numPartitions` | 表并行读写使用的最大的分区数量。也决定了并行的JDBC连接的最大数量。如果要写入的数据集的分区数超过这个限制，Spark会在写数据前调用 `coalesce(numPartitions)`。 |
| `fetchsize` | JDBC抓取大小，决定一次性获取的记录条数。这个选项有助于提升抓取大小比较小的（比如，Oracel是10）JDBC driver的性能。仅适用于读。 |
| `batchsize` | JDBC批次大小，决定一次插入的记录条数。这个选项有助于提升JDBC drivers的性能。仅适用于写。默认，1000。 |
| `isolationLevel` | 事务独立级别，仅适用于当前连接。可以是 `NONE`, `READ_COMMITTED`, `READ_UNCOMMITTED`, `REPEATABLE_READ`, 或者 `SERIALIZABLE`, 对应于 JDBC的Connection对象的标准事独立性级别，默认的是`READ_UNCOMMITTED`。仅适用于写。具体查阅`java.sql.Connection`文档。 |
| `truncate` | 这是一个JDBC写相关选项。当开启 `SaveMode.Overwrite` 时，这个选项让Spark truncate已经存在的表，而不是先drop表然后再create表。这更加高效，并且防止表元数据（比如，索引）被删除。但是，某些情况下不起作用，比如，当新的数据schema不同时。默认`false`，只适用于写。 |
| `createTableOptions` | 这是一个JDBC写相关的选项。如果指定，当创建表时这个选项允许设置数据库专用的表或分区选项 \(比如, `CREATE TABLE t (name string) ENGINE=InnoDB.`\)。仅适用于写。 |
| `createTableColumnTypes` | 创建表时，用于替换默认列类型的数据库列数据类型。数据类型信息的指定应该与CREATE TABLE语法格式相同（比如： `"name CHAR(64), comments VARCHAR(1024)"`）。指定的类型应该是合法的Spark SQL数据类型。仅适用于写。 |

```scala
// Note: JDBC loading and saving can be achieved via either the load/save or jdbc methods
// Loading data from a JDBC source
val jdbcDF = spark.read
  .format("jdbc")
  .option("url", "jdbc:postgresql:dbserver")
  .option("dbtable", "schema.tablename")
  .option("user", "username")
  .option("password", "password")
  .load()

val connectionProperties = new Properties()
connectionProperties.put("user", "username")
connectionProperties.put("password", "password")
val jdbcDF2 = spark.read
  .jdbc("jdbc:postgresql:dbserver", "schema.tablename", connectionProperties)

// Saving data to a JDBC source
jdbcDF.write
  .format("jdbc")
  .option("url", "jdbc:postgresql:dbserver")
  .option("dbtable", "schema.tablename")
  .option("user", "username")
  .option("password", "password")
  .save()

jdbcDF2.write
  .jdbc("jdbc:postgresql:dbserver", "schema.tablename", connectionProperties)

// Specifying create table column data types on write
jdbcDF.write
  .option("createTableColumnTypes", "name CHAR(64), comments VARCHAR(1024)")
  .jdbc("jdbc:postgresql:dbserver", "schema.tablename", connectionProperties)
```

**使用 Spark JDBC 时需要[注意](https://c2fo.github.io/spark/jdbc/data-partitioning/db/io/2022/12/01/apache-spark-a-note-when-using-jdbc-partitioning/)**：spark 通过 jdbc 并行读取数据（多分区）就是通过多个独立的 jdbc 连接去读取数据库。指定了 `numPartitions` 时也需要指定 `partitionColumn, lowerBound, upperBound` 三个属性才能生效，否则读取结果都是只有一个分区。`partitionColumn, lowerBound, upperBound` 三个属性结合 `numPartitions` 属性决定了每个独立（并行）查询的步长：

```scala
val stride: Long = upperBound / numPartitions - lowerBound / numPartitions
```

另，JDBC 写数据时，如果多分区（并行）写数据，可能会遇到多个并发连接同时操作数据库的情况，可能出现 `lock write timeout` 异常，此时如果能保证不出错，可以设置 `isolationLevel` 选项为 `NONE` 关闭事务，从而提高写数据的效率。

如果写的**数据量很大，频率很高，可能会对数据库造成较大压力**，还会有存储压力（MySQL Binary Log 会激增）。

#### 6、故障排除

* JDBC driver必须对客户的session和所有executors上的原始类加载器（primordial class loader）可见。因为在打开connection时，Java的DriverManager会进行安全检查，它会忽略对原始类加载器不可见的所有drivers。一种简单的方法是，修改所有工作者节点上的`compute_classpath.sh`以包含driver JARs。
* 某些数据库，比如H2，把所有名称转换为大写。在Spark SQL中要使用大写来指向这些名称。

#### 自定义数据源

对于Spark SQL没有内置的支持的数据源，可以通过自定义数据源来实现自定义的支持。



