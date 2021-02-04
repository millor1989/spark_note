#### 1、入口：SparkSession

Spark的所有功能的入口是`SparkSession`类。使用`SparkSession.builder()`创建：

```scala
import org.apache.spark.sql.SparkSession

val spark = SparkSession
  .builder()
  .appName("Spark SQL basic example")
  .config("spark.some.config.option", "some-value")
  .getOrCreate()

// For implicit conversions like converting RDDs to DataFrames
import spark.implicits._
```

Spark 2.0的`SparkSession`提供了对于Hive特征（包括使用HiveQL写查询语句的能力，访问Hive的UDFs，从Hive表读取数据的能力）的内部支持。不用进行Hive设置就能使用这些特征。

#### 2、创建DataFrames

使用`SparkSession`，应用可以从RDDs、Hive表或者其它Spark支持的数据源创建DataFrames。比如，从JSON文件创建DataFrame：

```scala
val df = spark.read.json("examples/src/main/resources/people.json")

// Displays the content of the DataFrame to stdout
df.show()
// +----+-------+
// | age|   name|
// +----+-------+
// |null|Michael|
// |  30|   Andy|
// |  19| Justin|
// +----+-------+
```

#### 3、无类型Dataset Operations（aka DataFrame Operations）

DataFrames提供了Scala、Java、Python和R语言的DSL（domain-specific language，领域专用语言）用于结构化数据的处理。

Spark 2.0的Java和Scala API中，DataFrame就是`Row`s组成的Dataset。与强类型的Scala/Java数据集的“typed transfromations”相反，DataFrame的操作也叫做“untyped transformations”。

```scala
// This import is needed to use the $-notation
import spark.implicits._
// Print the schema in a tree format
df.printSchema()
// root
// |-- age: long (nullable = true)
// |-- name: string (nullable = true)

// Select only the "name" column
df.select("name").show()
// +-------+
// |   name|
// +-------+
// |Michael|
// |   Andy|
// | Justin|
// +-------+

// Select everybody, but increment the age by 1
df.select($"name", $"age" + 1).show()
// +-------+---------+
// |   name|(age + 1)|
// +-------+---------+
// |Michael|     null|
// |   Andy|       31|
// | Justin|       20|
// +-------+---------+

// Select people older than 21
df.filter($"age" > 21).show()
// +---+----+
// |age|name|
// +---+----+
// | 30|Andy|
// +---+----+

// Count people by age
df.groupBy("age").count().show()
// +----+-----+
// | age|count|
// +----+-----+
// |  19|    1|
// |null|    1|
// |  30|    1|
// +----+-----+
```

