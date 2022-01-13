### Structured Streaming

#### ForeachWriter

`ForeachWriter` 中，在 `open` 方法中创建了 redis 连接池，但是 redis 连接池在 `close` 方法中总是（不能调用连接池的 close 方法，也不能做其他使用）报错： `redis.clients.jedis.exceptions.JedisConnectionException: no reachable node in cluster`， 难道是 `close` 之前关闭了连接池吗？

#### checkpoint

一个 executor 时的 `checkpoint` 在改了 executor 数量之后再次执行，则会报错——checkpoint 中文件找不到之类的。难道 checkpoint 也记录了 executor 信息？

#### *org.apache.hadoop.yarn.exceptions.ApplicationAttemptNotFoundException*: Application attempt appattempt_xxxxxxxxxxxx_xxxxx_000001 *doesn't exist in ApplicationMasterService cache*

看其它的应用执行日志，`DiskBlockManager` 会创建本地目录：

```
21/08/27 11:21:05 INFO storage.DiskBlockManager: Created local directory at /data02/yarn/nm/usercache/hdfs/appcache/application_xxxxx_xxxxx/blockmgr-c23ac3cd-f988-4ac7-a1a5-2bf65952fd49
21/08/27 11:21:05 INFO storage.DiskBlockManager: Created local directory at /data04/yarn/nm/usercache/hdfs/appcache/application_xxxxx_xxxxx/blockmgr-bae60339-e1c1-4086-8c0d-419c5b48a75c
21/08/27 11:21:05 INFO storage.DiskBlockManager: Created local directory at /data03/yarn/nm/usercache/hdfs/appcache/application_xxxxx_xxxxx/blockmgr-9575e157-d701-4571-96f6-8771df0d14f4
21/08/27 11:21:05 INFO storage.DiskBlockManager: Created local directory at /data01/yarn/nm/usercache/hdfs/appcache/application_xxxxx_xxxxx/blockmgr-eb07d275-39e5-4262-9322-e561d1429fcb
21/08/27 11:21:05 INFO storage.DiskBlockManager: Created local directory at /data06/yarn/nm/usercache/hdfs/appcache/application_xxxxx_xxxxx/blockmgr-e3948e8a-305f-4940-9ea9-34107cdb80eb
21/08/27 11:21:05 INFO storage.DiskBlockManager: Created local directory at /data05/yarn/nm/usercache/hdfs/appcache/application_xxxxx_xxxxx/blockmgr-49986069-6cb1-47b1-8875-1157e14d38e6
```

会往其中写入一些状态：

```
21/08/27 11:21:32 INFO session.SessionState: Created local directory: /data06/yarn/nm/usercache/hdfs/appcache/application_xxxxx_xxxxx/container_xxxxx_xxxxx_01_000001/tmp/nobody
21/08/27 11:21:34 INFO session.SessionState: Created local directory: /data06/yarn/nm/usercache/hdfs/appcache/application_xxxxx_xxxxx/container_xxxxx_xxxxx_01_000001/tmp/c2c490df-2bc3-4cbd-97b7-06d537fea88e_resources
21/08/27 11:21:34 INFO session.SessionState: Created HDFS directory: /tmp/hive/hdfs/c2c490df-2bc3-4cbd-97b7-06d537fea88e
21/08/27 11:21:34 INFO session.SessionState: Created local directory: /data06/yarn/nm/usercache/hdfs/appcache/application_xxxxx_xxxxx/container_xxxxx_xxxxx_01_000001/tmp/nobody/c2c490df-2bc3-4cbd-97b7-06d537fea88e
21/08/27 11:21:34 INFO session.SessionState: Created HDFS directory: /tmp/hive/hdfs/c2c490df-2bc3-4cbd-97b7-06d537fea88e/_tmp_space.db
```

最后会删除这些目录：

```
21/08/27 11:22:58 INFO util.ShutdownHookManager: Deleting directory /data06/yarn/nm/usercache/hdfs/appcache/application_xxxxx_xxxxx/spark-778a395d-5e7c-4860-97f5-a0fc4ebf0b3a
21/08/27 11:22:58 INFO util.ShutdownHookManager: Deleting directory /data03/yarn/nm/usercache/hdfs/appcache/application_xxxxx_xxxxx/spark-af6e2617-b734-4e85-8e22-362d25a97a7d
21/08/27 11:22:59 INFO util.ShutdownHookManager: Deleting directory /data02/yarn/nm/usercache/hdfs/appcache/application_xxxxx_xxxxx/spark-c50bfbde-5da1-409f-9e66-ac38307813da
21/08/27 11:22:59 INFO util.ShutdownHookManager: Deleting directory /data05/yarn/nm/usercache/hdfs/appcache/application_xxxxx_xxxxx/spark-296c7bc0-84be-4e84-968b-9bd10a315dbf
21/08/27 11:22:59 INFO util.ShutdownHookManager: Deleting directory /data04/yarn/nm/usercache/hdfs/appcache/application_xxxxx_xxxxx/spark-5f482a9b-b3d5-4bba-9f6a-a9c30b777e1a
21/08/27 11:22:59 INFO util.ShutdownHookManager: Deleting directory /data01/yarn/nm/usercache/hdfs/appcache/application_xxxxx_xxxxx/spark-e47f24f1-3595-46e5-80f7-de6b5fc3c7ce
```

难道是，中途这些目录被删除了……导致应用读不到一些缓存的状态……