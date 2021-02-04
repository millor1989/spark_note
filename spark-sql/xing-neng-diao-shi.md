### 性能调试

通过把数据在内存中缓存，或者调试一些试验性的选项，有可能会提升某些工作的性能。

#### 把数据缓存到内存

Spark SQL可以使用`spark.catalog.cacheTable("tableName")` 或者 `dataFrame.cache()`，以在内存的列式格式（in-memory columnar foramt），把表进行缓存。然后，Spark SQL会只扫描需要的列，并且会自动地调试压缩以最小化内存使用和GC压力。可以使用`spark.catalog.uncacheTable("tableName")` 把表从内存中移除。

通过使用`SparkSession`的`setConf`，或者使用SQL运行`SET key=value`命令，可以对在内存缓存进行配置。

| Property Name                                  | Default | Meaning                                                      |
| ---------------------------------------------- | ------- | ------------------------------------------------------------ |
| `spark.sql.inMemoryColumnarStorage.compressed` | true    | 当设置为true，Spark SQL会自动地基于数据统计，为每一列选择一个压缩codec。 |
| `spark.sql.inMemoryColumnarStorage.batchSize`  | 10000   | 控制列式缓存的批次大小（size of batches）。较大的批次大小会提升内存的使用率和压缩，但是缓存数据时有OOM的风险。 |

#### 其它配置选项

如下选项也能用来调试查询执行的性能。因为越来越多的优化将会自动执行，以后的Spark版本中这些选项可能会被废弃。

| Property Name                          | Default            | Meaning                                                      |
| -------------------------------------- | ------------------ | ------------------------------------------------------------ |
| `spark.sql.files.maxPartitionBytes`    | 134217728 (128 MB) | 读取文件时，打包到一个分区中的最大字节数量。                 |
| `spark.sql.files.openCostInBytes`      | 4194304 (4 MB)     | 打开一个文件的预估开销，用同一时间可以扫描的字节数来衡量。用于把多个文件放进同一个分区的时候。估计值较大时比较好，此时对于较小的文件会比较大的文件（首先被调度）更快速。 |
| `spark.sql.broadcastTimeout`           | 300                | 在广播join（broadcast joins）中广播等待超时时间（秒）。      |
| `spark.sql.autoBroadcastJoinThreshold` | 10485760 (10 MB)   | 配置进行join时，会被广播到所有工作者节点的表的最大字节大小。把这个值设置为`-1`可以关闭广播。注意，根据目前的统计数据，只支持可以执行`ANALYZE TABLE <tableName> COMPUTE STATISTICS noscan` 命令的Hive Metastore表。 |
| `spark.sql.shuffle.partitions`         | 200                | 配置对joins或者聚合进行数据混洗时的分区的数量。              |