在[Dataset API 文档](http://spark.apache.org/docs/2.2.0/api/scala/index.html#org.apache.spark.sql.Dataset)可以查看Dataset全部的操作。

另外，除了简单的列引用和表达式，Datasets还有函数库包含了字符串操作、日前计算、常用数学运算等，在[DataFrame Function文档](http://spark.apache.org/docs/2.2.0/api/scala/index.html#org.apache.spark.sql.functions$)中可以查看这些函数。

#### 4、编程执行SQL查询

`SparkSession.sql(...)`

```scala
// Register the DataFrame as a SQL temporary view
df.createOrReplaceTempView("people")

val sqlDF = spark.sql("SELECT * FROM people")
sqlDF.show()
// +----+-------+
// | age|   name|
// +----+-------+
// |null|Michael|
// |  30|   Andy|
// |  19| Justin|
// +----+-------+
```

#### 5、全局的临时视图（Global Temporary View）

Spark临时视图的作用域是Spark session，如果创建临时视图的session结束，临时视图则消失。如果要创建一个所有session共享的临时视图，并且使它在Spark应用结束前一直活跃，那么可以创建一个全局临时视图。全局临时视图被绑定到系统保留数据库`global_temp`，可以使用这个限定名来使用它。

```scala
// Register the DataFrame as a global temporary view
df.createGlobalTempView("people")

// Global temporary view is tied to a system preserved database `global_temp`
spark.sql("SELECT * FROM global_temp.people").show()
// +----+-------+
// | age|   name|
// +----+-------+
// |null|Michael|
// |  30|   Andy|
// |  19| Justin|
// +----+-------+

// Global temporary view is cross-session
spark.newSession().sql("SELECT * FROM global_temp.people").show()
// +----+-------+
// | age|   name|
// +----+-------+
// |null|Michael|
// |  30|   Andy|
// |  19| Justin|
// +----+-------+
```

#### 6、创建Datasets

Datasets与RDDs类似，但是为了对其中的对象进行处理或者通过网络传输，需要使用`Encoder`对对象进行序列化，而不是使用Java序列化或者Kryo。然而，要把对象转换为字节encoders和标准的序列化都是需要的，encoders是动态地生成的代码并且使用一种格式，让Spark可以在不把字节反序列化回对象的情况下，执行许多像过滤、排序和哈希等许多操作。

```scala
// Note: Case classes in Scala 2.10 can support only up to 22 fields. To work around this limit,
// you can use custom classes that implement the Product interface
case class Person(name: String, age: Long)

// Encoders are created for case classes
val caseClassDS = Seq(Person("Andy", 32)).toDS()
caseClassDS.show()
// +----+---+
// |name|age|
// +----+---+
// |Andy| 32|
// +----+---+

// Encoders for most common types are automatically provided by importing spark.implicits._
val primitiveDS = Seq(1, 2, 3).toDS()
primitiveDS.map(_ + 1).collect() // Returns: Array(2, 3, 4)

// DataFrames can be converted to a Dataset by providing a class. Mapping will be done by name
val path = "examples/src/main/resources/people.json"
val peopleDS = spark.read.json(path).as[Person]
peopleDS.show()
// +----+-------+
// | age|   name|
// +----+-------+
// |null|Michael|
// |  30|   Andy|
// |  19| Justin|
// +----+-------+
```

#### 7、与RDDs的互操作（interoperating with RDDs）

Spark SQL支持两种把存在的RDDs转换为Datasets的方法：第一，使用反射来推测包含特定类型对象的RDD的schema；已经知道schema的情况下，在编写Spark应用时，这种基于反射的方法使代码更加简洁。

第二种方法，通过程式化的结构创建schema并应用到存在的RDD上。尽管这种方法更加复杂，但是，在只有运行时才能知道数据类型的情况下，使得构建Datasets成为可能。

##### 7.1、使用反射推测Schema

Spark SQL的Scala接口支持自动地将包含case classes的RDD转换为DataFrame。case class定义了表的schema。使用反射读取case class的参数名并作为列名。case classes可以嵌套或者包含像`Seq`或者`Array`这样的敷在类型。这种RDD可以隐式得转换为DataFrame并且然后被注册为一个表。表可以用在后续的SQL语句中。

```scala
/ For implicit conversions from RDDs to DataFrames
import spark.implicits._

// Create an RDD of Person objects from a text file, convert it to a Dataframe
val peopleDF = spark.sparkContext
  .textFile("examples/src/main/resources/people.txt")
  .map(_.split(","))
  .map(attributes => Person(attributes(0), attributes(1).trim.toInt))
  .toDF()
// Register the DataFrame as a temporary view
peopleDF.createOrReplaceTempView("people")

// SQL statements can be run by using the sql methods provided by Spark
val teenagersDF = spark.sql("SELECT name, age FROM people WHERE age BETWEEN 13 AND 19")

// The columns of a row in the result can be accessed by field index
teenagersDF.map(teenager => "Name: " + teenager(0)).show()
// +------------+
// |       value|
// +------------+
// |Name: Justin|
// +------------+

// or by field name
teenagersDF.map(teenager => "Name: " + teenager.getAs[String]("name")).show()
// +------------+
// |       value|
// +------------+
// |Name: Justin|
// +------------+

// No pre-defined encoders for Dataset[Map[K,V]], define explicitly
implicit val mapEncoder = org.apache.spark.sql.Encoders.kryo[Map[String, Any]]
// Primitive types and case classes can be also defined as
// implicit val stringIntMapEncoder: Encoder[Map[String, Any]] = ExpressionEncoder()

// row.getValuesMap[T] retrieves multiple columns at once into a Map[String, T]
teenagersDF.map(teenager => teenager.getValuesMap[Any](List("name", "age"))).collect()
// Array(Map("name" -> "Justin", "age" -> 19))
```

##### 7.2、程式化地指定Schema

如果不能提前定义case classes（比如，记录的结构编码在一个字符串中，或者要解析文本数据集，并且不同的用户对应不同的字段_for example， the structure of records is encoded in a string，or a text dataset will be parsed and fields will be projected differently for different users_），通过以下三步可以程式化地创建`DataFrame`：

1. 从原始RDD创建一个`Row`s的RDD；
2. 创建用`StructType`表示第一步创建的RDD的`Row`s的结构对应的schema；
3. 通过`SparkSession`提供的`createDataFrame`方法把schema应用到RDD。

```scala
import org.apache.spark.sql.types._

// Create an RDD
val peopleRDD = spark.sparkContext.textFile("examples/src/main/resources/people.txt")

// The schema is encoded in a string
val schemaString = "name age"

// Generate the schema based on the string of schema
val fields = schemaString.split(" ")
  .map(fieldName => StructField(fieldName, StringType, nullable = true))
val schema = StructType(fields)

// Convert records of the RDD (people) to Rows
val rowRDD = peopleRDD
  .map(_.split(","))
  .map(attributes => Row(attributes(0), attributes(1).trim))

// Apply the schema to the RDD
val peopleDF = spark.createDataFrame(rowRDD, schema)

// Creates a temporary view using the DataFrame
peopleDF.createOrReplaceTempView("people")

// SQL can be run over a temporary view created using DataFrames
val results = spark.sql("SELECT name FROM people")

// The results of SQL queries are DataFrames and support all the normal RDD operations
// The columns of a row in the result can be accessed by field index or by field name
results.map(attributes => "Name: " + attributes(0)).show()
// +-------------+
// |        value|
// +-------------+
// |Name: Michael|
// |   Name: Andy|
// | Name: Justin|
// +-------------+
```

#### 8、聚合

[内置的DataFrame函数](http://spark.apache.org/docs/2.2.0/api/scala/index.html#org.apache.spark.sql.functions$)提供了像 `count()`, `countDistinct()`, `avg()`, `max()`, `min()`, 等等的通用聚合函数。但是这些函数是为DataFrames设计的，Spark SQL的Scala和Java API还有一些类型安全（type-safe）版本内置函数可以用于强类型的Datasets。另外，除了预定义的聚合函数，用户还可以自定义聚合函数。

##### 8.1、无类型用户定义聚合函数（Untyped User-Defined Aggregate Functions）

用户必须继承[UserDefinedAggregateFunction](http://spark.apache.org/docs/2.2.0/api/scala/index.html#org.apache.spark.sql.expressions.UserDefinedAggregateFunction)这个抽象类来实现一个自定义的无类型聚合函数。比如，可以定义一个求平均值得函数如下：

```scala
import org.apache.spark.sql.expressions.MutableAggregationBuffer
import org.apache.spark.sql.expressions.UserDefinedAggregateFunction
import org.apache.spark.sql.types._
import org.apache.spark.sql.Row
import org.apache.spark.sql.SparkSession

object MyAverage extends UserDefinedAggregateFunction {
  // Data types of input arguments of this aggregate function
  def inputSchema: StructType = StructType(StructField("inputColumn", LongType) :: Nil)
  // Data types of values in the aggregation buffer
  def bufferSchema: StructType = {
    StructType(StructField("sum", LongType) :: StructField("count", LongType) :: Nil)
  }
  // The data type of the returned value
  def dataType: DataType = DoubleType
  // Whether this function always returns the same output on the identical input
  def deterministic: Boolean = true
  // Initializes the given aggregation buffer. The buffer itself is a `Row` that in addition to
  // standard methods like retrieving a value at an index (e.g., get(), getBoolean()), provides
  // the opportunity to update its values. Note that arrays and maps inside the buffer are still
  // immutable.
  def initialize(buffer: MutableAggregationBuffer): Unit = {
    buffer(0) = 0L
    buffer(1) = 0L
  }
  // Updates the given aggregation buffer `buffer` with new input data from `input`
  def update(buffer: MutableAggregationBuffer, input: Row): Unit = {
    if (!input.isNullAt(0)) {
      buffer(0) = buffer.getLong(0) + input.getLong(0)
      buffer(1) = buffer.getLong(1) + 1
    }
  }
  // Merges two aggregation buffers and stores the updated buffer values back to `buffer1`
  def merge(buffer1: MutableAggregationBuffer, buffer2: Row): Unit = {
    buffer1(0) = buffer1.getLong(0) + buffer2.getLong(0)
    buffer1(1) = buffer1.getLong(1) + buffer2.getLong(1)
  }
  // Calculates the final result
  def evaluate(buffer: Row): Double = buffer.getLong(0).toDouble / buffer.getLong(1)
}

// Register the function to access it
spark.udf.register("myAverage", MyAverage)

val df = spark.read.json("examples/src/main/resources/employees.json")
df.createOrReplaceTempView("employees")
df.show()
// +-------+------+
// |   name|salary|
// +-------+------+
// |Michael|  3000|
// |   Andy|  4500|
// | Justin|  3500|
// |  Berta|  4000|
// +-------+------+

val result = spark.sql("SELECT myAverage(salary) as average_salary FROM employees")
result.show()
// +--------------+
// |average_salary|
// +--------------+
// |        3750.0|
// +--------------+
```

##### 8.2、类型安全的用户定义聚合函数（Type-Safe User-Defined Aggregate Functions）

强类型Datasets的用户定义聚合函数要继承[Aggregator](http://spark.apache.org/docs/2.2.0/api/scala/index.html#org.apache.spark.sql.expressions.Aggregator)抽象类。比如，可以定义一个类型安全的用户定义求平均值得函数如下：

```scala
import org.apache.spark.sql.expressions.Aggregator
import org.apache.spark.sql.Encoder
import org.apache.spark.sql.Encoders
import org.apache.spark.sql.SparkSession

case class Employee(name: String, salary: Long)
case class Average(var sum: Long, var count: Long)

object MyAverage extends Aggregator[Employee, Average, Double] {
  // A zero value for this aggregation. Should satisfy the property that any b + zero = b
  def zero: Average = Average(0L, 0L)
  // Combine two values to produce a new value. For performance, the function may modify `buffer`
  // and return it instead of constructing a new object
  def reduce(buffer: Average, employee: Employee): Average = {
    buffer.sum += employee.salary
    buffer.count += 1
    buffer
  }
  // Merge two intermediate values
  def merge(b1: Average, b2: Average): Average = {
    b1.sum += b2.sum
    b1.count += b2.count
    b1
  }
  // Transform the output of the reduction
  def finish(reduction: Average): Double = reduction.sum.toDouble / reduction.count
  // Specifies the Encoder for the intermediate value type
  def bufferEncoder: Encoder[Average] = Encoders.product
  // Specifies the Encoder for the final output value type
  def outputEncoder: Encoder[Double] = Encoders.scalaDouble
}

val ds = spark.read.json("examples/src/main/resources/employees.json").as[Employee]
ds.show()
// +-------+------+
// |   name|salary|
// +-------+------+
// |Michael|  3000|
// |   Andy|  4500|
// | Justin|  3500|
// |  Berta|  4000|
// +-------+------+

// Convert the function to a `TypedColumn` and give it a name
val averageSalary = MyAverage.toColumn.name("average_salary")
val result = ds.select(averageSalary)
result.show()
// +--------------+
// |average_salary|
// +--------------+
// |        3750.0|
// +--------------+
```



