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

#### `split`函数——换行

文本里面包含换行，也会在换行处进行切分，不管切分的pattern是否是换行符。奇怪……hive、和spark sql都是这样……

#### `json_tuple`返回值的类型

如果`json_tuple(col, field)`，`col`json字符串的`field`属性是一个字符串数组，`json_tuple`执行的结果类型是元组。`select`子句中改为`json_tuple(col, field).as("fname")`，`fname`的类型好像就成了string。但是使用`regexp_replace(json_tuple(col, field), "asd", "")`时，报错：

```
User class threw exception: org.apache.spark.sql.AnalysisException: cannot resolve 'regexp_replace(json_tuple(facet_logs_history.`eventcontent`, 'ctr'), '^\\[|]$|\\s', '')' due to data type mismatch: argument 1 requires string type, however, 'json_tuple(facet_logs_history.`eventcontent`, 'ctr')' is of array<struct<c0:string>> type
```

Spark将其识别为数组`array<struct<c0:string>>`。是Spark对类型进行了隐式转换吗？

