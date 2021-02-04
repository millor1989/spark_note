[内置函数](https://spark.apache.org/docs/latest/api/sql/index.html)可以用在Spark SQL的sql语句中。

* **!**

  !expr

  逻辑非

* **%**

  expr1 % expr2

  返回`expr1 / expr2`之后的余数（remainder）

  ```sql
  > SELECT 2 % 1.8;
   0.2
  > SELECT MOD(2, 1.8);
   0.2
  ```

* **&**

  expr1 & expr2

  返回`expr1`和`expr2`按位与（bitwise AND）的结果

  ```sql
  > SELECT 3 & 5;
   1
  ```

* **\***

  expr1 \* expr2

  乘法

* **+**

  expr1 + expr2

  加法

* ---

  expr1 - expr2

  减法

* **/**

  expr1 / expr2

  浮点除法（floating point division），返回结果总是浮点类型

  ```sql
  > SELECT 3 / 2;
   1.5
  > SELECT 2L / 2L;
   1.0
  ```

* **&lt;**

  expr1 &lt; expr2

  小于

  `expr1`、`expr2`必须是相同类型或者可以被转换为一个共同的类型，并且必须是一个可以排序的类型。`map`类型是不能排序的（not orderable），所以不支持使用该函数。对于复杂类型（比如array/struct），它们的属性（fields）必须是可排序的。

  ```sql
  > SELECT 1 < 2;
   true
  > SELECT 1.1 < '1';
   false
  > SELECT to_date('2009-07-30 04:17:52') < to_date('2009-07-30 04:17:52');
   false
  > SELECT to_date('2009-07-30 04:17:52') < to_date('2009-08-01 04:17:52');
   true
  > SELECT 1 < NULL;
   NULL
  ```

* **&lt;=**

  expr1 &lt;= expr2

  小于等于

  参数要求同`<`

* **&lt;=&gt;**

  expr1 &lt;=&gt; expr2

  对于非null的操作数，与`=`操作符相同；如果两侧操作符都为null，返回true，而只有一个操作符为null时返回false。

  参数要求同`<`

  ```sql
  > SELECT 2 <=> 2;
   true
  > SELECT 1 <=> '1';
   true
  > SELECT true <=> NULL;
   false
  > SELECT NULL <=> NULL;
   true
  ```

* **=**

  expr1 = expr2

  等于

  ```sql
  > SELECT 2 = 2;
   true
  > SELECT 1 = '1';
   true
  > SELECT true = NULL;
   NULL
  > SELECT NULL = NULL;
   NULL
  ```

  参数要求同`<`

* **==**

  expr1 == expr2

  同`=`

* **&gt;**

  expr1 &gt; expr2

* 大于

  参数要求同`<`

* **&gt;=**

  expr1 &gt;= expr2

  大于等于

  参数要求同`<`

* **^**

  expr1 ^ expr2

  返回`expr1`和`expr2`按位异或（bitwise exclusive OR）的结果。

* **abs**

  abs\(expr\)

  返回数值的绝对值（absolute value）。

* **acos**

  acos\(expr\)

  返回`expr`的反余弦（inverse cosine，即arc cosine），与`java.lang.Math.acos`计算结果相同。

  ```sql
  > SELECT acos(1);
   0.0
  > SELECT acos(2);
   NaN
  ```

* **add\_months**

  add\_months\(start\_date,num\_months\)

  返回`start_date`加上`num_months`之后的date。

  ```sql
  > SELECT add_months('2016-08-31', 1);
   2016-09-30
  ```

  **Since**:1.5.0

* **aggregate**

  aggregate\(expr,start,merge,finish\)

  bula……不懂

  **Since**:2.4.0

* **and**

  expr1 and expr2

  逻辑与

* **approx\_count\_distinct**

  approx\_count\_distinct\(expr\[, relativeSD\]\)

  返回按照HyperLogLog++算法计算的近似基数（estimated cardinality，近似的去重后的元素数量）；可通过`relativeSD`定义允许的最大近似误差（maximum estimation error）。

* **approx\_percentile**

  approx\_percentile\(col, percentage\[,accuracy\]\)

  返回数值列`col`中，与指定`percentage`（百分比、百分数）对应的近似百分位的（approximate percentile）值。`percentage`的值必须介于0.0和1.0之间，`percentage`是数组时，它的每个元素都必须介于0.0和0.1之间，对应地，此时的返回结果也是对应百分位组成的数组。`accuracy`（默认值100）是一个正数字面量，以内存为代价来控制近似精度（approximation accuracy）。`accuracy`越大精度越高。

  ```sql
  /*ids_ttable的id为0~10000的整数*/
  > select approx_percentile(id,array(0.5,0.4,0.1),10000) from ids_ttable
   [4999.0, 3999.0, 999.0] 
  > select approx_percentile(id,0.198,1000) from ids_ttable
   1973.0
  ```

* **array**

  array\(expr, ...\)

  使用指定的元素返回一个数组

  ```sql
  > SELECT array(1, 2, 3);
   [1,2,3]
  ```

* **array\_contains**

  array\_contains\(array, value\)

  如果数组`array`包含元素`value`，返回`true`。如果数组`array`为`null`，则返回`null`。

  ```sql
  > SELECT array_contains(array(1, 2, 3), 2);
   true
  ```

* **array\_distinct**

  array\_distinct\(array\)

  移除数组`array`中的重复元素，返回数组

  ```sql
  > SELECT array_distinct(array(1, 2, 3, null, 3));
   [1,2,3,null]
  ```

  **Since**：2.4.0

* **array\_except**

  array\_except\(array1, array2\)

  返回数组`array1`剔除数组`array2`中相同元素后的结果，结果不包含重复元素。

  ```sql
  > SELECT array_except(array(1, 2, 3), array(1, 3, 5));
   [2]
  ```

  **Since**：2.4.0

* **array\_intersect**

  array\_intersect\(array1, array2\)

  返回数组`array1`和`array2`的交集，结果不包含重复元素

  ```sql
  > SELECT array_intersect(array(1, 2, 3), array(1, 3, 5));
   [1,3]
  ```

  **Since**：2.4.0

* **array\_join**

  array\_join\(array, delimiter\[, nullReplacement\]\)

  使用分隔符`delimiter`、可选的替换`null`值得字符串`nullReplacement`来连结数组`array`中的元素；如果不使用`nullReplacement`，数组中的`null`值将被忽略。

  ```
  > SELECT array_join(array('hello', 'world'), ' ');
   hello world
  > SELECT array_join(array('hello', null ,'world'), ' ');
   hello world
  > SELECT array_join(array('hello', null ,'world'), ' ', ',');
   hello , world
  ```

  **Since**：2.4.0

* **array\_max**

  array\_max\(array\)

  返回数组中最大值，忽略`null`值。

  ```
  > SELECT array_max(array(1, 20, null, 3));
   20
  ```

  **Since**：2.4.0

* **array\_min**

  array\_min\(array\)

  返回数组中最小值，忽略`null`值。

  ```
  > SELECT array_min(array(1, 20, null, 3));
   1
  ```

  **Since**：2.4.0

* **array\_position**

  array\_position\(array, element\)

  返回指定元素在数组中的（从1开始的）索引，结果类型为long。

  ```
  > SELECT array_position(array(3, 2, 1), 1);
   3
  ```

  **Since**：2.4.0



**to be continued.....**

