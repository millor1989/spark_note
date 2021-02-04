### 数据倾斜调优

##### 数据倾斜时的现象：

- 绝大多数task执行得都非常快，个别task执行极慢。比较常见。
- 本来可以正常执行的Spark作业，某天忽然OOM异常，观察异常栈，是业务代码造成的。这种情况比较少见。

##### 数据倾斜发生的原理：

shuffle时必须将各个节点上相同的key拉取到某个节点的一个task进行处理，比如按照key进行聚合或者join等操作；此时，如果某个key对应的数据量特别大，就会发生数据倾斜；处理数据量大的那个key的task运行很慢，从而导致整个Spark作业运行时间很长。

因此，数据倾斜时，Spark作业看起来运行非常慢，甚至可能因为某个task处理的数据量过大而导致内存溢出。

##### 导致数据倾斜代码的定位：

数据倾斜只发生在shuffle过程中。常见的触发shuffle的算子有：distinct、groupByKey、reduceByKey、aggregateByKey、join、cogroup、repartition等。数据倾斜就可能是这些算子中的某个导致的。

##### 某个task执行慢的情况

首先要查看，数据倾斜发生在哪个stage中。

如果是yarn-client模式提交，本地可以看到log，可以在log中查看当前运行到哪个stage；如果是yarn-cluster模式提交，可通过Spark Web UI查看当前是哪个stage。此外，不论是yarn-client还是yarn-cluster模式，都可以通过Spark Web UI查看当前stage各个task分配的数据量，从而确定task的数据量分配是否导致了数据倾斜。

确定了数据倾斜的stage后，可以根据stage划分原理推断数据倾斜stage对应代码中的哪一段，并找到对应的shuffle算子。

##### 某个task莫名其妙内存溢出的情况

一般可以直接通过log中的异常栈定位到代码中引发内存溢出的那一行，进而可以找到引发异常的shuffle算子。

但是，不能根据偶然的内存溢出就判定是数据倾斜。代码bug或者数据偶然异常也可能导致内存溢出。还需要确认stage各个task的运行时间和分配的数据量才能确定是否是由于数据倾斜导致了内存溢出。

##### 查看导致数据倾斜的key的数据分布情况

发生数据倾斜后，通常要分析一下那个执行了shuffle操作并且导致了数据倾斜的RDD/Hive表，查看其中的key的分布情况，以为解决方案的选择提供依据。

查看key分布的方式：1、如果是Spark SQL中的group by、join导致的数据倾斜，可以通过SQL查询表的key分布情况。2、如果是Spark RDD执行shuffle算子导致数据倾斜，可以在Spark作业中加入查看key分布的代码，比如`RDD.countByKey()`，将统计结果collect/take到客户端输出一下，就可查看到key的分布状况。

#### 数据倾斜的解决方案：

##### 1、使用Hive ETL预处理数据

**适用场景**：Hive表key数据不均匀导致数据倾斜，而业务场景需要频繁用Spark对Hive表执行分析

**实现思路**：使用Hive对数据进行预处理（即通过Hive ETL预先对数据倾斜的Hive表按照key进行聚合，或者预先和其它表进行join），将结果落到新的Hive表；那么，Spark基于新Hive表进行操作，就避免了引起数据倾斜的步骤。

这种方案简单，效果也好，Spark作业避免了数据倾斜，但是Hive ETL时还是会遇到数据倾斜的问题。

如果Spark作业需要频繁多次执行，那么在此之前通过Hive ETL对引发数据倾斜的Hive表进行预处理是值得的。

##### 2、过滤少数导致倾斜的key

**使用场景**：如果导致数据倾斜的是某几个key，并且这几个key不影响最终的结果，或者不关心这几个key的数据。那么可以考虑过滤掉这些引发数据倾斜的key对应的数据。

**实现思路**：在Spark SQL中可以用where/filter，在Spark Core中可以对RDD执行filter，过滤引发倾斜的key。如果每次Spark作业执行时，需要动态判断过滤数据量大的key，那么可以使用sample算子对RDD进行采样，然后计算出每个key的数量，过滤那些数据量大的key即可。

这种方案也很简单，可以避免数据倾斜，但是适用场景可能较窄。

对于异常key引发的数据倾斜等少数场景适用。

##### 3、提高shuffle的并行度

**适用场景**：必须要在Spark作业中处理掉倾斜的数据

