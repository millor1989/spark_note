### 待扩展

### DataSource 表

DataSource 方式读取文件

```scala
spark.read.text("hdfs://nameservice/data/dw/dw_db/db_logs/p_appkey=app/p_day=20221202")
```

文件读取参数（sql 前缀适用于 `spark.sql` 读取 Parequet、JSON、ORC，其它适用于 `spark.read`）：

`spark.sql.files.maxPartitionBytes`

`spark.sql.files.openCostInBytes`

`spark.sql.files.minPartitionNum`

如下两个没找到用的场合：

`spark.files.maxPartitionBytes`

`spark.files.openCostInBytes`



##### 读取 text 文件时，

对于 spark.read RDD 实现是 `FileScanRDD`，

而 `spark.sql` 对应的 RDD 实现是 `HadoopRDD`，每个目录对应一个 `HadoopRDD`。

spark task 【stage tab】 inputsize 







spark datasource 读取支持使用通配符，底层使用的是 Hadoop 的 `Path` API：

```scala
/* 读取20230321 至 20230326 的数据 */
spark.read.text("hdfs://nameservice/data/db_logs/p_appkey=app/p_day=2023032[1-6]")
```

DataSource 表，使用 Spark 的 DataSource 创建的表称为 DataSource 表。建表语句使用 `Using` 加 datasource 的语法。[参考链接1](https://blog.csdn.net/weixin_40893503/article/details/124664973?spm=1001.2101.3001.6650.7&utm_medium=distribute.pc_relevant.none-task-blog-2%7Edefault%7EBlogCommendFromBaidu%7ERate-7-124664973-blog-103087029.pc_relevant_recovery_v2&depth_1-utm_source=distribute.pc_relevant.none-task-blog-2%7Edefault%7EBlogCommendFromBaidu%7ERate-7-124664973-blog-103087029.pc_relevant_recovery_v2&utm_relevant_index=8)、[参考链接2](https://www.jianshu.com/p/c2ce36583af5)

```sql
CREATE ...
USING <datasorce>
```

使用 `USING` 关键字时，Spark 将使用自身的 SerDe 进行序列化和反序列化，而使用 `STORED AS ...` 时 Spark 将会使用 Hive 的 SerDe。使用 Spark 自身的 SerDe 时效率会高很多。

DataSource 表在读取文件时，会自动合并小文件。

如果数据源在变化，应用有多个行动算子且对结果实时性要求较高，那么需要对 DataSource 进行刷新（或者对 catalog 缓存进行清理）。

##### 例

```sql
create table if not exists datasoure_table_test_tmp (
    uid BIGINT COMMENT '用户ID',
    deviceid STRING COMMENT '设备ID',
    plateform STRING COMMENT '平台：h5/ios/wap/android',
    sessionid STRING,
    p_appkey string,
    p_day int)
using parquet
-- location 'hdfs://nameservice/group/xxx/db/xxx.db/logs_table_temp'
options(path='hdfs://nameservice/group/xxx/db/xxx.db/logs_table_temp')
partitioned by (p_appkey,p_day)
```

创建时，分区字段必须包含在字段列表中，但是指定分区字段时不需要指定分区字段类型。

创建的表默认是外部表（虽然没有使用 `external` 关键字），但不是临时表（`isTemporary ` 为 `false`）。

```scala
scala> spark.catalog.listTables.filter('name === "datasoure_table_test_tmp").show(false)
+------------------------+--------+-----------+---------+-----------+
|name                    |database|description|tableType|isTemporary|
+------------------------+--------+-----------+---------+-----------+
|datasoure_table_test_tmp|default |null       |EXTERNAL |false      |
+------------------------+--------+-----------+---------+-----------+
```

虽然对应的 path 有数据，但是，查询计数结果是 0（即使执行 `refreshTable`），新建的表需要执行 `recoverPartitions` （就是 Hive `add partitions` 或 `repair table ...` 的道理吧）才能发现数据：

```bash
# 刷新表
scala> spark.catalog.refreshTable("datasoure_table_test_tmp")

scala> spark.sql("select count(1) from datasoure_table_test_tmp").show(false)
+--------+
|count(1)|
+--------+
|0       |
+--------+

# 执行分区发现
scala> spark.catalog.recoverPartitions("datasoure_table_test_tmp")

scala> spark.sql("select count(1) from datasoure_table_test_tmp where p_appkey='wuliguaiguai' and p_day=20221201").show(false)

+--------+
|count(1)|
+--------+
|2300000 |
+--------+
```

如果执行了 `df.save(...)` 操作修改分区，或者直接改动了分区下的文件，需要执行 `spark.catalog.refreshTable(...)` 来刷新缓存的表的数据和元数据信息（比如，文件的 block 信息），否则查询会报错。

如果执行了 `df.save(...)` 操作增加分区还需要执行 `spark.catalog.recoverPartitions(...)`。

可能是因为分区字段有一个是 `int` 类型的缘故，查询抛出了**警告**信息：

```
22/12/02 17:27:53 WARN client.Shim_v1_1: Caught Hive MetaException attempting to get partition metadata by filter from Hive. Falling back to fetching all partition metadata, which will degrade performance. Modifying your Hive metastoreconfiguration to set hive.metastore.try.direct.sql to true may resolve this problem.
java.lang.reflect.InvocationTargetException
...
Caused by: MetaException(message:Filtering is supported only on partition keys of type string)
```

`spark.catalog.isCached("datasoure_table_test_tmp")` 如果表被缓存到内存中，则返回 `true`。

`spark.sql("drop table xxxx")` 删除外部表不会删除表的数据（hdfs 文件），因为创建的相当于是外部表。

#### datasoure 表的 `insert overwrite table...`

如果执行 `insert overwrite ` 时，没有明确的指定分区（`partition(pt='xxxx')`）、或者只是模糊的指定了分区（`part_a='xxx',part_b`），那么默认情况下 datasource 表的整个表数据或者任何匹配的分区的数据都会被覆盖，而不是根据提供的数据中的分区数据进行动态的分区覆盖。这种情况是**静态分区覆盖**。

`spark.sql.sources.partitionOverwriteMode` 配置控制覆盖 datasource 表分区的模式（默认值 `STATIC`），如果要使用**动态分区覆盖**，需要将其配置为 `dynamic`，那么就会根据插入数据中分区字段的值进行动态分区覆盖。

#### 不能覆盖正读取的表

对于 datasource 表来说不支持使用 `insert overwrite` 覆盖正在读取的表。即使设置 `spark.sql.hive.convertMetastoreParquet` 为 `false` 也是不行的。如果要覆盖正在读取的表的分区，使用 `DataFrameWriter.save(...)`。

非 datasource 表（parquet） 是可以的。

#### 空表保存

##### 1、DataSource 的方式保存

**空的 DataFrame 保存**：

```scala
val df = spark.emptyDataFrame

df.write.mode(SaveMode.Overwrite).save(s"hdfs://nameservice/group/xxxxxx")
```

会抛出异常：

```
org.apache.spark.sql.AnalysisException:
Datasource does not support writing empty or nested empty schemas.
Please make sure the data schema has at least one or more column(s).
         ;
        at org.apache.spark.sql.execution.datasources.DataSource$.org$apache$spark$sql$execution$datasources$DataSource$$validateSchema(DataSource.scala:733)
        at org.apache.spark.sql.execution.datasources.DataSource.planForWriting(DataSource.scala:523)
        at org.apache.spark.sql.DataFrameWriter.saveToV1Source(DataFrameWriter.scala:281)
        at org.apache.spark.sql.DataFrameWriter.save(DataFrameWriter.scala:270)
        at org.apache.spark.sql.DataFrameWriter.save(DataFrameWriter.scala:228)
        at com.jdd.moto.features.DT$.main(DT.scala:18)
        at com.jdd.moto.features.DT.main(DT.scala)
        at sun.reflect.NativeMethodAccessorImpl.invoke0(Native Method)
        at sun.reflect.NativeMethodAccessorImpl.invoke(NativeMethodAccessorImpl.java:62)
        at sun.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:43)
        at java.lang.reflect.Method.invoke(Method.java:498)
        at org.apache.spark.deploy.yarn.ApplicationMaster$$anon$2.run(ApplicationMaster.scala:694)
```

看异常信息，是需要有 schema 的。

但是如果创建了**空 DataSet**，是可以保存的：

```scala
import spark.implicits._

case class DS_t(deviceid: String, event_hour: Byte, duration: Int)

val ds = spark.emptyDataFrame.as[DS_t]

ds.write.mode(SaveMode.Overwrite).save(s"hdfs://nameservice/group/xxxxxx")
```

但是这种情况下，还是会产生数据文件的（因此会将原有的数据覆盖），尽管很小（byte 级别）。parquet 文件内容是 schema 之类东西：

```
PAR1LH
      spark_schema
                  deviceid%%
event_hour%duration
                   )org.apache.spark.sql.parquet.row.metadata{"type":"struct","fields":[{"name":"deviceid","type":"string","nullable":true,"metadata":{}},{"name":"event_hour","type":"byte","nullable":false,"metadata":{}},{"name":"duration","type":"integer","nullable":false,"metadata":{}}]}9parquet-mr version 1.5.0-cdh5.13.3 (build ${buildNumber})
```

##### 2、sql 方式保存

无论是 Datasource 表还是普通 Hive 表，`spark.sql("insert overwrite...")` 数据源表是空的时候，**都不会产生任何影响**。

**综上**，如果 Spark 想要删除表文件，只能通过 直接操作文件的方式。



### 被 `try {...} catch...` 包括行动操作如果失败，不会导致整个应用的失败，后续的 job（行动操作）还会执行。



### Stage 类型

#### 1、ShuffleMapStage

ShuffleMapStage 可以被看作是 Spark 物理执行 DAG 的中间（intermediate） Stage。它会产生其它 Stage 需要的数据。在 AQE 中可以将 ShuffleMapStage 当作 Spark 执行的最终 stage，因为对于 AQE，ShuffleMapStage 可能会被作为独立的 Spark job 进行提交。

执行时 ShuffleMapStage 会保存 map output files，reduce tasks 会获取这些文件。当所有的 map output 文件准备就绪，ShuffleMapStage 就算是完成了。

ShuffleMapStage 可以被当作是 DAG 中后续 Spark stages 的输入，即 shuffle dependency 的 map 端。shuffle 操作之前，ShuffleMapStage 可以有多种管道操作（比如 map、filter）。不同 jobs 可以共用一个 shuffleMapStage。

#### 2、ResultStage

一个 job 的最后一个 stage



### Join

Spark 将数据分发在不同的节点以进行并行处理，当进行两个 DataFrame join 时，如果两个 DataFrame 的数据都分布在集群的多个节点上，就需要混洗数据——因为对于每个进行 join 的 key 的数据可能不是都位于在同一个节点上，并且为了进行数据的 join 每个 key 的数据都应集中到相同的节点。

#### 1、Broadcast Join

将较小的 DataFrame 广播到所有的 executor（处理大表数据的 task 位于的所有 executors），executor 将小 DataFrame 的数据保存在内存中，这样就不用混洗较大 DataFrame 的数据——因为每个 executor 都有 join 需要的数据。

被**广播的 DataFrame 应该足够小**——Spark Driver / Executor 能够承载，否则会导致 Driver / Executor OOM。

##### 1.1、[Broadcast hash join](https://www.hadoopinrealworld.com/how-does-broadcast-hash-join-work-in-spark/)

broadcast hash join 是最快的 join 算法。

没有 shuffle 和 sort

执行计划中对应的名称为 `BroadcastHashJoin `。

- **广播阶段**：小表被广播到所有的 executors

  ![Broadcast Hash join Spark stage 1](/assets/Broadcast-Hash-join-Spark-stage-1.png)

- **hash join 阶段**：被广播到 executor 的小表被按 key hashed 到 buckets 中；小表被哈希化、分桶完成后，开始与大表进行 join，大表的 key 按照 key 的哈希值只会和小表对应的 bucket 中的 key 尝试进行匹配。

  ![Broadcast Hash join Spark stage1 - partition 1 and 2](/assets/Broadcast-Hash-join-Spark-stage1-partition-1-and-2.png)

**发生场景**：仅对等值连接生效、并且对 full outer join 不生效。

##### 1.2、[Broadcast nested loop join](https://www.hadoopinrealworld.com/how-does-broadcast-nested-loop-join-work-in-spark/)

嵌套的 for 循环 join。

不会 shuffle 和 sort。

未被广播的 DataFrame 的每一条记录会尝试**与被广播表的所有记录进行 join**。所以，有可能会非常慢。

等值或者非等值 join 都可以执行 BroadcastNestedLoopJoin，**当进行非等值 join 时，这种方式是默认的 join 方式**。

执行计划中对应的名称为 `BroadcastNestedLoopJoin `。

分为两个阶段：

- 广播阶段：小表被广播到所有 executors

  ![Broadcast Nested Loop Spark stage 1](/assets/Broadcast-Nested-Loop-Spark-stage-1.png)

- 嵌套循环 join 阶段：每一条记录会和被广播表的所有记录进行循环匹配，循环直到贯穿整个数据集才会停止。

  ![Broadcast Nested Loop Spark stage 1 and 2](/assets/Broadcast-Nested-Loop-Spark-stage-1-and-2.png)

**发生场景**：等值、非等值连接，所有类型的连接（inner、各种outer）都可能发生 BroadcastNestedLoopJoin。

broadcast nested loop join 是应该竭力避免的，如果能转换为 broadcast hash join 当然是最好的。

**例**：

如下为 broadcast nested loop join：

```scala
df1.join(broadcast(df2), $"id1" === $"id2" || $"id2" === $"id3", "left")
```

换一种代码实现方式，即可转换为 broadcast hash join：

```scala
val part1 = df1.join(broadcast(df2), $"id1" === $"id2" ", "left") 
val part2 = df1.join(broadcast(df2), $"id2" === $"id3", "left") 
val resultDF = part1.unionByName(part2)
```

#### 2、Shuffle Hash Join

对进行 join 的两个表进行 shuffle，两个表中，相同的 key 会位于相同的分区（task）。shuffle 完成后，两个表中较小的表会被 hashed 为 buckets，然后按分区执行 hash join。Shuffle Hash Join 不需要排序（sort）。

`spark.sql.join.preferSortMergeJoin` 应该设置为 `false`。并且，不满足广播的条件时，才会启用 shuffle hash join。执行计划中对应名称为 `ShuffledHashJoin `。

- **shuffle 阶段**：读取 join 的两张表并进行 shuffle。shuffle 之后，具有相同 key 的记录位于相同的分区。

  ![Shuffle Hash Join Stages](/assets/Shuffle-Hash-Join-Stages.png)

- **hash join 阶段**：对小表侧的 shuffle 结果按 key 哈希化为 buckets。分区内，大表侧的 shuffle 结果按 key 尝试与小表侧对应的 bucket 进行匹配。 

  ![Shuffle Hash Join Stage 3](/assets/Shuffle-Hash-Join-Stage-3.png)

**发生场景**：仅等值连接时发生，各种类型 join 都可能发生。

对于数据倾斜严重的数据不适用，某个严重倾斜的 key 对应记录被发送的一个分区——在一个分区内对这个 key 的记录进行 hash 会导致 OOM。所以，Shuffle hash join 仅应该使用在比较均衡的数据集上。

#### 3、Shuffle Sort Merge Join

`spark.sql.join.preferSortMergeJoin` 默认为 `true`。即，当两个 join 的表足够大时，执行 （Shuffle）Sort Merge Join。

在执行计划中 shuffle sort merge join 的名称为 `SortMergeJoin `。

- **shuffle 阶段**：两个表都进行 shuffle。shuffle 之后，相同 key 的数据会位于相同的 分区

- **sort 阶段**：两个表都按 key 进行 sort。按 key 进行排序，不会进行哈希化和分桶操作。

- **merge 阶段**：遍历两个表并基于连接 key 进行 join。通过对排序后的数据集进行遍历来执行 join。因为数据是排过序的，当遇到匹配的 key 之后 join 操作就会停止——不会对所有的 key 尝试匹配。

  ![Shufflee Sort Merge stage 3](/assets/Shufflee-Sort-Merge-stage-3.png)

**发生场景**：仅在等值连接时才会发生，没有表会被广播时发生。

##### shuffle sort mege join 转换为 shuffle hash join

如果数据集是均匀分布的，并且一个表 hashed 时不会导致 OOM，如果较小数据集的大小的 3 倍小于较大数据集的大小（`small_df_size * 3 < large_df_size`），那么可以考虑将 shuffle sort merge join 转换为 shuffle hash join。

前提条件是：`spark.sql.join.preferSortMergeJoin` 设置为 `false`；`spark.sql.autoBroadcastJoinThreshold` 设置为较小值，或者关闭。

#### 4、Cartesian Product Join

笛卡尔积连接，Cartesian product join 也叫作 shuffle-and replication nested loop join，与 broadcast nested loop join 类似，但是不涉及表的广播。

shuffle-and-replication 并不意味着真正的 shuffle，而是整个表都被发送（或者备份）到所有的分区以执行 cross（nested loop） join。

- **执行**：读取所有的表数据，一个表的所有分区被发送到另一个表的各个分区。然后执行 nested loop join。如果一个表有 n 条数据，另一个表有 m 条数据，那么会对 `n * m` 条数据执行 nested loop。

  ![Cartesian product join Stage 1](/assets/Cartesian-product-join-Stage-1.png)

此图有些错误，仅供参考，因为不存在分区间数据的交互——其中一个表是整个发送到另一个表的分区的。

**发生场景**：只发生在类似 inner join 的连接时，等值、非等值连接都可能发生。

开销非常之大，很可能导致 OOM。

#### 5、Spark Join 实现的选择

**Catalyst** 在由优化的逻辑计划生成物理计划的过程中，会根据 **`org.apache.spark.sql.execution.SparkStrategies` 类的 `JoinSelection` 对象**提供的规则按顺序确定 join 的执行方式。join 策略的选择会按照效率从高到低的优先级来排列。

##### 5.1、Join 条件（join condition）

等值连接（`=`）可以使用所有的 join 实现。

非等值连接（除了 `=` 条件）可以使用的 join 实现：broadcast nested loop join、cartesian product join

##### 5.2、Join 类型（join types）

join 类型：inner、outer、left-smei、etc.

###### join 类型及其语义：

![img](/assets/join_types_and_its_semantics.png)

可用于**所有 join 类型**的 spark join 实现：shuffle hash join、shuffle sort merge join、broadcast nested loop join。

**broadcast hash join** 可用于**除了 full outer join** 以外类型的 spark join 实现。

**cartesian product join** 实现**只能用于 inner like joins**。

##### 5.3、是否指定暗示（hints）

spark 3.0 可以指定 hints 暗示，让 spark 使用指定的 join 实现。

**指定了暗示时**：

1. broadcast hint：如果 join 类型（types）支持广播 join，就使用 broadcast hash join；如果连接的两个表都有 broadcast hint 那么选择较小的表进行广播。
2. sort merge hint：如果连接 key 可以排序，就是用 sort merge join
3. shuffle hash hint：如果 join 类型支持 shuffle hash hint 就使用，如果连接的两个表都有 shuffle hash hint，那么对较小的表构建哈希map进行哈希分桶。
4. shuffle replicate NL hint：如果 join 类型是类 inner join，就是用 cartesian product join。

**没有指定暗示时**，spark join 实现选择的优先级顺序（按照效率从高到低）：

1. 当某一侧表小到可以被广播，并且 join 类型支持，就使用 broadcast hash join，如果连接的两个表都可以被广播，那么选择较小的表进行广播。
2. 如果 `spark.sql.join.prefer.SortMergeJoin` 为 `false`，并且一个表比较小，并且能够对较小的表构建本地 hash map（小于 `spark.sql.autoBroadcastJoinThreshold * spark.sql.shuffle.partitions`），那么选择 shuffle hash join。
3. 如果连接 keys 是可排序的，使用 sort merge join
4. 如果连接类型是类 inner join，选择 cartesian product join
5. 以上都不满足，使用 broadcast nested loop join。（尽管可能导致 OOM，但是别无选择）

#### 6、[`not in ` 和 `not exists`](https://docs.databricks.com/_static/notebooks/kb/sql/broadcastnestedloopjoin-example.html)

有时候，即使关闭了自动广播（设置 `spark.sql.autoBroadcastJoinThreshold` 为 `-1`），还是看到了 spark 尝试广播表并且广播发生错误（超时）。这不是 bug，很可能是 `BroadcastNestedLoopJoin ` 引起的。

假设有两个表，一个没有 null 值，一个有 null 值：

```scala
spark.sql("SELECT id FROM RANGE(10)").write.mode("overwrite").saveAsTable("tblA_NoNull")
spark.sql("SELECT id FROM RANGE(50) UNION SELECT NULL").write.mode("overwrite").saveAsTable("table_withNull")

spark.conf.set("spark.sql.autoBroadcastJoinThreshold", -1)
spark.sql("select * from table_withNull where id not in (select id from tblA_NoNull)").explain(true)
```

如果查看执行计划，会发现 `BroadcastNestedLoopJoin`：

```
*(2) BroadcastNestedLoopJoin BuildRight, LeftAnti, ((id#2482L = id#2483L) || isnull((id#2482L = id#2483L)))
:- *(2) FileScan parquet default.table_withnull[id#2482L] Batched: true, DataFilters: [], Format: Parquet, Location: InMemoryFileIndex[dbfs:/user/hive/warehouse/table_withnull], PartitionFilters: [], PushedFilters: [], ReadSchema: struct<id:bigint>
+- BroadcastExchange IdentityBroadcastMode, [id=#2586]
   +- *(1) FileScan parquet default.tbla_nonull[id#2483L] Batched: true, DataFilters: [], Format: Parquet, Location: InMemoryFileIndex[dbfs:/user/hive/warehouse/tbla_nonull], PartitionFilters: [], PushedFilters: [], ReadSchema: struct<id:bigint>
```

如果表比较大，那么 Spark 尝试广播表时就会发生广播错误。

如果使用 `not exists` 替代 `not in`：

```scala
sql("select * from table_withNull where not exists (select 1 from tblA_NoNull where table_withNull.id = tblA_NoNull.id)").explain(true)
```

执行计划中就成了 `SortMergeJoin`：

```
SortMergeJoin [id#2482L], [id#2483L], LeftAnti
:- Sort [id#2482L ASC NULLS FIRST], false, 0
:  +- Exchange hashpartitioning(id#2482L, 200), [id=#2653]
:     +- *(1) FileScan parquet default.table_withnull[id#2482L] Batched: true, DataFilters: [], Format: Parquet, Location: InMemoryFileIndex[dbfs:/user/hive/warehouse/table_withnull], PartitionFilters: [], PushedFilters: [], ReadSchema: struct<id:bigint>
+- Sort [id#2483L ASC NULLS FIRST], false, 0
   +- Exchange hashpartitioning(id#2483L, 200), [id=#2656]
      +- *(2) Project [id#2483L]
         +- *(2) Filter isnotnull(id#2483L)
            +- *(2) FileScan parquet default.tbla_nonull[id#2483L] Batched: true, DataFilters: [isnotnull(id#2483L)], Format: Parquet, Location: InMemoryFileIndex[dbfs:/user/hive/warehouse/tbla_nonull], PartitionFilters: [], PushedFilters: [IsNotNull(id)], ReadSchema: struct<id:bigint>
```

由于 `not in` 和 `not exists` 语义上有所不同，所以，Spark 不会自动进行转换：

- 对于 `in`、`not in` 只有当**判断结果为 true** 时才返回对应的结果；所以，如果是 `not in` 子查询中有 `null` 时，整个查询为空；并且 `in`、`not in` 返回的结果中都不会有 null 值。
- 对于 `exists`、`not exists` 关心的只是**子查询是否有结果**（`not exists` 只要子查询没有结果就返回对应值）；`not exists` 总是将 `null` 值返回到结果中，而 `exists` 的结果中不会有 null 值。

另，`in`、`not in` 会先执行子查询，而 `exists`、`not exists` 则是后执行子查询；`in`、`not in` 适用于子查询结果较少的情况，而 `exists`、`not exists` 适用于外部查询结果较少的情况。



### Spark 3.0 Join

Spark 会根据一些内部逻辑选择 join 算法，但是 Spark 的默认选择可能不是最好的。

Spark 3.0 提供了一个使用策略暗示（strategy hints）选择指定算法的方式：

```scala
dfA.join(dfB.hint(algorithm), join_condition)
```

其中 `algorithm` 可以是：`broadcast`、`shuffle_hash`、`shuffle_merge`；而 Spark 3.0 之前，只能是 `broadcast`——作用与 `broadcast` 函数相同：

```scala
dfA.join(broadcast(dfB), join_condition)
```

#### 1、`JoinSelection` 策略

在物理计划生成阶段，逻辑计划的每个节点必须使用策略（strategies）转换为一个或者多个算子。Spark 使用 `JoinSelection` 策略决定使用那种算法进行 join。其中，影响进行 join 算法选择的因素有：

- 暗示
- 连接条件（是否是等值连接）
- 连接的类型（inner、left、full outer、...）
- 连接时的数据量

#### 2、广播连接（BroadcastHashJoin）

join 一侧的表足够小（小于配置 `spark.sql.autoBroadcastJoinThreshold`，按字节，默认 `10 * 1024 * 1024`），小表被广播到每个 executor，大表不用混洗。DAG 如下：

![img](/assets/Pzvy3UDoNIP6g1pOZwu1qw.png)

Spark 对表大小的评估依赖于读取数据的方式、metastore 中是否计算了统计数据（statistics）以及 CBO（cost-based optimization）特性是否开启。`autoBroadcastJoinThreshold` 的默认值是比较保守的，设置合适的值依赖于集群资源——要比埋你 OOM 或者广播超时（`spark.sql.broadcastTimeout`，默认值 300，单位秒）。

除了数据量大导致广播超时，获得小表数据的运算过于复杂也会导致广播超时。比如：

```scala
dfA = spark.table(...)

dfB = (
  data
  .withColumn("x", udf_call())
  .groupBy("id").sum("x")
)

dfA.join(dfB.hint("broadcast"), "id")
```

其中 `dfB` 是一个比较大的表 `data` 聚合后的结果。但是，如果 UDF 耗时过长就会导致广播超时。除了增加广播超时时间，还可以使用 `cache()`，预先计算出 `dfB`，这样仍然可以进行广播 join：

```sacla
dfA = spark.table(...)

dfB = (
  data
  .withColumn("x", udf_call())
  .groupBy("id").sum("x")
).cache()

dfB.count()

dfA.join(dfB.hint("broadcast"), "id")
```

此时，这个查询会以三个 jobs 执行。第一个 job 由 `count` 触发，计算聚合结果并缓存；第二个 job 负责将缓存结果广播到每个 executor；最后一个 job 执行 join。

#### 2、SortMergeJoin

如果 join 两侧的表都不能被广播， join key 可以排序并且 join 是等值 join，Spark 将会采用 `SortMergeJoin`。

`SortMergeJoin` 需要两侧的表有合适的分区和排序，通常通过 join 两侧分支的混洗和排序来满足这个条件。典型的物理计划 DAG 如下：

![img](/assets/HW3YcSgA2KdS2MYGbPsxpw.png)

其中 `Exchange` 和 `Sort` 算子确保数据被以合适的方式分区、排序，从而能够进行最终的合并。

另外，通过合理运用 `repartition` 算子或对数据源表进行合适的分桶可以减少混洗的次数。

#### 3、ShuffledHashJoin

虽然，Spark 提供了 `ShuffledHashJoin` 但是它不常见，因为 Spark 的内部配置 `spark.sql.join.preferSortMergeJoin` 默认为 `true`，即，当 Spark 在 `SortMergeJoin` 和 `ShuffledHashJoin` 之间做选择时会选择 `SortMergeJoin`，因为就 OOM 而言 SortMergeJoin 更稳定。使用 `ShuffedHashJoin` 时，如果一个分区不能放进内存，job 就会失败，但是，对于 `SortMergeJoin` 会把数据溢出到磁盘，虽然执行速度慢但是保证了 job 正常执行。

如果 `spark.sql.join.preferSortMergeJoin` 为 `false`，不能进行 `BroadcastHashJoin`，join 一侧表的**分区的平均大小**大小的 **3 倍**小于另一侧的表且分区平均大小小于广播阈值（**canBuildLocalHashMap** `plan.stats.sizeInBytes < conf.autoBroadcastJoinThreshold * conf.numShufflePartitions`）会使用 `ShuffledHashJoin`。因为检查的是平均分区大小，如果数据严重倾斜，某个分区特别大，内存无法容纳，就会失败。

与 `SortMergeJoin` 类似，`ShuffledHashJoin` 需要数据正确地进行分区，所以通常它会在 join 的两个分支引入混洗。但是与 `SortMergeJoin` 不同的是，`ShuffledHashJoin` 不需要排序，排序也是一个开销很大的操作，因此 `ShuffledHashJoin` 很可能比 `SortMergeJoin` 更快。

当 join 一侧表比另一侧表小得多（3 倍小于）时，`ShuffledHashJoin` 会比 `SortMergeJoin` 更快，因为此时对一个表构建 hash map 相比对两个表进行排序优势明显。`ShuffledHashJoin` 的物理计划：

![img](/assets/IDRGpNF7C-IwlMYoOEvAA.png)

#### 4、BroadcastNestedLoopJoin

上面的三个 join 算法要求是等值连接。如果是非等值连接，Spark 就只能采用 `BroadcastNestedLoopJoin` 或者笛卡尔积。如果一侧表可以被广播，就会采用 `BroadcastNestedLoopJoin`。`BroadcastNestedLoopJoin` 和笛卡尔积都是非常慢的，所以应该尽可能使用等值连接。



### ShuffleManager

Spark shuffle 过程中的主要组件是 ShuffleManager。

Spark 2.0 开始摒弃了 **HashShffleManager**，只保留了 **SortShuffleManager**。

**HashShuffleManager** 有一个很严重的弊端——会产生大量的磁盘文件，大量的磁盘 IO 会严重影响性能。

**SortShuffleManager** 虽然也会产生较多的临时磁盘文件，但是每个 task 最后会将所产生的临时文件合并为一个磁盘文件，所以每个 task 只有一个磁盘文件（另外还有一个索引文件）。下一个 stage 的 shuffle read task 只需要根据索引读取磁盘文件中对应部分的数据即可。

SortShuffleManager 使用的是 sort-based shuffle，输入记录会根据目标分区 id 进行排序，然后写到一个 map 输出文件。reducers 通过索引获取 map 输出中相应（连续的）部分的文件。如果 map 输出数据太大超过内存限制，map 输出的排过序的子集会溢出到磁盘，最终这些溢出的文件会被合并以产生最终的输出文件。

#### 1、sort-based shuffle 生成 map 输出文件的两种方式

##### 1.1、序列化排序（serialized sorting）

满足以下三个条件时，会进行序列化的排序：

1. shuffle dependency 没有指定 map-side combine
2. shuffle serializer 支持序列化的值得重定位（relocation）——目前 KryoSerializer 和 Spark SQL 的自定义序列化器都支持
3. shuffle 产生的分区数小于等于 16777216

条件不满足，则使用非序列化排序。

在序列化排序模式下，输入记录被 shuffle writer 读取到时，会马上进行序列化，并且在排序期间是以序列化的形式缓存的。这种模式实现了几点优化：

- 对序列化的二进制数据而不是 Java 对象进行排序，减少了内存消耗和 GC 开销。
- 使用一个特殊的缓存高效的（cache-efficient）排序器（`ShuffleExternalSorter`）对压缩的记录指针和分区 id 的数组进行排序。在对这些数组的排序中每条记录只使用了 8 字节的空间，可以将更多的数组加入到缓存。
- 溢出合并处理（spill merging procedure）操作的是属于相同分区的序列化记录的块（blocks），在合并期间不需要反序列化处理。
- 当溢出压缩 codec 支持压缩数据的串联（concatenation）时，溢出合并操作就只是简单地将序列化并且压缩后的溢出数据进行串联，从而产生最终的分区输出。这样就能够使用高效的复制方法，比如 NIO 的 `transferTo`，避免了合并期间分配解压缩或者复制缓冲区的需要。

##### 1.2、非序列化排序（deserialized sorting）

不满足序列化排序时使用非序列化排序。

#### 2、SortShuffleManager：

根据 `SparkEnv` 源码的 `create` 方法：

```scala
private[spark] val SHUFFLE_MANAGER = ConfigBuilder("spark.shuffle.manager").stringConf.createWithDefault("sort")

// Let the user specify short names for shuffle managers
val shortShuffleMgrNames = Map(
    "sort" -> classOf[org.apache.spark.shuffle.sort.SortShuffleManager].getName,
    "tungsten-sort" -> classOf[org.apache.spark.shuffle.sort.SortShuffleManager].getName)
val shuffleMgrName = conf.get(config.SHUFFLE_MANAGER)
val shuffleMgrClass =
shortShuffleMgrNames.getOrElse(shuffleMgrName.toLowerCase(Locale.ROOT), shuffleMgrName)
val shuffleManager = instantiateClass[ShuffleManager](shuffleMgrClass)

// spark 1.6+ 开始就不支持 spark.shuffle.spill 设置为 false 了。shuffle 会溢出到磁盘。
if (!conf.getBoolean("spark.shuffle.spill", true)) {
    logWarning(
        "spark.shuffle.spill was set to false, but this configuration is ignored as of Spark 1.6+." +
        " Shuffle will continue to spill to disk when necessary.")
}
```

无论通过 `spark.shuffle.manager` 指定使用 `sort` 还是 `tungsten-sort`，最终都只使用 `SortShuffleManager`。

`SortShuffleManager` 有三个核心方法：

1.  `registerShuffle()` 方法，用来注册 Shuffle 的机制，返回对应的 `ShuffleHandle`, shuffle handle 会存储 shuffle 依赖信息，根据 shuffle handle 可以确定使用 的 `ShuffleWriter`。
2. `getWriter()` 方法，获取 `ShuffleWriter`。在 executors 上，由 map tasks 调用。
3. `getReader()` 方法，获取一定范围的 reduce 分区的 `ShuffleReader`。在 executors 上，由 reduce tasks 调用。

##### 2.1、`SortShuffleManager` 的三种运行机制

根据 `registerShuffle()` 方法实现，有三种运行机制——对应三种 `shuffleHandle`，`getWriter()` 方法根据不同的 `ShuffleHandle` 获取相应的 `ShuffleWriter`，

`ShuffleWriter` 都实现了 `write()` 方法，由 `scheduler.ShuffleMapTask.runTask()` 方法来调用。（spark 的 stage 分为 `ShuffleMapStage` 和 `ResultStage`，task 也分为 `ShuffleMapTask` 和 `ResultTask`。）

运行机制：

1. ##### 普通机制（BaseShuffleHandle）

   数据先写入内存数据结构（Map 或 Array）中，如果是聚合类的 shuffle 算子（reduceByKey 等）使用 Map 结构，一边通过 Map 进行聚合，一边写入内存；如果是不需要聚合的普通 shuffle 算子（join 等）使用 Array 结构，直接写入内存。写入内存的数据达到阈值之后，尝试将内存数据结构中的数据溢出到磁盘——溢写之前会根据 key 进行排序。溢写到磁盘文件是通过 `BufferedOutputStream` 实现的——会先将数据写入内存中的数据缓冲区，缓冲区写满后后才溢写到磁盘——这样可以减少磁盘 IO，提升性能。每次溢写都产生一个临时的磁盘文件，最后 shuffle write task 会将所有临时的磁盘文件合并，这就是 merge 过程。最终一个 task 只有一个磁盘文件。

   `BaseShuffleHandle` 对应的是普通的 `SortShuffleWriter`：

   ```scala
     /** Write a bunch of records to this task's output */
     override def write(records: Iterator[Product2[K, V]]): Unit = {
       sorter = if (dep.mapSideCombine) {
         new ExternalSorter[K, V, C](
           context, dep.aggregator, Some(dep.partitioner), dep.keyOrdering, dep.serializer)
       } else {
         // In this case we pass neither an aggregator nor an ordering to the sorter, because we don't
         // care whether the keys get sorted in each partition; that will be done on the reduce side
         // if the operation being run is sortByKey.
         new ExternalSorter[K, V, V](
           context, aggregator = None, Some(dep.partitioner), ordering = None, dep.serializer)
       }
       sorter.insertAll(records)
   
       // Don't bother including the time to open the merged output file in the shuffle write time,
       // because it just opens a single file, so is typically too fast to measure accurately
       // (see SPARK-3570).
       val output = shuffleBlockResolver.getDataFile(dep.shuffleId, mapId)
       val tmp = Utils.tempFileWith(output)
       try {
         val blockId = ShuffleBlockId(dep.shuffleId, mapId, IndexShuffleBlockResolver.NOOP_REDUCE_ID)
           // 写数据到磁盘
         val partitionLengths = sorter.writePartitionedFile(blockId, tmp)
         shuffleBlockResolver.writeIndexFileAndCommit(dep.shuffleId, mapId, partitionLengths, tmp)
         mapStatus = MapStatus(blockManager.shuffleServerId, partitionLengths)
       } finally {
         if (tmp.exists() && !tmp.delete()) {
           logError(s"Error while deleting temp file ${tmp.getAbsolutePath}")
         }
       }
     }
   ```

   `SortShuffleWriter` 不仅输出中间数据，还输出索引；它借助 `ExternalSorter` 类来处理数据。`write()` 方法中，**首先**就是创建外部排序器 `ExternalSorter`——如果 shuffle 依赖中有 map 端聚合就传入 `dep.aggregator`（预聚合） 和 `dep.keyOrdering`（key 排序）；如果没有 map 端聚合，就不传 aggregator 或 ordering，因为此时不关心每个分区中 keys 是否是排过序的，如果运行的算子是 `sortByKkey` 那么排序会在 reduce 端进行。**然后** `sorter.insertAll(records)` 将 shuffle 的数据放入 `ExternalSorter` 进行处理，并将处理之后的数据进行合并排序写到磁盘临时文件。**最后**`IndexShuffleBlockResolver` 则根据 `ExternalSorter` 输出的临时文件和分区大小生成最后的输出文件和文件索引。

   ```scala
     def insertAll(records: Iterator[Product2[K, V]]): Unit = {
       // TODO: stop combining if we find that the reduction factor isn't high
       val shouldCombine = aggregator.isDefined
   
       if (shouldCombine) {
         // Combine values in-memory first using our AppendOnlyMap
         val mergeValue = aggregator.get.mergeValue
         val createCombiner = aggregator.get.createCombiner
         var kv: Product2[K, V] = null
         val update = (hadValue: Boolean, oldValue: C) => {
           if (hadValue) mergeValue(oldValue, kv._2) else createCombiner(kv._2)
         }
         while (records.hasNext) {
           addElementsRead()
           kv = records.next()
           map.changeValue((getPartition(kv._1), kv._1), update)
           maybeSpillCollection(usingMap = true)
         }
       } else {
         // Stick values into our buffer
         while (records.hasNext) {
           addElementsRead()
           val kv = records.next()
           buffer.insert(getPartition(kv._1), kv._1, kv._2.asInstanceOf[C])
           maybeSpillCollection(usingMap = false)
         }
       }
     }
   ```

   **`ExternalSorter.insertAll()` 方法**，**插入数据时**，如果需要 map 端预聚合，会使用 `PartitionedAppendOnlyMap`（map）数据结构作为 buffer，如果不需要 map 端预聚合，就使用 `PartitionedPairBuffer`（array）数据结构作为 buffer。数据插入 buffer 后，会执行 `maybeSpillCollection()` 方法判断是否需要溢写，如果需要溢写文件，就重新 new 新的 map 或 array buffer，重新开始缓存。进行溢写判断 `Spillable.maybeSpill()` 时，如果当前 buffer 大小超过阈值 `spark.shuffle.spill.initialMemoryThreshold` 会**首先尝试从 shuffle memory pool 中申请扩充当前 buffer 的内存，如果申请不到内存才会真正开始溢写**。

   `ExternalSorter` 处理完数据后，通过 **`ExternalSorter.writePartitionedFile()`** 对数据进行排序与合并写到磁盘。输出文件之前，排序时，如果存在溢写文件，就将溢写文件与缓存数据合并排序，如果没有溢写文件只需对缓存数据进行排序。最终只会输出一个临时文件。

   `IndexShuffleBlockResolver` 生成索引文件并将 `ExternalSorter` 生成的临时文件重命名为正式文件。

2. ##### bypass 运行机制（BypassMergeSortShuffleHandle）

   前提：

   1. shuffle map task 数量小于 `spark.shuffle.sort.bypassMergeThreshold`（默认 200）参数的值。
   2. 不需要 map 端聚合——不是聚合类的 shuffle 算子（`reduceByKey`、`groupByKey` 等）——不需要排序

   优点：与普通机制相比，由于不需要 map 端聚合，省掉了写磁盘文件之前的排序开销，也避免了合并溢写文件时两次的序列化和反序列化。

   `BypassMergeSortShuffleHandle` 对应的是 `BypassMergeSortShuffleWriter`：

   由于没有 map 端预聚合，也不用排序，`BypassMergeSortShuffleWriter` 直接**按分区**写一批中间数据文件（也没有 map / array buffer），然后合并。合并完成后，`IndexShuffleBlockResolver` 生成索引文件并将 临时文件重命名为正式文件。

   虽然中间数据文件可能相对较多，但是数据量少、逻辑简单，在适当情况下也可能很快。

3. ##### 序列化 sort shuffle 机制（SerializedShuffleHandle）

   前提：

   1. 使用的序列化器支持序列化对象重定位（relocation）：即使用的序列化器可以对已经序列化的对象进行排序，并且排序效果与先对对象排序再进行序列化相同。KryoSerializer 和 Spark SQL 的自定义序列化器都支持对象重定位。
   2. 不需要 map 端聚合（shuffle 依赖没有指定聚合或者输出排序）
   3. 分区数量小于（2^24）（基本都能满足，千万级）

   优点：尝试以序列化的形式缓存 map 输出，更加高效。（普通机制，是以未序列化的形式缓存 map 输出的）
   
   序列化 sort shuffle 机制就是 tungsten-sort。
   
   Tungsten（钨丝）是 DataBricks 的一个 Spark 优化方案，从三方面优化 Spark 的 CPU 和内存效率：1)、显式内存管理和基于二进制的处理：由 Spark 应用自己管理（序列化的）对象和内存，消除 JVM 对象模型和 GC 等带来的 overhead；2)、对于缓存有感知的计算：提出高效的、能充分易用计算机存储体系的算法和数据结构；3)、代码生成技术方面：充分利用最新的编译器和 CPU 特性，提高运行效率。
   
   `SerializedShuffleHandle` 对应的是 `UnsafeShuffleWriter`：
   
   Tungsten 内部使用了很多 `sun.misc.Unsafe` 的 API，所以对应的 `ShuffleWriter` 类名为 `UnsafeShuffleWriter`。
   
   ```java
     @Override
     public void write(scala.collection.Iterator<Product2<K, V>> records) throws IOException {
       // Keep track of success so we know if we encountered an exception
       // We do this rather than a standard try/catch/re-throw to handle
       // generic throwables.
       boolean success = false;
       try {
         while (records.hasNext()) {
           insertRecordIntoSorter(records.next());
         }
         closeAndWriteOutput();
         success = true;
       } finally {
         if (sorter != null) {
           try {
             sorter.cleanupResources();
           } catch (Exception e) {
             // Only throw this error if we won't be masking another
             // error.
             if (success) {
               throw e;
             } else {
               logger.error("In addition to a failure during writing, we failed during " +
                            "cleanup.", e);
             }
           }
         }
       }
     }
   ```
   
   将 shuffle 数据插入排序器 `ShuffleExternalSorter` 进行处理，然后合并、写输出文件。
   
   `insertRecordIntoSorter` 方法实现：
   
   ```java
     @VisibleForTesting
     void insertRecordIntoSorter(Product2<K, V> record) throws IOException {
       assert(sorter != null);
       final K key = record._1();
       final int partitionId = partitioner.getPartition(key);
       serBuffer.reset();
       serOutputStream.writeKey(key, OBJECT_CLASS_TAG);
       serOutputStream.writeValue(record._2(), OBJECT_CLASS_TAG);
       serOutputStream.flush();
   
       final int serializedRecordSize = serBuffer.size();
       assert (serializedRecordSize > 0);
   
       sorter.insertRecord(
         serBuffer.getBuf(), Platform.BYTE_ARRAY_OFFSET, serializedRecordSize, partitionId);
     }
   ```
   
   可见，插入到排序器 `ShuffleExternalSorter` 中的数据是序列化之后的 shuffle 数据。shuffle wirte 结束之前都是对序列化数据进行操作的。具体实现[太复杂了](https://www.jianshu.com/p/1d714f0c5e07)。

##### 2.2、`ShuffleReader`

只有一种，`BlockStoreShuffleReader`。

读取 shuffle map task 的输出。



### Spark 3.0+ 查询计划（Query Plans）

基于查询计划中的信息，有可能找到引起 job 低效的原因，从而可以重写低效查询以提升性能。

直接调用 `df.explain()` 方法，展示的是查询的物理计划（Physical Plan）。Spark 3.0 中重载的 `explain` 方法可以接收一个 `mode` 参数——可选的值为 `formatted`、`cost`、`codegen`。使用 `formatted` 值可以看到一个格式美化后物理计划和对应算子的详细描述（物计划后面的标号，与下面算子前面的标号对应）。

![img](/assets/TRaSWP5otKhYcbdTZ809sA.png)

`cost` 模式的 `explain` 方法除了展示物理计划还展示优化的逻辑计划（Optimized Logical Plan）及每个算子相关的统计数据——可以看到每个执行步骤预估的数据大小。`codegen` 模式的 `explain` 会展示将要被执行的生成的 java 代码。

Spark WebUI 的 SQL 页签也可以看到查询计划和查询的 DAG 图。Spark 3.0 的 DAG 图中的 codegen 同时标注了对应的 codgen id。通过 codgen id 可以将 DAG 图和物理计划进行对应。

Spark 2.0 相对于 Spark 3.0 DAG 图就比较简陋了——没有 codegen id 展示。Spark 2.0 可以通过 `df.queryExecution.debug.codegen()` 查看分 Subtree 展示的各个 codegen 对应的物理计划：

![1676259743145](/assets/1676259743145.png)

物理执行计划中，支持 codegen 的算子前面回有 `*(<codegen id>)`，不支持的算子（比如 `Exchange`）前面则没有。

#### 1、`CollapseCodegenStages`

在 Spark 的 Web UI 中，可以看到 job DAG 中算子会被分组为 `WholeStageCodegen`。这实际上是在生成物理计划的过程中发生的一个优化特性——`CollapseCodegenStages`。

`CollapseCodegenStages` 将支持 code generation 的算子合并到一起执行，消除了虚拟函数调用从而提高执行效率。

#### 2、查询计划中的算子

##### 2.1、`Scan parquet`

`scan parquet` 算子表示从 parquet 格式文件中读取数据。从计划详情中可以看到从数据源筛选的字段。即使查询中不指定筛选字段，也会有一个叫作 `ColumnPruning` 的优化规则根据查询的实际需要从数据源筛选字段。还有两种过滤类型：

- `PartitionFilters`：基于数据源分区字段的过滤。**非常重要！**可以跳过扫描不需要的数据。Spark 2.4 中有一个属性 `partitionCount` ——实际扫描的分区数量，在 Spark 3.0 中已经没有了。
- `PushedFilters`：可以直接应用到 parquet 文件的对字段的过滤，如果 parquet 文件是按照这些过滤字段排序的，那么还可以利用内部的 parquet 结构跳过不需要的数据。parquet 文件是由行组（row groups）和包含了这些行组的元数据的文件的页脚（footer）组成的。这里的元数据包含了统计信息（比如每个行组的 min、max value），基于这些信息 Spark 可以决定是否要读取某个行组。

##### 2.2、`Filter`

过滤条件。通常它不对应于查询中使用的过滤条件。应为所有的过滤都是首先会被 Catalyst optimizer 处理，这可能会导致过滤被修改或重定位。逻辑过滤（logical filters）被转换为物理算子是的几个规则：

- `PushDownPredicates`（谓词下推）——通过几个其他的算子将过滤（并不是所有的）向数据源推近。比如，不是确定性的表达式（比如使用了函数 `first`、`last`、`collect_set`、`collect_list`、`rand` 等）不会被下推。
- `CombineFilters`——将两个相邻的算子组合为一个算子
- `InferFiltersFromConstraints`——这个规则实际上会创建新的 `Filter` 算子，比如对于 inner join 的连接条件会创建一个连接 key 不为 null 的过滤条件。
- `PruneFilters`——移除冗余的过滤（比如，条件值总是为 true 的过滤）

##### 2.3、`Project`

表示哪些字段会被选择。对 DataFrame 执行 `select`、`withColumn`、`drop` 操作时，Spark 会对逻辑计划增加一个 `Project` 算子。转化为物理计划中的对应算子时，也有一些优化规则：

- `ColumnPruning`——字段修剪，修剪掉不需要的字段，减少扫描的数据量。
- `CollapseProject`——将相邻的两个 `Project` 算子合并为一个。
- `PushProjectionThroughUnion`—— push the `Project` through both sides of the `Union` operator.

##### 2.4、`Exchange`

`Exchange` 表示混洗，是集群中物理的数据移动。开销很大的操作。在执行计划中，这个算子的信息包含了数据如何重新分区。比如 `hashpartitioning(user_id, 200)` 表示对数据按 `user_id` 进行哈希分区。分区类型：

- `HashPartitioning`——计算分区 key 的哈希值，然后对哈希值按分区数取模，结果相同则分配到相同的分区。
- `RoundRobinPartitioning`——数据被随机地分发到 `n`（分区数量） 个大小相同的分区。`repartition(n)`。
- `RangePartitioning`——用于排序数据是，在调用 `orderBy` 或 `sort` 之后使用。
- `SinglePartition`——所有数据被移动到一个分区。当调用了窗口函数而窗口是整个 DataFrame（不为窗口的 `partitionB()` 函数提供参数）时会发生。

##### 2.5、`HashAggregate`

表示数据聚合。通常是成对的出现（第一个是分区内聚合，第二个是分区间的聚合）——有可能被 `Exchange` 隔开。

##### 2.6、`BroadcastHashJoin` & `BroadcastExchange`

`BroadcastHashJoin` 与 `BroadcastExchange` 成对出现，表示广播连接——数据被收集到 driver 然后发送到每个 executor，之后再进行 join。

此外，Spark 还有 `SortMergeJoin` 和 `ShuffleHashJoin` 算子。



Subtree 的数量会小于 codegen 的数量

Spark WebUI 的 DAG 图中“绿色”背景色表示是缓存在内存的数据。

`spark.sql.codegen.maxFields` 默认值 100，激活 whole stage codegen 之前支持的最大的字段数量。如果字段数量超过哦这个值，将不能产生 whole stage codegen——物理计划中对应的计划的算子前面就没有 codegen id。

wholestage codegen 是 spark tungsten 优化的一部分，与之相对应的是旧版本 Spark 采用的火山模型。

`spark.sql.codegen.wholeStage` 开启 wholestagecodegen，默认为 true。

https://www.jianshu.com/p/9232b632090b







### Spark 源码

SparkSubmit

###### Spark 应用入口
SparkApplication

Dependency
	NarrowDependency
	ShuffleDependency
	OneToOneDependency
	RangeDependency

Task
	ShuffleMapTask
	ResultTask

Stage
	ShuffleMapStage
	ResultStage

###### 高级调度层（面向 stage 调度的实现），为每个 job 计算出 stage 组成的 DAG
DAGScheduler
	######A running job in the DAGScheduler
	ActiveJob	// A running job in the DAGScheduler

TaskScheduler

###### TaskScheduler 的唯一实现
TaskSchedulerImpl

###### 位于 TaskSchedulerImpl 中，TaskScheduler 的通信终端
SchedulerBackend



SparkContext
	###### 获取集群管理器
	###### 创建一个task scheduler 和 SchedulerBackend
createTaskScheduler()





### Hive UDTF

```
# 上传 jar、依赖的 jar
add jar /root/fastjson-1.2.5.jar;

add jar /root/moto_report_etl.jar;

# 查看 Hive 会话中添加的 jar 列表。
list jars;

# 创建临时函数（临时函数只存在于当前的 Hive 会话）
create temporary function ext_ctr as 'com.jdd.utils.CtrUDTF';

# 创建永久函数（会话无关的函数）
create function ext_ctr as 'com.jdd.utils.CtrUDTF' USING JAR  'hdfs://<jar_file_path>';

#函数是与库相关的，创建函数时如果不指定库名，则默认使用当前的库。使用函数时也需要库名，默认当前库。

# 删除函数
drop function ext_ctr;
```





![1681380474980](/assets/bad_broadcast.png)



broadcastJOin 不适用于所有的场合，如果多表 union（union 不是混洗操作） 和 小表（被广播） join，然后落表，此时可能会产生很多的小文件；反而导致 job 会比 sortmergejoin 更慢。

每个 Task 输入数据的大小\数量

![1681434602410](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\1681434602410.png)

每个 HadoopRDD 对应一个文件目录

![1681434517450](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\1681434517450.png)



### join 实现方式的选择

### Spark 优化

CBO `spark.sql.cbo.enabled`

https://blog.csdn.net/longlovefilm/article/details/121418148

小文件：https://blog.csdn.net/longlovefilm/article/details/120372001

数据倾斜：https://cloud.tencent.com/developer/article/2086649

