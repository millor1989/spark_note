### 数据分析

Spark提供了丰富的 API 来进行数据分析。

Spark SQL 的内置函数中有丰富的统计函数（`avg`、`max`、`min`、`stddev`等），也有支持窗口操作的窗口函数（`row_number`、`rank`、`lag`、`lead`等）；还可以通过限定窗口（Window）范围（窗口的`rowsBetween`、`rangeBetween`）进行分析。

Dataset 的 `describe()` 方法，类似于Pandas DataFrame 的 `desc`，可以查看数据集的一些摘要（Summary）信息。`describe()` 底层调用的是 Dataset 的`summary` 方法，`summary` 可以计算数据集的最大\最小值、数量、标准差、百分位数（四分之一位数、中位数等）。这些摘要统计是近似值。需要精确值则须要使用Spark SQL的相应内置函数实现，有些内置的统计函数也分抽样（sample）统计和总体（population）统计两种。