**实现思路**：对于RDD，在执行shuffle算子时指定并行度，比如reduceByKey可以通过reduceByKey(1000)设置这个shuffle算子执行时shuffle read task的数量。对于Spark SQL中的shuffle算子，比如group by、join，可以通过参数`spark.sql.shuffle.partitions`设置shuffle read task的并行度，该参数默认值200，可能对于某些场景偏小。

增加了shuffle read task的数量，本来有一个task处理的数据量可以分给多个task，每个task处理的数据量就更少，那么就可以环境倾斜数据对task造成的压力。

这种方法也比较简单，可以缓解数据倾斜的影响，但是没有根本解决问题，效果可能有限。有时需要结合其它方案使用。

##### 4、两阶段聚合（局部聚合 + 全局聚合）

**使用场景**：对RDD执行reduceByKey等聚合类shuffle算子，或者在Spark SQL中使用gorup by语句进行分组聚合

**实现思路**：首先局部聚合，给每个key赋予均匀分布的随机数（比如10以内的随机数）作为局部聚合key，先对局部聚合key进行聚合，再在局部聚合结果上进行原始key的聚合。这样把一个stage的处理分散到两个stage，从而减少task需要处理的数据量。

这种方案对于聚合类shuffle算子引发的数据倾斜效果较好，至少可以大幅缓解数据倾斜。

但是只适用于聚合类的操作，使用范围较窄。对于join类shuffle不适用。

##### 5、将reduce join转为map join

**适用场景**：在对RDD使用join类操作，或者是在Spark SQL中使用join语句时，join操作的一个RDD或表的数据量比较小（比如几百M或者一两个G）

**实现思路**：不使用join算子进行连接操作，而使用广播变量与map类算子实现join操作，从而避免shuffle类操作进而避免数据倾斜。将较小的RDD通过collect拉取到Driver，并将其创建为广播变量；对另一个RDD执行map类算子，在算子函数内，从广播变量获取较小的RDD，并与当前RDD的每一条数据按照连接key进行对比，如果连接key相同则将两个RDD的数据用需要的方式进行处理。

普通的join操作会进行shuffle，将相同key的数据拉取到一个shuffle read task中进行join，即reduce join。如果一个RDD较小，将较小RDD广播并结合map算子实现join效果，也就是map join，此时就不会发生shuffle。

对于join操作导致的数据倾斜效果很好，避免了数据倾斜，但是仅适用于一个RDD比较小的场合。

##### 6、采样倾斜key并分拆join操作

**适用场景**：对于join时两个RDD/Hive表都比较大的场合，如果其中一个RDD/Hive表的少数几个key的数据量较大，而另一个RDD/Hive表的所有的key数量分布比较均匀，可以采用这种方案

**实现思路**：1、对包含少数几个数据量过大的key的RDD，通过sample算子采样出一份样本，统计一下每个key的数量，计算出数据量较大的几个key；2、将这几个key对应的数据从原理的RDD中拆分出来，形成一个单独的RDD，并为每个key的数据赋予n以内均匀分布的随机数作为局部key；3、将需要join的另一个RDD也过滤出对应的key的数据，形成另一个RDD，将每条数据膨胀为n条数据，这n条数据都有一个0~n的局部key；同时将不引起数据倾斜的key对应的数据形成另一个RDD；4、将附加了局部key的两个RDD进行join，那么原先相同的key被分为了n份，分散到多个task中进行join了；其余部分两个RDD照常进行join；5、将两部分join结果使用`UNION`合并便是最终的join结果。

对于join导致的数据倾斜，如果只是某几个key导致了倾斜，这种方式比较有效；将少数数据量大的key拆分的同时，将另一个数据均匀分布的对应少数key的数据量进行了膨胀，而不是对全量数据膨胀。避免了占用过多内存。

如果导致倾斜的key比较多，则不适用此方案。

##### 7、使用随机前缀和扩容RDD进行join

**适用场景**：如果进行join操作时，一个RDD有大量key导致数据倾斜

**方案思路**：与**方案6**类似，1、查看RDD/Hive表的数据分布情况，确认造成数据倾斜的key的数量；2、将该RDD的每条数据通过赋予n以内均匀分布的随机数作为局部key；3、对另外一个RDD，每条数据膨胀n倍，并赋予0~n的局部key；4、将两个处理后的RDD进行join

这种方案对于join类的数据倾斜基本都可以处理，效果也较好，但是对内存资源要求高。

该方案更多的是缓解数据倾斜，而不是彻底避免数据倾斜。

##### 8、上述多种方案组合使用

