#### Spark SQL DataFrame 中的 `array` 类型对应的是 Scala 中的 `scala.collection.mutable.WrappedArray` 类型[#](https://blog.csdn.net/weixin_43648241/article/details/108657619)

在使用 `udf` 时，如果 `array` 字段参数类型设置为 `Array` 而不是 `scala.collection.mutable.WrappedArray`，会抛出如下异常：

```
java.lang.ClassCastException: scala.collection.mutable.WrappedArray$ofRef cannot be cast to [Ljava.lang.String;
```

Scala `Array` 转换为 `scala.collection.mutable.WrappedArray` 的途径是使用 `wrapRefArray` 方法：

```scala
wrapRefArray(Array[String] ....)
```

#### Spark SQL DataFrame 的 `struct` 类型对应的是 Scala 中的二维元组：

```scala
/**
  * 自定义函数——将字符串数组转换为 struct<key:String, value:String> 数组
  */
def kvStruct1: UserDefinedFunction = udf((x: mutable.WrappedArray[String]) => x.map(kvStr => {
  val kv = kvStr.split(":")
  // 返回二维元组即可
  (kv(0), kv(1))
}))

scala> val df = spark.createDataset(Seq((1,"川崎:9.8,奥古斯塔:1.4,QJMOTOR:1.4,杜卡迪:1.4,本田WING:0.7"))).toDF("id","str")

scala> df.select(kvStruct1(split('str, ",")).as("ary2map")).printSchema
root
 |-- ary2map: array (nullable = true)
 |    |-- element: struct (containsNull = true)
 |    |    |-- _1: string (nullable = true)
 |    |    |-- _2: string (nullable = true)

scala> df.select(kvStruct1(split('str, ",")).as("ary2map")).show(false)
+------------------------------------------------------------------------------+
|ary2map                                                                       |
+------------------------------------------------------------------------------+
|[[川崎, 9.8], [奥古斯塔, 1.4], [QJMOTOR, 1.4], [杜卡迪, 1.4], [本田WING, 0.7]]|
+------------------------------------------------------------------------------+


scala> df.select(map_from_entries(kvStruct1(split('str, ",")).as("ary2map"))).show(false)
+------------------------------------------------------------------------------+
|map_from_entries(UDF(split(str, ,)) AS `ary2map`)                             |
+------------------------------------------------------------------------------+
|[川崎 -> 9.8, 奥古斯塔 -> 1.4, QJMOTOR -> 1.4, 杜卡迪 -> 1.4, 本田WING -> 0.7]|
+------------------------------------------------------------------------------+
```

#### Spark SQL DataFrame 的 `map` 类型对应的是 Scala 中的 `Map`

