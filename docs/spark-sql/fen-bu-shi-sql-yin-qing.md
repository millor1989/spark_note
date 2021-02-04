### 分布式的SQL引擎

通过使用Spark SQL的JDBC/ODBC或者命令行接口，可以把Spark SQL作为分布式查询引擎。在这种模式下，用户和应用可以直接通过SQL查询与Spark SQL交互，而不用写任何代码。

#### 1、运行Thrift JDBC/ODBC服务器

此处的Thrift JDBC/ODBC服务器实现对应于Hive 1.2.1的HiveServer2，可以使用Spark或者Hive 1.2.1的beeline脚本来测试JDBC服务器。

在Spark目录下运行JDBC/ODBC服务器：

```shell
./sbin/start-thriftserver.sh
```

这个脚本可以接收`bin/spark-submit` 的所有命令行选项，另外还支持通过`--hiveconf` 来指定Hive属性。可以运行 `./sbin/start-thriftserver.sh --help`来查看完整的可用选项列表。默认情况下，JDBC/ODBC服务监听的是`localhost:10000`，可以使用环境变量或者系统属性来进行配置：

```bash
export HIVE_SERVER2_THRIFT_PORT=<listening-port>
export HIVE_SERVER2_THRIFT_BIND_HOST=<listening-host>
./sbin/start-thriftserver.sh \
  --master <master-uri> \
  ...
```

```bash
./sbin/start-thriftserver.sh \
  --hiveconf hive.server2.thrift.port=<listening-port> \
  --hiveconf hive.server2.thrift.bind.host=<listening-host> \
  --master <master-uri>
  ...
```

然后，可以使用beeline来测试Thrift JDBC/ODBC服务器。

```bash
./bin/beeline
```

在beeline中连接JDBC/ODBC服务器：

```bash
beeline> !connect jdbc:hive2://localhost:10000
```

beeline会需要用户名和密码，在非安全模式下，简单的输入机器的用户名和空的密码就可以，对于安全模式，根据[beeline文档](https://cwiki.apache.org/confluence/display/Hive/HiveServer2+Clients)指示进行操作。

把`hive-site.xml`, `core-site.xml` 和 `hdfs-site.xml` 文件放进`conf/`即可完成Hive配置。

也可以使用Hive自带的beeline脚本。

Thrift JDBC服务也支持通过HTTP传输来发送thrift RPC消息。可以通过系统属性或者修改`conf/`目录中的`hive-site.xml`文件的如下属性，来开启HTTP模式：

```
hive.server2.transport.mode - Set this to value: http
hive.server2.thrift.http.port - HTTP port number to listen on; default is 10001
hive.server2.http.endpoint - HTTP endpoint; default is cliservice
```

如果要测试，使用beeline用http模式连接JDBC/ODBC服务器：

```
beeline> !connect jdbc:hive2://<host>:<port>/<database>?hive.server2.transport.mode=http;hive.server2.thrift.http.path=<http_endpoint>
```

#### 2、Spark SQL CLI

Spark SQL CLI是以本地模式运行Hive metastore服务并执行命令行输入查询的方便工具。Spark SQL CLI不能和Thrift JDBC服务器沟通。

在Spark目录下运行如下命令启动Spark SQL CLI：

```
./bin/spark-sql
```

把`hive-site.xml`, `core-site.xml` 和 `hdfs-site.xml` 文件放进`conf/`即可完成Hive配置。可以运行`./bin/spark-sql --help`命令来查看所有的可用选项。