对于比较复杂的数据倾斜场景，可能需要多种方案组合使用。

### Shuffle调优

shuffle环节包含大量的磁盘IO、序列化、网络传输等操作，对Spark作业性能影响很大。但是，影响Spark作业性能的，更主要的还是代码开发、资源参数、数据倾斜，shuffle调优对Spark作业性能影响相对较小。

#### shuffleManager

负责shuffle过程的执行、计算和处理的组件主要是ShuffleManager，即shuffle管理器。

Spark 1.2之前，默认的shuffle管理器是HashShuffleManager，它有一个严重的弊端：会产生大量的中间磁盘文件，有大量磁盘IO从而影响了性能。

Spark 1.2之后，默认的shuffle管理器是SortShuffleManager，相较于HashShuffleManager，有了一定的改进。主要在于，每个task在shuffle时，虽然也有较多的临时磁盘文件，但是最后会将所有临时文件合并（merge）为一个磁盘文件，因此每个task只有一个磁盘文件。在下一个stage的shuffle read task拉取自己的数据时，只要根据索引读取每个磁盘文件中的部分数据即可。

###### 未经优化的HashShuffleManager

![](/assets/08f3b596.png)

这里明确一个假设前提：每个executor只有1个CPU核心，即无论这个executor上分配多少task，同一时间只能执行一个task线程。

在shuffle write阶段，主要是在一个stage结束后，为了下一个stage可以执行shuffle类算子（比如reduceByKey），而将每个task处理的数据按key进行“分类”。所谓“分类”，是对相同的key执行hash算法，从而将相同key都写入同一个磁盘文件中，每个磁盘文件都只属于下游stage的一个task。在将数据写入磁盘之前，先将数据写入内存缓冲区，内存缓冲区填满后，才溢出到磁盘文件中。下一个stage有多少个task，当前stage的每个task就要创建多少个磁盘文件，可见，未经优化的shuffle write操作产生的磁盘文件数量是很大的。

shuffle read通常是一个stage刚开始时要做的事情，该stage的每个task要从各个节点，将上一个stage的计算结果中的所有相同key，通过网络拉取到自己所在的节点；然后，进行key的聚合或者连接操作。shuffle write过程中，给下游stage的task都创建了一个磁盘文件，因此shuffle read过程中，每个task只用从上游stage的所有task所在节点拉取属于自己task的磁盘文件即可。

shuffle read过程是一边拉取文件一边聚合的。每个shuffle read task都有一个自己的buffer缓存，每次只能拉取与buffer缓存相同大小的数据，然后通过内存中的一个map进行聚合等操作。聚合完一批数据，再拉取下一批数据并放在缓存区域进行聚合操作。以此类推，直到将所有数据拉取完，得到最终结果。

###### 优化后的HashShuffleManager

![](/assets/3f3c6a82.png)

此处的优化是指设置参数`spark.shuffle.consolidateFiles`，该参数默认为false，设置为true即开启优化机制。如果使用HashShuffleManager，建议开启这个优化。

开启consolidate机制后，shuffle write过程中，task就不是为下游stage的每个taks创建一个磁盘文件了。此时会出现一个shuffleFileGroup的概念，每个shuffleFileGroup对应一批磁盘文件，磁盘文件的数量与下游stage的task数量是相同的。

一个executor有多少CPU核心，就可以并行执行多少task，第一批并行执行的每个task都会创建一个shuffleFileGroup，并将数据写入对应磁盘文件内。当executor的CPU核心执行完一批task，接着执行下一批task时，下一批task会复用之前已有的shuffleFileGroup和其中的磁盘文件；此时task会将数据写入已有的磁盘文件中，而不是创建新的磁盘文件。因此consolidate机制允许不同task复用同一批磁盘文件，这样可以将多个task磁盘文件进行一定程度的合并，从而大幅减少磁盘文件的数量，提升shuffle write性能。

###### SortShuffleManager的普通运行机制

![](/assets/b6d1140d.png)

该模式下，数据首先写入内存数据结构中，根据不同的shuffle算子，可以选择不同的数据结构。如果是reduceByKey这种聚合类的shuffle算子，会使用map数据结构，一边通过map进行聚合一边写入内存；如果是join这种shuffle算子，会使用Array数据结构，直接写入内存。

