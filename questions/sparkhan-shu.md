### `Column`的`isNaN`方法，和Spark的内置函数`isnan(col:Column )`

如果是NaN则返回True，否则返回False。

但是，这两个方法需要目标列是DOUBLE类型，如果不是DOUBLE类型会先进行强制转换。

### `NULL`不是NaN，不是NaN！！！

### `nanvl(col1: Column, col2: Column)`，`col1`为字符串且为`NULL`时，返回结果是`NULL`，竟然不是`col2`；

```
  /**
   * Returns col1 if it is not NaN, or col2 if col1 is NaN.
   *
   * Both inputs should be floating point columns (DoubleType or FloatType).
   *
   * @group normal_funcs
   * @since 1.5.0
   */
  def nanvl(col1: Column, col2: Column): Column = withExpr { NaNvl(col1.expr, col2.expr) }
```

看函数的描述，两个参数都应该是**浮点类型**的列。

从spark-shell的执行结果来看，`nanvl`会自动执行非浮点值到`Double`的强制转换，但是对于`NULL`返回的结果是`NULL`。

![](/assets/1589872388084.png)

如果要达到`col1`为`NULL`时返回`col2`的目的，可以使用`coalesce(e: Column*)`函数，或者使用hive SQL的`NVL(col1,col2)`替换`nanvl`。

#### `split` 函数——换行

文本里面包含换行，也会在换行处进行切分，不管切分的pattern是否是换行符。奇怪……hive、和spark sql都是这样……

#### `json_tuple`返回值的类型

如果`json_tuple(col, field)`，`col`json字符串的`field`属性是一个字符串数组，`json_tuple`执行的结果类型是元组。`select`子句中改为`json_tuple(col, field).as("fname")`，`fname`的类型好像就成了string。但是使用`regexp_replace(json_tuple(col, field), "asd", "")`时，报错：

```
User class threw exception: org.apache.spark.sql.AnalysisException: cannot resolve 'regexp_replace(json_tuple(facet_logs_history.`eventcontent`, 'ctr'), '^\\[|]$|\\s', '')' due to data type mismatch: argument 1 requires string type, however, 'json_tuple(facet_logs_history.`eventcontent`, 'ctr')' is of array<struct<c0:string>> type
```

Spark将其识别为数组`array<struct<c0:string>>`。是Spark对类型进行了隐式转换吗？

#### collect_set、collect_list

如果分组元素都是 `null`，它们返回的结果是空的集合，而不是 `null`。

#### 如果函数的返回结果不是一个单纯的列，那么不能对函数结果直接进行解析等操作

否则会报错 ：

```
Generators are not supported when it's nested in expressions
```

如下代码会报错，因为 `json_tuple` 返回的不是单纯的一列，可能是多列：

```scala
df.select(
    from_json(
        json_tuple('value.cast("string"), "data").as("data"),
        msgDataSchema
    ).as("data")
)
```

具体报错为：

```
org.apache.spark.sql.AnalysisException: Generators are not supported when it's nested in expressions, but got: jsontostructs(json_tuple(CAST(value AS STRING), data
) AS `data`);
```

多一层`select`处理后就可以正常执行了：

```
df.select(json_tuple('value.cast("string"), "data").as("data"))
      .select(from_json('data, msgDataSchema).as("data"))
```

类似 `json_tuple`，`explode` 也可能引发这个错。

#### 可以使用 `from_json` 解析 json 字符串 `{k1:v1,k2:v2,k11:v11,k3:v3,...}` 为 map

`from_json` 无法直接解析 `MapType` 类型数据。需要将其转化为 `StructType`。

```
df.select('device_id,
        explode(
          from_json(
            concat(lit("{\"k\":"), 'kws, lit("}")),
            StructType(Seq(StructField("k", MapType(StringType, DoubleType))))
          ).getField("k")
        )
      )
```

解析后通过  `getField("k")` 即可将字符串读取为 `MapType` 的列。通过 `explode` 函数，便可以将 map 解析为名为 `key` 和 `value` 的两列数据。

#### `explode` 函数在遇到 `NULL` 值得时候，会将其过滤掉。

如果想要保留 `NULL` 值对应的记录，可以使用函数 `explode_outer`。