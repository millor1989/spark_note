### 1、Spark 读小文件

对 `spark.read.text(...)` 之类直接读小文件起作用的参数：

- `spark.files.openCostInBytes` ：默认值 4MiB；官方文档建议设置为比较小的值，尚硅谷优化教程推荐设置为与小文件大小差不多。

- `spark.files.maxPartitionBytes`：默认值 128MiB；读取文件时，一个分区读取的最大的文件总字节数

奇怪的是**尝试改变这两个参数竟然似乎不影响读取文件的分区数**。

对 `spark.sql(...)` 读小文件起作用的参数，与 `spark.read.xxx()` 类似，但是只对读 parquet、json、orc 文件起作用：

- `spark.sql.files.openCostInBytes` 

- `spark.sql.files.maxPartitionBytes`

也许只要用对就行，不需要刻意调整这两个参数的大小。