如果写入到内存的数据达到临界值，会尝试将内存数据结构中的数据溢出写到磁盘，然后清空内存数据结构。在溢出到磁盘文件之前，会先根据key对内存数据结构中已有的数据进行排序。排序后分批写入磁盘文件，默认每批数量10000条。写入磁盘是通过java的`BufferedOutputStream`实现的，`BufferedOutputStream`首先会将数据缓存在缓冲区，缓冲区满后再一次写入磁盘文件，可以减少磁盘IO提升性能。

一个task将所有数据写入内存数据结构的过程中，可能会有多次溢出到磁盘的过程，那么就有多个临时文件，最后会将所有临时文件都合并，即将所有临时文件的数据读取并依次写入最终磁盘文件，这一过程为merge。一个task只对应一个磁盘文件，即该task为下游stage的task准备的数据都在这一个文件中，因此还会单独写一个索引文件用来标识下游各task的数据在数据文件中的start offset和end offset。

###### SortShuffleManager的bypass运行机制

![](/assets/35ec4dc8.png)

bypass机制的**触发条件**：1、shuffle read task的数量小于或者等于`spark.shuffle.sort.bypassMergeThreshold`参数的值（默认200）时；2、不是聚合类型的shuffle算子（比如reduceByKey）。

此时task会为下游stage每个task创建一个临时磁盘文件，将数据key进行hash然后根据key的hash值，将key写入对应的磁盘文件中。写磁盘文件也是先写入内存缓冲区，然后溢出到磁盘；同样，最后会将临时磁盘文件合并为一个磁盘文件并创建一个单独的索引文件。该过程前期与未经优化的HashShuffleManager相同，只是最后会合并文件。相比未经优化的HashShuffleManager，少量的最终磁盘文件使得shuffle read性能更好。

该机制相对于SortShuffleManager普通运行机制：磁盘写机制不同，不会进行排序。所有启用该机制，shuffle write过程中不需要进行数据的排序操作，从而节省了这部分性能开销。

#### Shuffle参数调优：

##### 1、spark.shuffle.file.buffer

默认值32k

用于设置shuffle write task的BufferedOutputStream的缓存区大小

如果内存资源充足，可以适当增加这个参数（比如64k），从而减少shuffle write过程的磁盘IO。

##### 2、spark.reducer.maxSizeInFlight

默认值48m

用于设置shuffle read task的缓存大小，决定了每次读取的数据大小

如果内存充足，可以适当增加（比如96m），以减少读取数据的次数，同时减少网络传输次数。

##### 3、spark.shuffle.io.maxRetries

默认值3

shuffle read task从shuffle write task所在节点拉取数据失败时自动重试的最大次数。如果重试次数达到这个值还没有成功，那么task失败。

对于包含特别耗时的shuffle操作的作业，可将重试次数设置为最大值（比如60），以避免由于JVM full GC或者网络不稳定等因素导致失败。对于超大数据量（几十亿~上百亿）的shuffle过程，增大该参数可增加作业的稳定性。

##### 4、spark.shuffle.io.retryWait

默认值5s

shuffle read task从shuffle write task所在节点拉取数据失败后，自动重试等待的时间间隔

建议加大（比如60s）以增加作业稳定性

##### 5、spark.shuffle.memoryFraction

默认值0.2

executor内存中用于shuffle read task进行聚合操作的内存比例

如果内存充足，并且很少进行持久化操作，可适当增加。

##### 6、spark.shuffle.manager

默认值sort

设置shuffleManager类型，可选项为：hash、sort和tungsten-sort。hash代表HashShuffleManager；sort代表SortShuffleManager；tungsten-sort与sort类似，但是前者使用了tungsten计划中的堆外内存管理机制，内存使用效率更高。

SortShuffleManager默认要排序，如果要排序可以使用SortShuffleManager；如果不需要排序，建议参考后面的几个参数调优，通过bypass机制或者优化的HashShuffleManager来避免不必要的排序，同时提供较好的磁盘读写性能。tungsten-sort慎用，可能有一些bug。

##### 7、spark.shuffle.sort.bypassMergeThreshold

默认值200

ShuffleManager为SortShuffleManager时，如果shuffle read task数量少于这个阈值，则启用bypass运行机制，否则采用普通运行机制。

如果使用SortShuffleManager，而不需要排序，可以设置得大些。

##### 8、spark.shuffle.consolidateFiles

默认值false

ShuffleManager为HashShuffleManager时，如果设置为true，则开启consolidate机制。

如果不需要排序，可以将`spark.shuffle.manager`指定为hash，同时将该参数置为true开启consolidate机制。其性能比开启了bypass机制的SortShuffleManager要高。