#### broadcast join

将`spark.sql.autoBroadcastJoinThreshold`设置为正值（-1表示禁用broadcast join模式，默认10MB，单位为字节）以开启。broadcast join模式下，会将小于`spark.sql.autoBroadcastJoinThreshold`的表广播到其他node进行计算，从而避免了shuffle，以提高效率。

Spark源码`SparkStrategies.scala`的`JoinSelection`是用来选择Join的实现方式的

```scala
  /**
   * Select the proper physical plan for join based on joining keys and size of logical plan.
   *
   * At first, uses the [[ExtractEquiJoinKeys]] pattern to find joins where at least some of the
   * predicates can be evaluated by matching join keys. If found,  Join implementations are chosen
   * with the following precedence:
   *
   * - Broadcast: if one side of the join has an estimated physical size that is smaller than the
   *     user-configurable [[SQLConf.AUTO_BROADCASTJOIN_THRESHOLD]] threshold
   *     or if that side has an explicit broadcast hint (e.g. the user applied the
   *     [[org.apache.spark.sql.functions.broadcast()]] function to a DataFrame), then that side
   *     of the join will be broadcasted and the other side will be streamed, with no shuffling
   *     performed. If both sides of the join are eligible to be broadcasted then the
   * - Shuffle hash join: if the average size of a single partition is small enough to build a hash
   *     table.
   * - Sort merge: if the matching join keys are sortable.
   *
   * If there is no joining keys, Join implementations are chosen with the following precedence:
   * - BroadcastNestedLoopJoin: if one side of the join could be broadcasted
   * - CartesianProduct: for Inner join
   * - BroadcastNestedLoopJoin
   */
  object JoinSelection extends Strategy with PredicateHelper {
  ......
  }
```

通过注释可见，如果join的一侧评估物理大小比用户定义的`SQLConf.AUTO_BROADCASTJOIN_THRESHOLD`小，或者用户明确地表示要广播表（比如对一个DataFrame使用了`org.apache.spark.sql.functions.broadcast()`函数），那么这一侧的表会被广播从而执行broadcast hash join。如果两侧的表都符合广播的条件，则进行Shuffle hash join。如果某个分区的平均大小小到能够构建hash表，进行Shuffle hash join。如果匹配join keys是可分类的，则进行sort merge join。

如果没有join keys，则按照BroadcastNestedLoopJoin（join的一侧可以广播）、CartesianProduct（对于inner join）、BroadcastNestedLoopJoin的优先级选择join实现。

广播是需要耗用带宽的，`spark.sql.broadcastTimeout`用于控制广播的超时时间，默认300s。



扩展阅读：[SparkSQL的三种Join实现](https://cloud.tencent.com/developer/article/1460186)

