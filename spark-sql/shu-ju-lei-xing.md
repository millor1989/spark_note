### 数据类型

Spark SQL和DataFrames支持如下的数据类型：

* 数值类型
  * `ByteType`：表示1字节有符号的整数。范围`-128` to `127`。
  * `ShortType`：表示2字节有符号的整数。范围`-32768` to `32767`。
  * `IntegerType`：表示4字节有符号的整数。范围`-2147483648` to `2147483647`。
  * `LongType`：表示8字节有符号的整数。范围`-9223372036854775808` to `9223372036854775807`。
  * `FloatType`：表示4字节单精度浮点数值。
  * `DoubleType`：表示8字节双精度浮点数值。
  * `DecimalType`：表示任意精度的有符号的小数。内部实现是`java.math.BigDecimal`。一个`BigDecimal`由一个任意精度的整数非标度值和一个32位（32-bit）整数标度值构成。
* 字符串类型
  * `StringType`：表示字符串值。
* 二进制类型
  * `BinaryType`：表示字节序列值。
* 布尔类型
  * `BooleanType`：表示布尔值。
* 日期时间类型
  * `TimestampType`：表示由年、月、日、时、分、秒构成的日期时间。
  * `DateType`：表示由年、月、日构成的日期。
* 复杂类型
  * `ArrayType(elementType, containsNull)`：表示由`elementType`类型的元素序列构成的数组。 `containsNull` 表示`ArrayType`中的元素是否可以有`null`值。
  * `MapType(keyType, valueType, valueContainsNull)`：表示一组键值对构成的值。`keyType`
     表示key的数据类型，`valueType`表示value的数据类型。对于`MapType`值，key不能是`null`。`valueContainsNull`用来表示`MapType`的value是否可以是`null`。
  * `StructType(fields)`：表示结构为`StructField`s \(`fields`\)的元素序列组成的值。
    * `StructField(name, dataType, nullable)`：表示`StructType`的一个属性。属性名称为`name`。属性类型为`dataType`。 `nullable`表示属性值是否可以是`null`。

Spark SQL所有的数据类型都在`org.apache.spark.sql.types`包中。

| Data type         | Value type in Scala                                          | API to access or create a data type                          |
| ----------------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| **ByteType**      | Byte                                                         | ByteType                                                     |
| **ShortType**     | Short                                                        | ShortType                                                    |
| **IntegerType**   | Int                                                          | IntegerType                                                  |
| **LongType**      | Long                                                         | LongType                                                     |
| **FloatType**     | Float                                                        | FloatType                                                    |
| **DoubleType**    | Double                                                       | DoubleType                                                   |
| **DecimalType**   | java.math.BigDecimal                                         | DecimalType                                                  |
| **StringType**    | String                                                       | StringType                                                   |
| **BinaryType**    | Array[Byte]                                                  | BinaryType                                                   |
| **BooleanType**   | Boolean                                                      | BooleanType                                                  |
| **TimestampType** | java.sql.Timestamp                                           | TimestampType                                                |
| **DateType**      | java.sql.Date                                                | DateType                                                     |
| **ArrayType**     | scala.collection.Seq                                         | ArrayType(*elementType*, [*containsNull*])  **Note:**默认的*containsNull* 是 *true*。 |
| **MapType**       | scala.collection.Map                                         | MapType(*keyType*, *valueType*, [*valueContainsNull*])    **Note:** 默认的*valueContainsNull* 是*true*。 |
| **StructType**    | org.apache.spark.sql.Row                                     | StructType(*fields*)    **Note:** *fields*是一个StructFields组成的Seq。不能有两个同名属性。 |
| **StructField**   | The value type in Scala of the data type of this field   (For example, Int for a StructField with the data type IntegerType) | StructField(*name*, *dataType*, [*nullable*])    **Note:** 默认的*nullable* 是*true*。 |

#### NaN语义

在处理`float`或者`double`类型时，对于NaN（not-a-number，不匹配标准浮点语义的值）有专门的处理，特别是：

- NaN = NaN，结果是`true`；
- 在聚合中，所有的NaN被分为一组；
- join的keys中，NaN被作为一个普通的值；
- 在升序排序中，NaN排在最后，比任何其它数值都大。

